---
title: "sudo auth using a yubikey with pin required"
date: 2025-12-10
---

# Requirements

This guide is for Linux Mint 22.2

First start by installing the reqs

`sudo apt install libpam-u2f`

## Setup

Plug in your yubikey and create the folder for saving your key
`mkdir ~/.config/Yubico`

Run pamu2fcfg to add yubikey to the authorised key list 

`pamu2fcfg > ~/.config/Yubico/u2f_keys`

I highly recommend that you have a pin set! You can do this in the YubiKey. If you do not have a pin set that means anyone who has physical access to your YubiKey can gain root access on your machine by simply plugging it in and pressing the button!

Change the permissions to `0644` (Meaning only the owner can read/write and anyone else can read)

`sudo chmod 0644 ~/.config/Yubico/u2f_keys`

Once thats done we can go ahead and edit our `pam.d`

`sudo nano /etc/pam.d/sudo` 

##### WARNING: While you are making changes to your pam.d do not close the terminal until you are 100% sure the changes are functioning as intended. You have the potential to screw up your auth so be warned!


```bash
#%PAM-1.0

# Set up user limits from /etc/security/limits.conf.
session    required   pam_limits.so

session    required   pam_env.so readenv=1 user_readenv=0
session    required   pam_env.so readenv=1 envfile=/etc/default/locale user_readenv=0

@include common-auth
@include common-account
@include common-session-noninteractive
```

You should see a file looking something like this if you are on base install of Linux Mint.
`common-auth` is your standard password login. The order of this file will determine which methods come first

In this example we will be placing our u2f above the `common-auth` and `sufficient` that if we have a u2f key present in the system we will be prompted to use it. Otherwise a regular root password is fine. 
I am using the Yubikey here not as a true 2 factor but rather as simpler solution of PIN + Touch.

### WARNING: If you want to make your yubikey `required` please ensure that your `u2f_keys` file is NOT INSIDE YOUR HOME DIRECTORY AND A VALID PATH IS CONFIGURED IN PAM-U2F If you use any form of home directory enycrption you will be UNABLE to authorise on first login.


With all the warnings out of the way. Insert your `auth` method above `common-auth` like so

```bash
...
auth sufficient pam_u2f.so pinverification=1 cue [cue_prompt=Tap the yubikey to sudo]
@include common-auth
...
```

Now if we save this file and open a new terminal

```
22:43:32 chad@peka
 -> ~ sudo su
Please enter the PIN:
Tap the yubikey to sudo
```
And your yubikey should start flashing for you to press it to verify physical presence. 

# Done!


## Thoughts and notes

This is a convenience and mild security thing for me. Outside of system startup I would like to avoid typing my root password as much as possible so using a yubikey means I am less vulnerable software based keylogging attacks. The best solution for security would be a combination of password & yubikey with a backup key. Since I dont currently own a spare yubikey this will do for now.

If i ever lose or forget my yubikey. Outside of some serious inconvenience it would not be a complete car-crash for me.

If you are interested in more information [this page](https://developers.yubico.com/pam-u2f/) from Yubico is a great resource and was used in writing this guide



