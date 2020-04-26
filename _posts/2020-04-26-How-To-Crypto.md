---
layout: post
section-type: post
title: How to Crypto
category: Investment
tags: [ 'Investment' ]
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

The process for bitcoin may well be similar to ethereum, but my initial thoughts are to
use [bitaddress](http://bitaddress.org) which I have downloaded from github (noting the
repo hadn't changed in years and checksums were correct).  I may come back to this.


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



