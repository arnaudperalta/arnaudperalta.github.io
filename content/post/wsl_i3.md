---
title: "WSL + X Server + i3 : The good way to develop on Windows."
date: 2022-03-25T23:27:58+01:00
draft: true
---
During my university years, I searched a good way to work as a developper when I'm forced to work on Windows.

At the very beginning, I started the web developpment with WampServer and upgraded to WSL2 with a Ubuntu distribution. However, I was not really satisfied as my work environnment was not fully on GNU/Linux.

As a web developper, I always found out that working on a different OS than the deployment server was very silly. This is why I "fully" work in a linux environnment even if my computer is running Windows.

The first step is to install a WSL2 distribution, personally, I chose [ArchWSL](https://github.com/yuk7/ArchWSL) but you can install the one you want if it supports i3. 
![ArchWSL installed](/images/archwsl.png)

The second step is to install a X Server. For this step you also have multiple choices : X410, MobaXterm, VcXsrv. I personally use [VcXsrv](https://sourceforge.net/projects/vcxsrv/) (free and simple to use).

Notes : For every X Server, you will need to change the DISPLAY environnment variable to connect i3 to the correct display address. For VcXsrv, you'll need to set these two lines in a `.bashrc` or `.zshrc` file for example.

```
export DISPLAY=$(ip route list default | awk '{print $3}'):0
export LIBGL_ALWAYS_INDIRECT=1
```

Notes : If you are using Windows 11 and want to use an other window manager like me, you will need to prevent WSLg from starting. You can do it by writing these two lines in your `%userdata%/.wslconfig` file.
```
[wsl2]
guiApplications=false 
```

Once your X Server is started and you did setup the DISPLAY variable you are now ready to install i3.

For an ArchWSL install :
```shell
$ sudo pacman -S i3
```

Then we can launch it :
```shell
$ i3 &! > /dev/null
```
