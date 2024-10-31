---
tags:
  - bittorrent
link: https://transmissionbt.com
source: https://github.com/transmission/transmission
related-to:
---

#bittorrent 

| link       | https://transmissionbt.com                 |
| ---------- | ------------------------------------------- |
| source     | https://github.com/transmission/transmission |
| related-to |                     |

[[Learn BitTorrent]] client. One of the three popular clients, the other are [[Deluge (BitTorrent)]] and [[qBittorrent]], with the most attractive looking website. Comparing to the other two, Transmission does not depend on [[libtorrent-rasterbar]], but rather has its own implementation of the BitTorrent protocol.

> As noted in GitHub’s REDME, [Transmission's documentation](https://github.com/transmission/transmission/blob/main/docs/README.md) is currently out-of-date, but the team has recently begun a new project to update it and is looking for volunteers. If you're interested, please feel free to submit pull requests!

It supports all major OS and has a native macos support.

The latest release is from 30 May 2024.

### How does it look like?

TBD...

### Building

Some loose notes for now... TBD..

```bash
CMake Error in gtk/CMakeLists.txt:
  Imported target "transmission::gtk_impl" includes non-existent path

    "/Library/Developer/CommandLineTools/SDKs/MacOSX13.sdk/usr/include/ffi"

  in its INTERFACE_INCLUDE_DIRECTORIES.  Possible reasons include:

  * The path was deleted, renamed, or moved to another location.

  * An install or uninstall procedure did not complete successfully.

  * The installation package was faulty and references files it does not
  provide.



CMake Error in gtk/CMakeLists.txt:
  Imported target "transmission::gtk_impl" includes non-existent path

    "/Library/Developer/CommandLineTools/SDKs/MacOSX13.sdk/usr/include/ffi"

  in its INTERFACE_INCLUDE_DIRECTORIES.  Possible reasons include:

  * The path was deleted, renamed, or moved to another location.

  * An install or uninstall procedure did not complete successfully.

  * The installation package was faulty and references files it does not
  provide.



-- Generating done (0.2s)
CMake Generate step failed.  Build files cannot be regenerated correctly.
~/code/open-source/bittorrent/transmission-mac-cmake(main)
» brew uninstall gtk4 gtkmm4                                                                                        1 ↵
Uninstalling /opt/homebrew/Cellar/gtk4/4.16.3... (598 files, 57.8MB)
Uninstalling /opt/homebrew/Cellar/gtkmm4/4.16.0... (700 files, 9.7MB)
==> Autoremoving 8 unneeded formulae:
cairomm
gdk-pixbuf
glibmm
graphene
hicolor-icon-theme
libepoxy
libsigc++
pangomm
Uninstalling /opt/homebrew/Cellar/gdk-pixbuf/2.42.12... (152 files, 4.0MB)
Uninstalling /opt/homebrew/Cellar/pangomm/2.54.0... (70 files, 755.8KB)
Uninstalling /opt/homebrew/Cellar/graphene/1.10.8... (37 files, 1MB)
Uninstalling /opt/homebrew/Cellar/hicolor-icon-theme/0.18... (8 files, 80.8KB)
Uninstalling /opt/homebrew/Cellar/libepoxy/1.5.10... (11 files, 2.6MB)
Uninstalling /opt/homebrew/Cellar/glibmm/2.82.0... (449 files, 6.4MB)
Uninstalling /opt/homebrew/Cellar/cairomm/1.18.0... (43 files, 592.4KB)
Uninstalling /opt/homebrew/Cellar/libsigc++/3.6.0... (52 files, 378.3KB)
~/code/open-source/bittorrent/transmission-mac-cmake(main)
» cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=RelWithDebInfo
-- Looking for clang-tidy
-- Looking for clang-tidy - not found
-- Configuring done (0.4s)
CMake Error in gtk/CMakeLists.txt:
  Imported target "transmission::gtk_impl" includes non-existent path

    "/opt/homebrew/Cellar/gtkmm4/4.16.0/include/gtkmm-4.0"

  in its INTERFACE_INCLUDE_DIRECTORIES.  Possible reasons include:

  * The path was deleted, renamed, or moved to another location.

  * An install or uninstall procedure did not complete successfully.

  * The installation package was faulty and references files it does not
  provide.



CMake Error in gtk/CMakeLists.txt:
  Imported target "transmission::gtk_impl" includes non-existent path

    "/opt/homebrew/Cellar/gtkmm4/4.16.0/include/gtkmm-4.0"

  in its INTERFACE_INCLUDE_DIRECTORIES.  Possible reasons include:

  * The path was deleted, renamed, or moved to another location.

  * An install or uninstall procedure did not complete successfully.

  * The installation package was faulty and references files it does not
  provide.



-- Generating done (0.2s)
CMake Generate step failed.  Build files cannot be regenerated correctly.
~/code/open-source/bittorrent/transmission-mac-cmake(main)
» ninja clean                                                                                                       1 ↵
ninja: error: loading 'build.ninja': No such file or directory
~/code/open-source/bittorrent/transmission-mac-cmake(main)
» rm -rf CMakeCache.txt CMakeFiles/                                                                                 1 ↵
~/code/open-source/bittorrent/transmission-mac-cmake(main)
» cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=RelWithDebInfo
-- Looking for clang-tidy
-- Looking for clang-tidy - not found
-- Configuring done (0.5s)
CMake Error in gtk/CMakeLists.txt:
  Imported target "transmission::gtk_impl" includes non-existent path

    "/opt/homebrew/Cellar/gtkmm4/4.16.0/include/gtkmm-4.0"

  in its INTERFACE_INCLUDE_DIRECTORIES.  Possible reasons include:

  * The path was deleted, renamed, or moved to another location.

  * An install or uninstall procedure did not complete successfully.

  * The installation package was faulty and references files it does not
  provide.



CMake Error in gtk/CMakeLists.txt:
  Imported target "transmission::gtk_impl" includes non-existent path

    "/opt/homebrew/Cellar/gtkmm4/4.16.0/include/gtkmm-4.0"

  in its INTERFACE_INCLUDE_DIRECTORIES.  Possible reasons include:

  * The path was deleted, renamed, or moved to another location.

  * An install or uninstall procedure did not complete successfully.

  * The installation package was faulty and references files it does not
  provide.



-- Generating done (0.2s)
CMake Generate step failed.  Build files cannot be regenerated correctly.
```

https://doc.qt.io/qt-6/macos.html#supported-versions
https://github.com/fontforge/fontforge/discussions/5164
https://github.com/fontforge/fontforge/issues/5153

https://www.reddit.com/r/MacOS/comments/1fs551h/macos_15_sequoia_update_caused_include_errors_in_c/

```bash
xcrun -sdk macosx --show-sdk-path
pkg-config --cflags gtkmm-4.0
pkg-config --cflags gtk4
```

`CMakeLists.txt` around line 361:

```cmake
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
            /opt/homebrew/Cellar/gtk4/4.16.3/include/gtk-4.0/unix-print)
            # ${GTK${GTK_VERSION}_INCLUDE_DIRS})

    target_link_directories(transmission::gtk_impl
        INTERFACE
            ${GTK${GTK_VERSION}_LIBRARY_DIRS})

    target_link_libraries(transmission::gtk_impl
        INTERFACE
            ${GTK${GTK_VERSION}_LIBRARIES})
endif()
```