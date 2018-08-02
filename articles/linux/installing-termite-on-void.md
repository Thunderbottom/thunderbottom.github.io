Installing Termite on Void Linux
================================

Termite is a VTE-based terminal emulator for GNU/Linux. It is available on most Linux distribution repositories, but not on Void Linux. This is because termite uses a custom VTE build which isn't (and won't be) included in the void packages.

In this tutorial I show you how to build and install termite on Void Linux.

# Build Dependencies

We need to install some dependencies to build both custom VTE library and termite. All of the required dependencies are available in the Void Linux repository.

```
	$ sudo xbps-install -Sy git gcc make automake autoconf gtk-doc glib-devel \
				vala-devel gobject-introspection pkg-config intltool \
				gettext-devel gnutls gnutls-devel gtk+3 gtk+3-devel \ 
				pango pango-devel gperf pcre2-devel
```

# Building VTE-ng

Make sure all of the dependencies have been installed before proceeding. We'll now be compiling the custom VTE library, VTE-ng. If you do not have a build directory yet, making one would lead to a better organization.

```
	$ mkdir build && cd build
```

We then need to clone the VTE-ng git repository and checkout the latest versioned branch.

```
	$ git clone https://github.com/jelly/vte-ng.git
	$ cd vte-ng
	$ git checkout 0.50.2-ng
```

It's now time to configure and build vte-ng. We need to add `--prefix=/usr` while configuring so that it installs the library to `/usr` and not `/usr/local`, which is not used by Void Linux.

```
	$ ./autogen.sh --prefix=/usr
	$ make
	$ sudo make install
```

That's it for the custom VTE library.

# Building Termite

To build and install termite, we need to go back to the build directory and clone the termite git repository. We'll clone it recursively as termite uses some git submodules.

```
	$ cd ..
	$ git clone --recursive https://github.com/thestinger/termite.git
	$ cd termite
```

We do not need to configure anything in here since everything is hardcoded in the Makefile. Although, we still need to edit the Makefile to make termite install to `/usr` and not `/usr/local`. We can do this using sed, and then finally make and install termite.

```
	$ sed 's/PREFIX = \/usr\/local/PREFIX = \/usr/' -i Makefile
	$ make
	$ sudo make install
```

That's all. If everything goes well, you will end up with termite installed on your system.
