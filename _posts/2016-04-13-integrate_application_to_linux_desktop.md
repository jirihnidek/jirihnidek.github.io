---
title: Integrate Application to Linux Desktop
layout: post
---

This post is about integration of desktop application to Linux Desktop and I will demonstrate it at [SublimeText](http://www.sublimetext.com/). It is great text editor and I'm writing this text in this editor right now, but installation at Linux can be pain in ass, because it is distributed as compressed tarball. No, I don't use Ubuntu and I can't install it using deb package. On other hand the tarball includes desktop file and other resources that can help us.

This tutorial can be also helpful, when you try to install any other proprietary desktop software or understand little more to moder Linux desktop. This tutorial is focused mostly on [Gnome 3](https://www.gnome.org/) desktop, but it should work in [KDE](https://www.kde.org/) and other Linux desktop systems.

When you [download](http://www.sublimetext.com/3) SublimeText tarball, decompress it and move it to `/opt` directory:

    $ sudo tar -jxf sublime_text_3-some-verson.tar.bz2
    $ sudo mv sublime_text /opt

Then you can find file `sublime_text.desktop` in `/opt/sublime_text`. It includes following text:

    [Desktop Entry]
    Version=1.0
    Type=Application
    Name=Sublime Text
    GenericName=Text Editor
    Comment=Sophisticated text editor for code, markup and prose
    Exec=/opt/sublime_text/sublime_text %F
    Terminal=false
    MimeType=text/plain;
    Icon=sublime-text
    Categories=TextEditor;Development;
    StartupNotify=true
    Actions=Window;Document;

    [Desktop Action Window]
    Name=New Window
    Exec=/opt/sublime_text/sublime_text -n
    OnlyShowIn=Unity;

    [Desktop Action Document]
    Name=New File
    Exec=/opt/sublime_text/sublime_text --command new_file
    OnlyShowIn=Unity;

The directory also includes directory `Icons` with icon files and you have to place all these files to proper directories. No, you will not to use many cp commands and restart you desktop. Modern Linux desktop offers some tools that should simplify installation it is `xdg-desktop-icon` and `xdg-icon-resource`.

Let focus at item `Icon=sublime-text`. Why isn't there some some path to the file with icon. Well, there could be path to icon file. But consider this. Will be one file with one resolution the best resolution for all use cases? No, it will not be, because Start-like menus require small icons and Dock-like launchers requires icons with big resolution. It means that desktop should be able to find icon with desired resolution.

Icon Files
----------

First of all we will install all icons:

    $ cd /opt/sublime_text/Icon
    $ sudo for dir in `ls`; do \
        xdg-icon-resource install --novendor --context apps --size ${dir#*x} ${dir}/sublime-text.png sublime-text; \
      done

All these icons are installed to subdirectories of '/usr/share/icons/hicolor'. It is installed in 'hicolor' directory, because we didn't specified any icon theme using '--theme' parameter.

Menu Icon File
--------------

When all resolution of icons are installed in proper directories, then we can install `sublime_text.desktop` file using:

    $ cd /opt/sublime_text
    $ sudo xdg-desktop-menu install --novendor --mode system sublime_text.desktop

After this you will be able to launch SublimeText using menu or other favorite application launcher. When you install .desktop file using `xdg-desktop-menu`, then desktop file will be copied to `/usr/local/share/applications`. When you don not have root account or you are not in sudousers, then you can install desktop menu using:

    $ xdg-desktop-menu install --novendor --mode user sublime_text.desktop

> Note: When some desktop application is installed from RPM package, then corresponding desktop file(s) are installed to directory `/usr/share/applications` and command `update-desktop-database /usr/share/applications` is called. This command is also internally called by `xdg-desktop-menu`.

Desktop Icon File
-----------------

If you want to see this icon launcher at your Linux desktop and your desktop support displaying icons at desktop, then you have to type:

    $ cd /opt/sublime_text
    $ xdg-desktop-icon install --novendor sublime_text.desktop

Mime Types
----------

Association of mime type to application can be done in desktop file:

    MimeType=text/plain;

But how to define new mime type or change default association of mime-type to application. To query information about file types and defining new file types you can use command `xdg-mime`. We can see that SublimeText is able to open document with mime type `text/plain`. What is default application for opening this mime type? Is it SublimeText?

    $ xdg-mime query default text/plain

It will be default desktop editor. If you use Gnome, then it will be: `org.gnome.gedit.desktop`. To change default text editor you have to type:

    $ xdg-mime default sublime_text.desktop text/plain

You can check changed settings clicking at some .txt file or re-run query command.

It could seem that SublimeText do not have some own file type, but you can use "Projects" in SublimeText and definition of this project is save to file with suffix `.sublime-project`.

To query some project file we can use:

    $ xdg-mime query filetype myproject.sublime-project

we will get: `text/plain`.

To define new mime type we have to create following XML file named `sublime_text-project.xml`:

```xml
<?xml version="1.0"?>
<mime-info xmlns='http://www.freedesktop.org/standards/shared-mime-info'>
    <mime-type type="text/sublime-project">
        <comment>SublimeText project file</comment>
        <glob pattern="*.sublime-project"/>
    </mime-type>
</mime-info>
```

To install new mime type run:

    $ sudo xdg-mime install sublime_text-project.xml

After installing new mime type you can check your project file:

    $ xdg-mime query filetype myproject.sublime-project

It should return: `text/sublime-project` now. OK, we have new mime type, but we would like to handle it properly now. How to deal with this new mime type in SublimeText? Well, let have a look at command-line options of SublimeText. Running: `sublime_text --help` prints this help:

    Sublime Text build 3103

    Usage: sublime_text [arguments] [files]         edit the given files
       or: sublime_text [arguments] [directories]   open the given directories

    Arguments:
      --project <project>: Load the given project
      --command <command>: Run the given command
      -n or --new-window:  Open a new window
      -a or --add:         Add folders to the current window
      -w or --wait:        Wait for the files to be closed before returning
      -b or --background:  Don't activate the application
      -h or --help:        Show help (this message) and exit
      -v or --version:     Show version and exit

    Filenames may be given a :line or :line:column suffix to open at a specific
    location.

We can see that SublimeText project can be opened using: `sublime_text --project myproject.sublime-project`. If we want to open it clicking at this file, we have to modify `sublime_text.desktop` file. We have to add following section:

    [Desktop Action Project]
    Name=SublimeText Project
    Exec=/opt/sublime_text/sublime_text --project %F
    Terminal=False
    MimeType=text/sublime-project

And we have to re-run:

    $ cd /opt/sublime_text
    $ sudo xdg-desktop-menu install --novendor --mode system sublime_text.desktop\

That's all folks.