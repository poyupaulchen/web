---
layout: post
title: Management of Jupyter kernels
subtitle:
tags: [programming, software installation, web development]
author: Yuanpeng Zhang
comments: true
use_math: true
---

<p align='center'>
<img src="/assets/img/posts/jupyter.png"
   style="border:none;"
   alt="jupyter"
   title="jupyter" />
</p>

<br />

<p style='text-align: justify'>
Having multiple kernels available for Jupyter service, we may need to manage them at some point, e.g., rename kernel names and remove kernels. First, to show all available kernels, run the following command,
</p>

<blockquote cite="">
jupyter kernelspec list
</blockquote>

<p style='text-align: justify'>
we will then see output like this,
</p>

<blockquote cite="">
Available kernels:

  python3    /opt/jupyterhub/lib64/python3.6/site-packages/ipykernel/resources

  diffpy     /usr/local/share/jupyter/kernels/diffpy

  py37       /usr/local/share/jupyter/kernels/py37

  python     /usr/local/share/jupyter/kernels/python
</blockquote>

<p style='text-align: justify'>
This is the full list of all available kernels on our machine and the directory specifies where the configuration file for each kernel is located. To change kernel name, we then need to go into the corresponding directory for a specific kernel and open the 'kernel.json' file to change the 'display_name' entry to whatever we want.

<br />

To remove a kernel, we want to use the following command,
</p>

<blockquote cite="">
jupyter kernelspec remove KERNEL_NAME
</blockquote>

<p style='text-align: justify'>
For example, if we want to remove the 'diffpy' kernel above, we want to execute,
</p>

<blockquote cite="">
jupyter kernelspec remove diffpy
</blockquote>

<p style='text-align: justify'>
Among all the available kernels, there is a special one - the 'base' JupyterHub kernel from kernelspec. The 'python3' kernel shown above is just the 'base' JupyterHub kernel. If we try to remove this kernel, it will complain that the kernel cannot be found. Here in Ref. [1] is given detailed description and solution to this issue. To remove this base kernel, as posted in Ref. [1], there are two main things to do, 1) rename one of the other kernels to the base kernel name [2] and 2) disable native kernel by changing Jupyter configuration file [3]. Concerning the second part, we may need to create Jupyter configuration file first if it is not already existing, as instructed in Ref. [4].
</p>

<br />

<b>References</b>

[1] [https://github.com/jupyterhub/jupyterhub/issues/2759](https://github.com/jupyterhub/jupyterhub/issues/2759)

[2] [https://github.com/jupyterhub/jupyterhub/issues/2759#issuecomment-538139098](https://github.com/jupyterhub/jupyterhub/issues/2759#issuecomment-538139098)

[3] [https://github.com/jupyterhub/jupyterhub/issues/2759#issuecomment-770025308](https://github.com/jupyterhub/jupyterhub/issues/2759#issuecomment-770025308)

[4] [https://testnb.readthedocs.io/en/latest/config.html](https://testnb.readthedocs.io/en/latest/config.html)