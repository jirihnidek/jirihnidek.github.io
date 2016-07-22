---
title: ParaView and Server/Cluster
layout: post
---

[ParaView](http://www.paraview.org) is Open Source program for visualization of scientific data. The big advantage of this software is its ability to run data processing and rendering on remote server or remote cluster. This tutorial will guide you to run ParaView servers at CentOS 7 Linux server and ParaView client at CentOS 7 as well.

Requirements
------------

First of all we have to download and install Paraview at server. You can install Praview from [EPEL](https://fedoraproject.org/wiki/EPEL) repository using following command:

    $ sudo yum install paraview # remmote server & client

If you use cluster, then you also can choose two other versions of ParaView in this repository: `paraview-openmpi` and `paraview-mpich`.

> Note: The EPEL repository usually does not include the latest version of ParaView. If you want to use the latest version, then you will have to download Paraview from [official site](http://www.paraview.org/download/).

First of all you will have to install and run X server at remote server. It is wise to install some desktop environment too. It is possible to do using:

    $ sudo yum groups install "GNOME Desktop" # remote server

When X.org is installed and running, then you have to login using gdm, kdm or any other display manager. Then open virtual terminal (gnome-terminal, xterm, etc.) and run:

    $ xhost + # desktop of remote server

It will enable opening GUI application for any user from any host. It is not recommended to use it in production, but for first testing it is sufficient. You can test it using xterm command (could be installed with: `yu m-y install xterm`). Login to remote server using SSH:

    $ ssh remote.server.org # client

And try to run xterm:

    $ xterm -display :0 # remote server

The new window should be opened at screen of remote server and not at screen of your client.

> Note: it is recommended to set up auto-login for one user and start `xhost +` after login.

Running Server
--------------

We are ready now to run ParaView at remote server. We have to options how to do it. When we have only one server for processing data and rendering, then we should run ParaView server using:

    $ pvserver -display :0 # remote server

When you want to split data processing and rendering between two servers, then it is possible. Keep in mind that following method is not recommended for situation with one machine, because splitting data processing and rendering causes useless overhead.

First start data server at machine dedicated for data processing:

    $ pvdataserver # remote data server

The output is following:

    Waiting for client...
    Connection URL: cdsrs://data.server.org:11111/<render-server-hostname>:22221
    Accepting connection(s): data.server.org:11111

and run render server at machine dedicated for rendering:

    $ pvrenderserver -display :0 # remote render server

The output of render server should be something like this:

    Waiting for client...
    Connection URL: cdsrs://<data-server-hostname>:11111/render.server.org:22221
    Accepting connection(s): render.server.org:22221

> Note: number of ports can be different at your installation and we will have to open these ports at firewall and we will need it during setup of connection to server.

Next step is opening ports at server:

    firewall-cmd --add-port=11111/tcp # pvserver or pvdataserver
    firewall-cmd --add-port=22221/tcp # renderserver

Then it is possible to connect to ParaView server from your desktop/laptop, but it can be done in better way and limit more connection to your server:

    firewall-cmd --add-rich-rule='rule family="ipv4" port port="11111" protocol="tcp" source address="147.230.0.0/16" accept limit value="1/s"'
    firewall-cmd --add-rich-rule='rule family="ipv4" port port="22221" protocol="tcp" source address="147.230.0.0/16" accept limit value="1/s"'

Testing
-------

Start ParaView at your client and click icon "Connect". Choose "Add Server" and fill items:

* Name: Some descriptive name
* Server Type: "Client/Server" when you use `pvserver` to start ParaView at remote server
* Host: hostname of you remote server
* Port: 11111

Click "Configure". At next dialog you can choose "Startup Type" as "Manual". In this case you will have to login to remote server using SSH and start `pvserver` manually. When the server is configured, then you can click at "Connect". After connection you will be able to open files at your remote server and start to process and visualize using power of your remote server/cluster.

GPU Monitoring
--------------

When you have GPu(s) from [nVidia](http://www.nvidia.com), then you can use command `nvidia-smi -l`. Intel and AMD provides similar Linux tools.

References
----------

[1] [http://www.paraview.org/Wiki/Setting_up_a_ParaView_Server](http://www.paraview.org/Wiki/Setting_up_a_ParaView_Server)