---
title: "Installing the HTML LCARS Screensaver on MacOS"
description: "A guide on installing an LCARS based screensaver on macOS"
author: "Stan Jewhurst"
date: "2021-08-25"
#url:
cover:
  image: images/lcars.jpg
  #caption: A featured image!
tags:
  - tutorial
  - macos
  - star-trek
ShowToc: false
draft: false
---
Years ago I was introduced to the fantastic [System47 LCARS](https://www.mewho.com/system47) screensaver through a random Google search. For years this was my goto screensaver of choice, but the project hasn't been maintained in years, and with the recent death of Adobe Flash (which was a requirement for this screensaver) running it on MacOS is no longer possible (not to metntion M1 Macbooks).

While screensavers are somewhat out of fashion, I have always been a fan of the nostalgia that this one brings about. Not only that, but it's a highly accurate depiction of the version used on the Star Trek series, so I was very pleased when I learned that a developer on Github know as [webOSpinn](https://github.com/webOSpinn/) had created an HTML version of this screensaver using the now defunct Google Swiffy runtime. 

I started by cloning the [repository](https://github.com/webOSpinn/System47):

```bash
git clone https://github.com/webOSpinn/System47
```

This provides several versions of the HTML code, converted using different versions of the Swiffy runtime. I was interested in the latest version of which there are two files:

```bash
stan@Pidgeotto ~/System47 (git)-[master] % ls -l | grep 8.0
-rw-r--r--@ 1 stan staff 1519622 25 Aug 09:28 v8.0.html
-rw-r--r-- 1 stan staff 1453993 25 Aug 09:28 v8.0_noaudio.html
```

Next, we need a program which can take a website and display it as a webpage. Thankfully another developer, [liquidx][1], has just the thing - [WebViewScreensaver][2]. Installation insructions are included, however as I use **homebrew** to manage my Mac, installation was incredibly straightforward:

```zsh
stan@Pidgeotto ~/System47 (git)-[master] % brew install webviewscreensaver
Updating Homebrew…
==> Auto-updated Homebrew!
Updated 1 tap (homebrew/core).
==> Updated Formulae
Updated 221 formulae.
==> Downloading https://github.com/liquidx/webviewscreensaver/releases/download/v2.2.1/WebViewScreenSaver-2.2.1.zip
==> Downloading from https://github-releases.githubusercontent.com/847378/d25f5c80-dfe2-11eb-816c-78779ac366b9?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20210825%2Fus-east
################################################################## 100.0%
==> Installing Cask webviewscreensaver
==> Moving Screen Saver 'WebViewScreenSaver.saver' to '/Users/stan/Library/Screen Savers/WebViewScreenSaver.saver'
🍺 webviewscreensaver was successfully installed!</pre>
```

With this installed, a new screensaver is now listed under screensavers:

![](images/lcars-ss-1.png)

It won't work immediately at first - I had to override Gatekeeper (in **System Preferences > Security & Privacy** **>** **General**). After that, I configured the path to the **v8.0_noaudio.html** under Screen Saver Options:

![](images/lcars-ss-2.png)

And Voila! A working Star Trek LCARS screensaver on an M1 MacBook Air.

 [1]: https://github.com/liquidx
 [2]: https://github.com/liquidx/webviewscreensaver