---
layout: post
section-type: post
title: Casting YouTube Videos from Tablet
category: Dev
tags: [ '' ]
frontPgDisp: Yes
---


I had been using Raspicast to cast YouTube videos from my Tablet to the TV (Raspberry Pi).
Unfortunately, it was a bit hit and miss, sometimes failing for no reason. Given this, I
decided to write a bash script to do this, which I called `ytplay.sh` :


```bash
#!/bin/bash

############################################################################################## 
#	
#	     Script:  ytplay.sh
#	     Author:  Iain Benson
#	     Desc:    Stream Youtube & other sites using omxplayer (HW acceleration)
#                 Caters for Playlists as well as single URLs.
#
#        Some Notes:
#                 *Sound*
#                 omxplayer doesn't use ALSA or Raspi config setting, but detects automagically
#                 Can force with -o flag that has options: hdmi, local, both
#
#                 *Launch from Phone (via Share manu)*
#                 Use Termux.  Create script in bin called 'termux-url-opener'
#                 Leave it to that script to kill existing ytplay instances
#
#        Dependencies:
#                 omxplayer
#                 youtube-dl
#                 jq
#                 A Raspberry Pi :)
#
############################################################################################## 

# Warning: going to use colour. Sunglasses on.
RED='\033[0;31m'
NO_COL='\033[0m'

PlayVid()
{
	# Fetch 'real' video link (-g) and title (-e) with youtube-dl
	# If specify diff audio/video streams they're not being multiplexed (ffmpeg issue?) => No sound
	# Use old youtube-dl behaviour specifying only quality results in single file - See github page.
	# Try for mp4 first (so wont get vp9).  If no mp4 then default to best that is avilable.
	clear; echo "Fetching video info"
	TITLEURL=$(youtube-dl -e  -g -f 'mp4[height <=? 720]/best[height <=? 720]' "$1")

	# Sometimes title is followed by newline which breaks sed regex
	FIXEDTITLEURL=$(echo $TITLEURL | tr '\n' ' ' | tr '\r' ' ')

	# Title followed by link.  Split on http
	LINK=$(echo $FIXEDTITLEURL | sed 's/.*http/http/')
	TITLE=$(echo $FIXEDTITLEURL | sed 's/http.*//')

	# Output Title to shell
	clear; echo -e "Playing: ${RED}$TITLE${NO_COL}"

	# omx (hidden) -b option clears sides of video for 3:4 vids as terminal was visible
	omxplayer -b "$LINK"
}


# Determine if playing single video or playlist
if [[ $1 = *"list="* ]]; then

	echo "Reading Youtube Playlist."

	# Issue with url containing watch not working.  fix it
	FIXED=$(echo "$1"|sed 's/watch.\+list=/playlist?list=/g')

	# If playlist download with youtube-dl it as json
	PLAYLIST_JSON=$(youtube-dl -j --flat-playlist "$FIXED" )
	
	# All on same line.  Split onto separate for easy processing. Note \n doesn't work.
	PLAYLIST_JSON=$(echo $PLAYLIST_JSON | sed 's/} {/}\r{/g')

	# For loop uses whitespace.  Use Internal File Separator to use \r instead
	IFS=$'\r'
	for line in $PLAYLIST_JSON
	do
		# Get the Title and Url.  Use jq to process json
		#TITLE=$(echo $line | jq -r '.title')
		URL=$(echo $line | jq -r '.url')

		# Create Url(web url not extracted omx compatable one) from id and play using our function above
		PlayVid "https://www.youtube.com/watch?v=$URL"

		# Once video ends, give user ability to quit out of playlist
		echo -e "${RED}"
		read -p "Press q to quit playlist " -t 2 -n 1 key <&1
		echo -e "${NO_COL}"
		if [[ $key = q ]] ; then
			break
		fi

	done

else

	# Easy, Playing Single Video
	PlayVid "$1"

fi
```

I had originally used [mpv](https://mpv.io/), which was nice as it handled YouTube playlists automatically.
But it isn't compiled to use the Raspberry Pi's hardware acceleration, so some videos were
jerky.  I tried (and failed) to recompile it with HW acceleration, so in the end decided to switch back to
[omxplayer](https://github.com/popcornmix/omxplayer/).

## PlayVid Function

This function uses the [youtube-dl](https://rg3.github.io/youtube-dl/) '-g' option to get the actual video link to be
passed to omxplayer. The -e option returns the video title, which I'm going to display on
my tablet.  The script has a bit of processing to extract the link/title from the returned
string.

The youtube-dl '-f' option is used to try and select the required video fidelity.  I'm
aiming to retrieve mp4 no greater than 720p that the Rapberry Pi can handle.  If that fails,
I'll just go for Best at 720p or less (The Pi has trouble with vp9, so this might not
work).  I'm not able to fetch separate audio/video streams as is commonly done, it seems
like there is an issue in them not getting multiplexed together and I end up with a video
with no sound.

Once I have the  omx compatible link, I call omxplayer with the -b option, there wasn't documentation for
that.  On old-style videos (3:4) the background terminal was visible at the sides while playing, this
blacks them out.

## Playlists

As omxplayer didn't handle playlists, I wrote code to do this for me.  It fetches the
playlist info using youtube-dl and then parses it in a for loop.  As json is returned I 
used jq to parse the output line at a time.  After each video, the user has the option to press q to exit
the playlist.

## Calling from Tablet

I use [Termux](https://termux.com/) on my tablet, which gives me a bash terminal.  When items are shared to
Termux, it looks for a handler script `termux-url-opener` :


```bash
#!/data/data/com.termux/files/usr/bin/bash

# Kill off existing ytplay instances on the Raspberry Pi
ssh pi@raspberrypi "pgrep ytplay && killall ytplay.sh; exit"
clear

# Play video using ytplay script on the pi via ssh
# -t option is essential to force pseudo terminal - as gives keyboard control
ssh pi@raspberrypi -t "/home/pi/bin/ytplay.sh \"$1\""

```

Termux is set up to use keyfiles, so no need to enter passwords when ssh-ing into the
pi. 

## Youtube-dl Slow

One minor annoyance is that youtube-dl takes a few seconds to extract the omx compatible link.  This
appears to be a known issue and is being caused by the python script importing extractors 
(it supports hundreds of sites, not just YouTube).  There is a github issue for this and
it looks like a flag will be added at some point to limit the extractors loaded (I did see
a patch that could be manually applied, I may do this at some point)


## Conclusion

With this set up, simply selecting share then Termux from the browser will play the media stream 
on the Pi.  It works well and other sites such as Vimeo, Soundcloud, etc are supported. 

The script is available from my [github repo](https://github.com/0x3F3F/bin/blob/master/ytplay.sh).




