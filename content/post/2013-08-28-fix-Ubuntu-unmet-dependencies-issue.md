---
title: "Fix Ubuntu \"Unmet Dependencies\" Issue"
date: 2013-08-28
categories:
  - Ubuntu
tags:
  - Reverse Binary
draft: false
---

When I try to install s2e in my Ubuntu 12.04, there is a weired problem happens. It give the following error message when I try to install some of the libraries. Google directed me to the following solution, which worked perfectly for me.

__error__

```
[2][21:36:18][rui@~]$sudo apt-get install libsdl1.2-dev

Reading package lists... Done

Building dependency tree     

Reading state information... Done

Some packages could not be installed. This may mean that you have

requested an impossible situation or if you are using the unstable

distribution that some required packages have not yet been created

or been moved out of Incoming.

The following information may help to resolve the situation:



The following packages have unmet dependencies:

 libsdl1.2-dev : Depends: libpulse-dev but it is not going to be installed

E: Unable to correct problems, you have held broken packages.
```

__Solution__

First a downgrade is required and done with the following: create the 'preferences' file:

```bash
sudo vi /etc/apt/preferences
```

and insert the following lines:

```bash
Package: *       
Pin: release a=precise*
Pin-Priority: 2012
```

enter `:wq` to write the file. Pin-Priority must be greater than 1000.

Then you may downgrade the offending applications with:

```
sudo apt-get dist-upgrade
```

Then you may install packages that complained about dependencies, like `sudo apt-get install ia32-libs-multiarch`, or `sudo apt-get install ia32-libs`.

Finally, you should remove the file you just created:

```
sudo rm /etc/apt/preferences
```

because else no new updates would be found.

__Reference__

1. http://forum.ubuntuusers.de/topic/paketabhaengigkeit-mit-ia32-libs-kann-nicht-au/2/
2. http://askubuntu.com/questions/138530/why-do-i-get-an-unmet-dependencies-error-when-trying-to-install-wine