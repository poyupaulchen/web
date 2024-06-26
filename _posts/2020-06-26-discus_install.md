---
layout: post
title: Install DISCUS version 6 on Windows 10
subtitle: Problem & solution
tags: [tool, software installation]
author: Yuanpeng Zhang
comments: true
use_math: true
---

<p align='center'>
<img src="/assets/img/posts/discus.png"
   style="border:none;"
   alt="discus"
   title="discus" />
<br />
</p>

<p style='text-align: justify'>
On my Windows 10 machine, I first tried to install DISCUS version 6 following the one-touch installation procedure and got the error relevant to 'HDF5'. Therefore, I turned to the manual installation procedures. During the installation, I have to install the 'libssl1.1' library manually. After successful installation of discus_suite on WSL ubuntu machine, I could not find the 'DiscusWSL' folder under 'C:\Users\DISCUS_INSTALLATION' as specified in the installation instruction. Instead, I have to manually copy the 'DiscusWSL' folder from WSL directory

<br />

`/home/discus/DIFFUSE_INSTALL/DiscusWSL`

<br />

to the Windows folder

<br />

`C:\Program Files (x86)`.

<br />

After the successful configuration following the manual installation instruction, I can lanuch the discus_suite terminal without problems. However, I came across the problem of not being able to start any xserver relevant service. For example, typing 'manual' within the 'suite' environment will give the error of something like 'cannot start display :0'. After a bit Googling, I found this is something to do with the xserver settings. I have to put the following line in the '.profile.local' file (this file will be called anytime discus_suite is started) on WSL system for the WSL subsystem to configure the display properly:

<br />

`export DISPLAY=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}'):0`

<br />

Then another problem pops up at this stage complaining about authentication failure, which is again relevant to the xserver stuff. I have to use the following command to start the 'vcxsrv' server to disable the authentication,

<br />

`"C:\Program Files\VcXsrv\vcxsrv.exe" :0 -ac -terminate -lesspointer -multiwindow -clipboard -wgl`

<br />

I haven't figured out the way to put the command above into the starting batch file of discus_suite, so currently I have to manually execute the command above before launching the discus_suite program. To make life easier, I created a shortcut and associate the shortcut with the command here. Then I put the shortcut into the starting up folder (open the Windows run box by pressing '#+r' - where '#' represents the Windows start key - and put 'shell:startup' in it, followed by pressing 'enter' to open up the starting up folder). In this way, every time Windows starts, the 'vcxsrv' will launch automatically with authentification disabled.
</p>