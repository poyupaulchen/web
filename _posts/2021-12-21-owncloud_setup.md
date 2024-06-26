---
layout: post
title: OwnCloud Server Setup
subtitle: Set up OwnCloud server on Windows with WSL
tags: [server, web development]
author: Yuanpeng Zhang
comments: true
use_math: true
---

<p align='center'>
<img src="/assets/img/posts/ownc_setup.png"
   style="border:none;"
   alt="ownc_setup"
   title="ownc_setup" />
</p>

<p align='center'>
<br />
</p>

👉 **Ports configuration**

<p style='text-align: justify'>
Refer to the following links to gist for configuration files concerning the ports configuration,
</p>

[/etc/apache2/ports.conf](https://draft.blogger.com/blog/post/edit/713170236114697752/459863480324374329#)

[/etc/apache2/sites-enabled/000-default.conf](https://draft.blogger.com/blog/post/edit/713170236114697752/459863480324374329#)

<br />

👉 **Connection to WSL Apache server from outside (i.e., beyond localhost)**

<p style='text-align: justify'>
<a target="_blank" href="https://draft.blogger.com/blog/post/edit/713170236114697752/459863480324374329#">This link</a> provide detailed explanation about the IP address assignment scheme for WSL 2 and we need to follow the instruction there to establish the connection from outside to WSL Apache server. Also, since each time when we launch the WSL, new IP address will be assigned to the virtual system and we have no way to make the IP address static. In this case, we can follow <a target="_blank" href="https://draft.blogger.com/blog/post/edit/713170236114697752/459863480324374329#">this link</a> to grab the IP address of the virtual WSL system automatically and connect it to our Windows host. The following function can be put in powershell profile file so that we can do the configurations all-in-once,
</p>

```
Function own {
    wsl sudo /etc/init.d/mysql start
    wsl sudo service apache2 restart
    $wsl_ip = (wsl hostname -I).trim()
    netsh interface portproxy add v4tov4 listenport=5051 listenaddress=0.0.0.0 connectport=5051 connectaddress=$wsl_ip
}
```

<br />

👉 **Port confliction & Firewall issues**

<p style='text-align: justify'>
Sometimes, if the connection cannot be established via public IP address (or domain name if configured), there could be multiple reasons. First, we may want to change the Apache configuration to let it listen to another alternative port, since sometimes other services (e.g., Jupyterlab) will take over the port used by Apache. Second, we want to make sure the rule for a specific port is added in Windows Firewall exception (inbound traffic).
</p>

<br />

👉 **Configure external drive**

<p style='text-align: justify'>
By default, onwCloud will use '/var/www/owncloud/data' as the data directory, where all files would be stored. Also, by default, using external local drive as storage is disabled (for safety reason). To enable this, we can refer to Ref. [3].
</p>

<br />

<b>References</b>

[1] [https://www.how2shout.com/how-to/how-to-install-owncloud-server-on-windows-10-wsl.html](https://www.how2shout.com/how-to/how-to-install-owncloud-server-on-windows-10-wsl.html)

[2] [https://lucidar.me/en/owncloud/install-owncloud-server-on-ubuntu-20-04/](https://lucidar.me/en/owncloud/install-owncloud-server-on-ubuntu-20-04/)

[3] [https://github.com/owncloud/core/pull/27590](https://github.com/owncloud/core/pull/27590)
