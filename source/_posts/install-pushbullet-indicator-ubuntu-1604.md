---
title: Install Pushbullet Indicator on Ubuntu 16.04
date: 2016-09-16 00:06:19
tags:
---
Recently after install the latest ElementaryOS, I found myself wanting to have a Pushbullet indicator in the wingpanel.

A quick Google-search showed that such an indicator exists, but I couldn't find a package for 16.04 in their PPA.

A [StackOverflow answer](http://askubuntu.com/a/804948) shows that the package for 16.04 is in another PPA. A minute after I had the indicator installed, follow the commands below to install it.

```bash
sudo add-apt-repository ppa:atareao/pushbullet
sudo apt update
sudo apt install pushbullet-indicator
```

If you get a command not found error while trying to run the first command, make sure you have the `software-properties-common` package installed.