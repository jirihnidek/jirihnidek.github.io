---
title: GSettings and DConf
layout: post
---

Configuration of Gnome desktop was changed in Gnome 3.x. It doesn't use GConf anymore (was used in Gnome 2.x), but it uses DConf as back end and GSettings as high-level configuration system. When you want to set something system-wide, then it can looks like a little bit tricky, but I will show you that is quite simple.

## Browsing Configuration ##

First of all we will need to find, where is desired configuration stored. Gnome 3 does not use any text files for configuration. Gnome 3 uses magic (aka dconf), but don't be scared it is quite simple and easy to solve almost any problem.

For browsing dconf is best to use Dconf Editor. it can be installed with following command:

    # yum install dconf-editor

Then you can simply start this GUI application using:

    $ dconf-editor

As you can see, configuration of any Gnome 3 application has some unique **path**. Each **path** has at least one **key**, and **key** have some type. A type of key could be __boolean__, __string__, __integer__, __array of strings__.

## Changing Values ##

It is easy to browse and set some settings with `dconf-editor`, but when you set something with `dconf-editor`, then this setting is user specific and other users will have default values. We can simply prove it with command-line tool `gsettings`.

For example, we will want to show date in calendar. This setting has following path:

    org.gnome.shell.calendar

and one value: `show-weekdate`. We can check current value with this command:

    $ gsettings get org.gnome.shell.calendar show-weekdate

When we set this value for current user, then there will be value `true`, but other users will have still `false` in this value.

## Changing Default Value ##

Go to the directory `/etc/dconf/db/local.d/` and create there new file called (e.g.) `01-calendar` and place following content to this file:

    [org/gnome/shell/calendar]
    show-weekdate=false

Then run following command as root:

    # dconf update

All users should see date in calendar (after restarting gnome shell).

## References ##

1. https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Desktop_Migration_and_Administration_Guide/part-Configuration_and_Administration.html
2. https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Desktop_Migration_and_Administration_Guide/configuration-overview-gsettings-dconf.html
3. https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Desktop_Migration_and_Administration_Guide/custom-default-values-system-settings.html