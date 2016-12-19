---
title: VirtualGL
layout: post
---

This post is about [VirtualGL](http://www.virtualgl.org) technology. It enables to render OpenGL content at powerful GPU card of remote server and sends only "images" to the client. It can be used with any Linux application using OpenGL and it can also solve problems with Indirect GLX.

When you use CentOS 7 or any other Linux OS, then you probably uses X.org that do not support Indirect GLX (IGLX). What the hell is this? It is technology that allows you to run OpenGL application on remote server and sends all OpenGL calls to client through SSH to local client and render OpenGL at local GPU. Well, to be honest. X.org supports IGLX, but you have to run X.org with `+idx` parameter and it is not easy to set up. Especially if you use `gdm`. Why is IGLX forbidden by default? There are some security reason for that and yes, bad boys try to get us, but wait. We need to do science and make things done. How to deal with that? There is project called [VirtualGL](http://www.virtualgl.org/) and it can compute all OpenGL stuff at GPU of remote server and send only rendered images to the client. It is possible to install VirtualGL from EPEL repository using:

    $ sudo yum install VirtualGL # remote server & client

When VirtualGL is installed, then you have to configure it at remote server using:

    $ sudo vglserver_config # remote server

It is recommended to add users working with ParaView to group: `vglusers`

    $ sudo usermod -a -G vglusers user.name # remote server

Before you connect to remote server, then it is required to run VirtualGL client on client machine:

    $ vglclient -force # client

If you want to use VirtualGL on remote server, then you can not connect to the remote server using `ssh` command, but you have to use:

    $ vglconnect -s user.name@remote.server.org # client

You can test running OpenGL application at remote server:

    $ vglrun glxinfo

It should print lot of information:

    name of display: localhost:10.0
    display: localhost:10  screen: 0
    direct rendering: Yes
    server glx vendor string: VirtualGL
    server glx version string: 1.4
    server glx extensions:
        GLX_ARB_create_context, GLX_ARB_create_context_profile, 
        GLX_ARB_get_proc_address, GLX_ARB_multisample, GLX_EXT_swap_control, 
        GLX_EXT_texture_from_pixmap, GLX_EXT_visual_info, GLX_EXT_visual_rating, 
        GLX_SGIX_fbconfig, GLX_SGIX_pbuffer, GLX_SGI_make_current_read, 
        GLX_SGI_swap_control, GLX_SUN_get_transparent_index
    client glx vendor string: VirtualGL
    client glx version string: 1.4
    client glx extensions:
        GLX_ARB_create_context, GLX_ARB_create_context_profile, 
        GLX_ARB_get_proc_address, GLX_ARB_multisample, GLX_EXT_swap_control, 
        GLX_EXT_texture_from_pixmap, GLX_EXT_visual_info, GLX_EXT_visual_rating, 
        GLX_SGIX_fbconfig, GLX_SGIX_pbuffer, GLX_SGI_make_current_read, 
        GLX_SGI_swap_control, GLX_SUN_get_transparent_index
    GLX version: 1.4
    GLX extensions:
        GLX_ARB_create_context, GLX_ARB_create_context_profile, 
        GLX_ARB_get_proc_address, GLX_ARB_multisample, GLX_EXT_swap_control, 
        GLX_EXT_texture_from_pixmap, GLX_EXT_visual_info, GLX_EXT_visual_rating, 
        GLX_SGIX_fbconfig, GLX_SGIX_pbuffer, GLX_SGI_make_current_read, 
        GLX_SGI_swap_control, GLX_SUN_get_transparent_index
    OpenGL vendor string: NVIDIA Corporation
    OpenGL renderer string: Quadro K5200/PCIe/SSE2
    OpenGL core profile version string: 4.4.0 NVIDIA 367.35
    OpenGL core profile shading language version string: 4.40 NVIDIA via Cg compiler
    ...

You can also test running some application with GUI:

    $ vglrun glxgears

When you want to use something more complex (like [Blender](http://www.blender.org)), then it is of course possible too. Enjoy!