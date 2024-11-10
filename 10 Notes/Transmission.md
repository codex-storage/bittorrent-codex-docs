---
tags:
  - bittorrent
link: https://transmissionbt.com
source: https://github.com/transmission/transmission
related-to:
---

#bittorrent 

| link       | https://transmissionbt.com                   |
| ---------- | -------------------------------------------- |
| source     | https://github.com/transmission/transmission |
| related-to |                                              |

[[Learn BitTorrent]] client. One of the three popular clients, the other are [[Deluge]] and [[qBittorrent]], with the most attractive looking website. Comparing to the other two, Transmission does not depend on [[libtorrent-rasterbar]], but rather has its own implementation of the BitTorrent protocol.

> As noted in GitHub’s REDME, [Transmission's documentation](https://github.com/transmission/transmission/blob/main/docs/README.md) is currently out-of-date, but the team has recently begun a new project to update it and is looking for volunteers. If you're interested, please feel free to submit pull requests!

It supports all major OS and has a native macos support.

The latest release is from 30 May 2024.

### How does it look like?

![[Pasted image 20241107085814.png]]
or in light mode:

![[Pasted image 20241107091409.png]]
![[Pasted image 20241107092811.png]]

Transmission does not seem to support version 2 torrent files and magnet links.

#### Settings

- Network
	![[Pasted image 20241107094119.png]]
- Peers
		![[Pasted image 20241107094219.png]]
Transmission has the most limited settings that are available to the user.
There are also GTK and QT versions, which on a Mac look just terrible... (I did not yet check how they build and look on ubuntu).

### Building

After cloning, building Transmission on macOS went smoothly. Only after adding `gtk` and `qt` dependencies started to brake the build.
Their CMake configuration is quite clean, yet, once it detects GTK it goes through some "branches" even when they are not needed for the given build configuration. Thus if you installed GTK deps:

```
brew install gtk4 gtkmm4
```

You will start getting errors like this:

```bash
CMake Error in gtk/CMakeLists.txt:
  Imported target "transmission::gtk_impl" includes non-existent path

    "/Library/Developer/CommandLineTools/SDKs/MacOSX13.sdk/usr/include/ffi"

  in its INTERFACE_INCLUDE_DIRECTORIES.  Possible reasons include:

  * The path was deleted, renamed, or moved to another location.

  * An install or uninstall procedure did not complete successfully.

  * The installation package was faulty and references files it does not
  provide.
```

Well, the problem here is wrong macOS version. To fix the build, first check which macOS path is the correct one for your system:

```bash
xcrun -sdk macosx --show-sdk-path
```

and then retrieve the paths currently included by running:

```bash
pkg-config --cflags gtkmm-4.0
pkg-config --cflags gtk4
```

Then in `CMakeLists.txt` manually add the combined paths returned by the `pkg-config` commands that you just run.

In my case, I changed:

```CMake
if(GTK_FOUND)
    add_library(transmission::gtk_impl INTERFACE IMPORTED)

    target_compile_options(transmission::gtk_impl
        INTERFACE
            ${GTK${GTK_VERSION}_CFLAGS_OTHER})

    target_include_directories(transmission::gtk_impl
        INTERFACE
            ${GTK${GTK_VERSION}_INCLUDE_DIRS})

    target_link_directories(transmission::gtk_impl
        INTERFACE
            ${GTK${GTK_VERSION}_LIBRARY_DIRS})

    target_link_libraries(transmission::gtk_impl
        INTERFACE
            ${GTK${GTK_VERSION}_LIBRARIES})
endif()
```

to:

```CMake
if(GTK_FOUND)
    add_library(transmission::gtk_impl INTERFACE IMPORTED)

    target_compile_options(transmission::gtk_impl
        INTERFACE
            ${GTK${GTK_VERSION}_CFLAGS_OTHER})

    target_include_directories(transmission::gtk_impl
        INTERFACE
            /opt/homebrew/Cellar/gtk4/4.16.3/include/gtk-4.0 
            /opt/homebrew/Cellar/pango/1.54.0/include/pango-1.0 
            /opt/homebrew/Cellar/harfbuzz/10.0.1_1/include/harfbuzz 
            /opt/homebrew/Cellar/fribidi/1.0.16/include/fribidi 
            /opt/homebrew/Cellar/graphite2/1.3.14/include 
            /opt/homebrew/Cellar/gdk-pixbuf/2.42.12/include/gdk-pixbuf-2.0 
            /opt/homebrew/Cellar/libtiff/4.7.0/include 
            /opt/homebrew/opt/zstd/include 
            /opt/homebrew/Cellar/xz/5.6.3/include 
            /opt/homebrew/Cellar/jpeg-turbo/3.0.4/include 
            /opt/homebrew/Cellar/cairo/1.18.2/include/cairo 
            /opt/homebrew/Cellar/fontconfig/2.15.0/include 
            /opt/homebrew/opt/freetype/include/freetype2 
            /opt/homebrew/opt/libpng/include/libpng16 
            /opt/homebrew/Cellar/libxext/1.3.6/include 
            /opt/homebrew/Cellar/libxrender/0.9.11/include 
            /opt/homebrew/Cellar/libx11/1.8.10/include 
            /opt/homebrew/Cellar/libxcb/1.17.0/include 
            /opt/homebrew/Cellar/libxau/1.0.11/include 
            /opt/homebrew/Cellar/libxdmcp/1.1.5/include 
            /opt/homebrew/Cellar/pixman/0.42.2/include/pixman-1 
            /opt/homebrew/Cellar/graphene/1.10.8/include/graphene-1.0 
            /opt/homebrew/Cellar/graphene/1.10.8/lib/graphene-1.0/include 
            /opt/homebrew/Cellar/glib/2.82.2/include 
            /opt/homebrew/Cellar/glib/2.82.2/include/glib-2.0 
            /opt/homebrew/Cellar/glib/2.82.2/lib/glib-2.0/include 
            /opt/homebrew/opt/gettext/include 
            /opt/homebrew/Cellar/pcre2/10.44/include 
            /opt/homebrew/Cellar/xorgproto/2024.1/include 
            /Library/Developer/CommandLineTools/SDKs/MacOSX15.0.sdk/usr/include/ffi
            /opt/homebrew/Cellar/gtkmm4/4.16.0/include/gtkmm-4.0 
            /opt/homebrew/Cellar/gtkmm4/4.16.0/lib/gtkmm-4.0/include 
            /opt/homebrew/Cellar/pangomm/2.54.0/include/pangomm-2.48 
            /opt/homebrew/Cellar/pangomm/2.54.0/lib/pangomm-2.48/include 
            /opt/homebrew/Cellar/glibmm/2.82.0/include/giomm-2.68 
            /opt/homebrew/Cellar/glibmm/2.82.0/lib/giomm-2.68/include 
            /opt/homebrew/Cellar/glib/2.82.2/include/gio-unix-2.0 
            /opt/homebrew/Cellar/glibmm/2.82.0/include/glibmm-2.68 
            /opt/homebrew/Cellar/glibmm/2.82.0/lib/glibmm-2.68/include 
            /opt/homebrew/Cellar/glib/2.82.2/include 
            /opt/homebrew/Cellar/cairomm/1.18.0/include/cairomm-1.16 
            /opt/homebrew/Cellar/cairomm/1.18.0/lib/cairomm-1.16/include 
            /opt/homebrew/Cellar/libsigc++/3.6.0/include/sigc++-3.0 
            /opt/homebrew/Cellar/libsigc++/3.6.0/lib/sigc++-3.0/include 
            /opt/homebrew/Cellar/gtk4/4.16.3/include/gtk-4.0/unix-print
    )

    target_link_directories(transmission::gtk_impl
        INTERFACE
            ${GTK${GTK_VERSION}_LIBRARY_DIRS})

    target_link_libraries(transmission::gtk_impl
        INTERFACE
            ${GTK${GTK_VERSION}_LIBRARIES})
endif()
```

Just make sure, you use the path returned by `pkg-config` commands on your machine. Above, is just example.

Remember that after you install GTK dependencies also the QT and native macOS builds will fail if you do not apply the above fix. You can also just remove `gtk4` and  `gtkmm4`:

```bash
brew uninstall gtk4 gtkmm4 
```

and then use native Xcode project or CMake for macOS.
