---
title: "WSL, Hyper and Vagrant"
author: "Stan Jewhurst"
description: "Building a local test environment with WSL, using Hyper terminal and Vagrant"
date: 2018-08-16
url: /wsl-hyper-and-vagrant/
tags:
  - wsl
  - linux
  - vagrant
draft: false
ShowToc: true
TocOpen: false
disableShare: false
hideSummary: false
searchHidden: false
ShowReadingTime: false
ShowBreadCrumbs: false
ShowPostNavLinks: true
cover:
  image: images/cover-image.webp
---
The first laptop I ever purchased was the excellent [Black MacBook][1] released by Apple in 2006. The 2nd generation of MacBooks to include Intel processors, once I'd got to grips with the UI and a few keyboard shortcuts, I realised just how powerful a UNIX-based operating system could be.

Fast forward 12 years and I'm writing this on a Dell XPS 13. I wish I had a MacBook Pro, or even the dinky 12" MacBook but when it comes to cost vs performance, Apple have priced themselves out of my reach. Likewise, the majority of businesses still use Windows-based platforms for day to day work and so, when my day to day work required the use of tools like **ssh**, **curl**, **python** and others, I started looking for a more integrated solution for Windows than PuTTY and a Linux VM.

### WSL

Windows Subsystem for Linux arrived shortly after the initial release of Windows 10, but has really come into its own in the last year. The capability to install different Linux Subsystems (rather than just Ubuntu, which was the only available at first) has made it a far more flexible platform. Currently Ubuntu, Debian, SUSE, OpenSUSE and Kali (BackTrack) are available and can be installed with a couple of clicks.

Firstly, open a Powershell prompt as Administrator and run:

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux
```

to enable the WSL feature. Next head to [the Windows Store][2] to choose your distro (I'm a fan of Debian) and install. You'll be asked to create a UNIX username and password for the new environment and after a few moments you'll be dropped to a bash shell, familiar to anyone who's used GNU/Linux in the past. Not only will you now have the standard Linux commands at your fingertips, but you also have access to package managers and linux packages - most of which work natively. After a few additions:

```bash
sudo apt-get install zsh vim wget inetutils-ping curl git python python-pip
```

I'm almost ready to start work.

### Hyper

While you can continue to launch your environment from the Start menu or by invoking bash from PowerShell, I wanted something that allowed me to run multiple sessions at once without windows open everywhere; Enter [Hyper][3]. Hyper is a terminal emulator based on javascript and CSS. It's not the most lightweight of terminals, but it's flexible and pretty (who doesn't like eye-candy)?

> Editor's Note (I guess that's still me) - I lost this image during a migration at some point.

Hyper is configured with a simple textfile and by default on Windows launches **cmd.exe**. This can be changed by locating and modifying the <mark>shell</mark> attribute:

```js
shell: 'C:\\Windows\\System32\\wsl.exe',
```

Most guides suggest changing the above to `-zsh.exe` as that is the default shell installed, but notice earlier I downloaded and installed **zsh**, my shell of choice. Improvements to WSL now allow you more control over your shell, and it's easy to change from bash to zsh. Within your environment run:

```bash
chsh -s $(which zsh)
```

and then invoke your environment using **wsl.exe** as above.

### Vagrant

While WSL and Hyper allow me to get on with my day to day work without the need for a virtual environment, there are still limitations (for example, networking under WSL is a bit broken and abstracted from Windows). When it comes to rapid testing or prototyping, [Vagrant][4] makes spinning up (and tearing down) virtual environments a cinch. It natively integrates with [VirtualBox][5] and with one tweak can work perfectly with WSL.

First, [download and install Virtualbox][6] and its extension pack. Next, grab an [install package for Vagrant][7] on your chosen distro. I chose Debian 64-bit. This can then be installed:

```bash
sudo dpkg -i vagrant_2.1.2_x86_64.deb
```

Only one tweak is required to get this working under WSL which is to run the following connand:

```bash
export VAGRANT_WSL_ENABLE_WINDOWS_ACCESS="1"
```

to ensure WSL is allowed to interface correctly with Windows. For ease,  I added this to my zshrc file to save me doing it every time I start a shell. Finally, create and boot a test environment:

```bash
% mkdir vagrant
% mkdir vagrant/test
% cd vagrant/test
% vagrant init
A `Vagrantfile` has been placed in this directory. You are now ready to `vagrant up` your first virtual environment! Please read the comments in the Vagrantfile as well as documentation on 'vagrantup.com' for more information on using Vagrant.
```

You'll need to modify the Vagrantfile to use a valid vagrant image. There is a wide selection available on [Vagrant's website][8] as well as many more on github. For demonstration you could modify the vagrantfile with:

```ruby
Vagrant.configure("2") do |config|
     config.vm.box = "ubuntu/bionic64"
end
```

and then

```bash
% vagrant up
```

Vagrant will go off and download the VM Image, boot it, set up a local port forward and install SSH keys.

> Editor's Note: Another runaway image

Once complete accessing your new environment is as easy as:

```bash
% vagrant ssh
```

Should you wish you can suspend, resume and halt the environment:

```bash
% vagrant suspend | halt | resume
```

or if you're finished with the environment completely:

```bash
% vagrant destroy
```

which will remove all traces of the virtual machine.

_P.S. I hit some issues booting this Ubuntu VM and needed to add the following to my basic Vagrantfile:_

```ruby
config.vm.provider "virtualbox" do |vb|
  vb.customize [ "modifyvm", :id, "--uartmode1", "disconnected" ]
end
```

 [1]: https://everymac.com/systems/apple/macbook/specs/macbook-core-2-duo-2.0-black-13-specs.html
 [2]: https://aka.ms/wslstore
 [3]: https://hyper.is/
 [4]: https://www.vagrantup.com/
 [5]: https://www.virtualbox.org
 [6]: https://www.virtualbox.org/wiki/Downloads
 [7]: https://www.vagrantup.com/downloads.html
 [8]: https://app.vagrantup.com/boxes/search?utf8=%E2%9C%93&sort=downloads&provider=&q=ubuntu+bionic
