---
layout: post
section-type: post
title: Buying Crypto Currencies
category: Investment
tags: [ 'Investment','Crypto' ]
frontPgDisp: Yes
---

<img style="border: 0;" src="/img/2020/20200426_header.jpg" />

I currently don't own any crypto currencies, though thought I'd investigate how I'd go
about buying some should I change my mind.

I wouldn't be keen on storing any currencies at an exchange, and so I'd opt for off-line
cold storage (in preference to a  hardware wallet). The basic idea being to generate and
addresses/keys on a computer that will never be connected to the internet. 

## Ethereum

My starting point was [this python script](https://github.com/vkobel/ethereum-generate-wallet) 
which I could follow and seemed reasonable.  I supplemented this as follows:

- Validate the generated address using python Web3 library
- Using the generated private key, ensured it generates the same public key once converted
  back using another method.
- Generate QR codes for the resulting ethereum address / private key.

[My script](https://github.com/0x3F3F/scripts/blob/master/ethGenWalletAndFiles.py) output to 
the terminal as follows:

<img style="border: 0;" src="/img/2020/20200426_ethereumScript.png" />

It generates a png containing the QR Codes:

<img style="border: 0;" src="/img/2020/20200426_ethereumQR.png" />

After I have the required files, I would use `gpg` to encrypt them before transferring them
off of the offline machine.  Any intermediate files would then be deleted using the linux
`shred` utility.


## Bitcoin

My starting point with bitcoin was the [bitaddress](http://bitaddress.org) site, which many 
use to generate their wallets. My plan was to review the code before using it - however, 
looking at the code there were over 10K lines of very low level code.

I then went about looking for python scripts, though couldn't find a working solution.
Bitcoin encoding is more difficult than that of ethereum.

I finally stumbled on a post on how to perform the encoding
[here](https://www.freecodecamp.org/news/how-to-generate-your-very-own-bitcoin-private-key-7ad0f4936e6c/) and 
[here](https://www.freecodecamp.org/news/how-to-create-a-bitcoin-wallet-address-from-a-private-key-eca3ddd9c05f/), 
along with a corresponding [github repo](https://github.com/Destiner/blocksmith) which I used to create a script. 

While I didn't trust the bitaddress code to generate the private key, the offline site
does have the option of generating addresses in various formats and QR codes.  My plan is
to use that to validate the script output and to get a png with QR code etc.


## Ripple

Offline options were limited for Ripple.  I initially found [xrppaperwallet](http://www.xrppaperwallet.com/#paper-wallet)
though couldn't find much further information on the site and so didn't pursue it further.

I then found a random [github repository](https://github.com/whotooktwarden/generateSecretOffline) which looked more 
promising.  Essentially it just executed a function in the ripple lib javascript library.  The only question mark was 
if that library was trustworthy.

I then decided to try to build the ripple javascript library myself, directly  from their [github
repository](https://github.com/ripple/xrpl-dev-portal/blob/master/content/tutorials/get-started/get-started-with-rippleapi-for-javascript.md#install-yarn)
, which proved to be a bit complicated as (i)Installing `yarn` on ubuntu installs cmdtest instead  (ii) ripple library 
now needs a jquery js file that the earlier one didn't.  Once I has the new trustworthy ripple-lib, I then just updated 
the generateSecretOffline page to use that instead.

In addition to the above, I also wrote a [python script](https://github.com/0x3F3F/scripts/blob/master/xrpGenQrCodes.py) to i
generate QR Codes that I could then encrypt before transferring off of the offline machine.



