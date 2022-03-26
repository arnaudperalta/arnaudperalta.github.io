---
title: "WSL + X Server + i3 : The good way to develop on Windows."
date: 2022-03-25T23:27:58+01:00
---
During my university years, I searched a good way to work as a developper when I'm forced to work on Windows.

At the very beginning, I started the web developpment with WampServer and upgraded to WSL2 with a Ubuntu distribution. However, I was not really satisfied as my work environnment was not fully on GNU/Linux.

As a web developper, I always found out that working on a different OS than the deployment server was very silly. This is why I "fully" work in a linux environnment even if my computer is running Windows.

The first step is to install a WSL2 distribution, personally, I chose [ArchWSL](https://github.com/yuk7/ArchWSL) but you can install the one you want if it supports i3. 
![ArchWSL installed](/images/archwsl.png#center)

Once your distribution, you can create your user if it's not the case :

```shell
useradd -m -G wheel -s /bin/bash <username>
```

Then make it sudoer (by uncommenting the `%wheel ALL=(ALL:ALL) ALL` line) :
```shell
EDITOR=vim visudo
```

Change the password and switch user :
```shell
passwd <username>
su - <username>
```

The second step is to install a X Server. For this step you also have multiple choices : X410, MobaXterm, VcXsrv. I personally use [VcXsrv](https://sourceforge.net/projects/vcxsrv/) (free and simple to use).

Notes : For every X Server, you will need to change the DISPLAY environnment variable to connect i3 to the correct display address. For VcXsrv, you'll need to set these two lines in a `.bashrc` or `.zshrc` file for example.

```
export DISPLAY=$(ip route list default | awk '{print $3}'):0
export LIBGL_ALWAYS_INDIRECT=1
```

Notes : If you are using Windows 11 and want to use an other window manager like me, you will need to prevent WSLg from starting. You can do it by writing these two lines in your `/etc/wsl.conf` file.
```
[wsl2]
guiApplications=false 
```
Still for Windows 11, you'll need to install `yay` and the `dbus-x11` and `xorg-server` packages.


I suggest you to setup VcXsrv with these settings :
![VcXsrv settings](/images/vcx_window_settings.png#center)

Then I will need to disable the access control to let WSL access to the X server display.
![VcXsrv access control](/images/vcx_access_control.png#center)

Once your X Server is started and you did setup the DISPLAY variable you are now ready to install and start i3.

For an ArchWSL install :
```shell
sudo pacman -S i3
```

Install [rofi](https://github.com/davatorium/rofi) and a font for i3 :
```shell 
sudo pacman -S rofi
sudo pacman -S ttf-dejavu
```

As we are still a little bit on a Windows environnment, I suggest you to create a config file to set the i3 mod key to Left Alt (Mod1)

```shell
mkdir -p ~/.config/i3
vim ~/.config/i3/config
```

Write these lines in the i3 config file :
```nothing
set $mod Mod1
font pango:DejaVu Sans mono 10
bindsym $mod+d exec rofi -show run
```

Then we can launch it :
```shell
i3 &! > /dev/null
```

You will normally see the i3 default status bar in your X Server :

![i3 status bar](/images/i3_status.png#center)

If you press Alt+D, you will have access to the rofi menu where you can start applications.

![i3 rofi](/images/i3_rofi.png#center)

Congratulation, you can now enjoy the joy of working in a GNU/Linux environnment when you are forced to use Windows. You just need to install a terminal emulator and your IDE.
