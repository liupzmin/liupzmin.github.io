---
title:  使用普通用户管理docker
date: 2019-09-24 16:09:12
tags: docker
categories: docker
---


>Docker的Daemon程序绑定到socket文件上（`/var/run/docker.sock`），而不是tcp端口.因此，默认情况下这个socket文件只能被`root`用户或者拥有`sudo`权限的用户访问. Docker daemon总是以`root`用户运行。

如果你不想总是在docker命令的前边加上`sudo`，那么可以创建一个名为`docker`的group，并且将你的用户加入到该组，那么docker daemon启动的时候会创建一个`docker`组成员可以访问的socket，例如：

```shell
ll /var/run/docker.sock 
srw-rw---- 1 root docker 0 Sep 24 11:24 /var/run/docker.sock
```

To create the `docker` group and add your user:

1. Create the `docker` group.

    ```shell
    $ sudo groupadd docker
    ```

2. Add your user to the `docker` group.
    ```shell
    $ sudo usermod -aG docker $USER
    ```

3. Log out and log back in so that your group membership is re-evaluated.
   If testing on a virtual machine, it may be necessary to restart the virtual machine for changes to take effect.
   On a desktop Linux environment such as X Windows, log out of your session completely and then log back in.
   On Linux, you can also run the following command to activate the changes to groups:

   ```shell
    $ newgrp docker 
   ```