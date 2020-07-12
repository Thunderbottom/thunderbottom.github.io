---
layout: post
title: Installing Termite on Void Linux
category: "Systems"
---

Termite is a VTE-based terminal emulator for GNU/Linux.

It is available as a package on most Linux distribution repositories, but not on Void Linux. This is because termite uses a custom VTE build which isn't (and won't be) included in the void packages repository.[^void-packages]

In this tutorial I show you how to build and install termite on Void Linux.

Even though termite is a pretty solid choice for a terminal emulator, I suggest checking out Alacritty[^alacritty], which is a GPU-accelerated terminal emulator and is [available](https://github.com/void-linux/void-packages/blob/master/srcpkgs/alacritty/template) in the Void Linux repository.
{: .message}

## Build Dependencies

We need to install some dependencies to set up the custom VTE build and compile termite. Luckily enough, all the required dependencies are available in the Void Linux repository.

```sh
$ sudo xbps-install -Sy git gcc make automake autoconf gtk-doc glib-devel \
	vala-devel gobject-introspection pkg-config intltool \
	gettext-devel gnutls gnutls-devel gtk+3 gtk+3-devel \ 
	pango pango-devel gperf pcre2-devel
```

## Building VTE-ng

Make sure all of the dependencies have been installed before proceeding. We'll now be compiling a custom VTE build, VTE-ng. If you do not have a build directory yet, making one would help with better organization.

```sh
$ mkdir build && cd build
```

We then need to clone the VTE-ng git repository and checkout the latest versioned branch.

```sh
$ git clone https://github.com/jelly/vte-ng.git
$ cd vte-ng
$ git checkout 0.50.2-ng
```

It is now time to configure and build vte-ng. We need to add `--prefix=/usr` while configuring so that it installs the library to `/usr` and not `/usr/local`, which is not used by Void Linux. Unless you choose to add it to your `$PATH`, of course.

```sh
$ ./autogen.sh --prefix=/usr
$ make
$ sudo make install
```

That's it for the custom VTE build.

## Building Termite

To build and install termite, we need to go back to the build directory and clone the termite git repository. We'll clone it recursively as termite requires some git submodules.

```sh
$ cd $BUILDDIR
$ git clone --recursive https://github.com/thestinger/termite.git
$ cd termite
```

We do not need to configure anything in here since everything is hardcoded in the Makefile. Although we still need to edit the Makefile to make termite install to `/usr` and not `/usr/local`. We can do this using sed, and then finally make and install termite.

```sh
$ sed 's/PREFIX = \/usr\/local/PREFIX = \/usr/' -i Makefile
$ make
$ sudo make install
```

That's all. If everything goes well, you should end up with termite installed on your system.

[^void-packages]: [GitHub: void-packages](https://github.com/void-linux/void-packages)
[^alacritty]: [Alacritty: A cross-platform, GPU-accelerated terminal emulator](https://github.com/alacritty/alacritty)
