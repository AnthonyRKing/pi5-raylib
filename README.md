# pi5-raylib
### My experience running Raylib's C code examples on Raspberry Pi 5 in desktop and native (DRM) modes, using OpenGL ES 3.1, and more

See [this post on Odin's forum](https://forum.odin-lang.org/t/problems-with-raylib-linux-raspberry-pi-5/1386/23?u=anthonyrking), where I initially discussed this.

I have managed to create a GUI Raylib window with a non-C language (Ada) on Raspberry Pi 5 Bookworm (Wayland), more of which later.

## Running Raylib Examples
Here I'm addressing compilation and running of Raylib C code examples in Raylib's `examples` directory.

The [Raylib Pi instructions](https://github.com/raysan5/raylib/wiki/Working-on-Raspberry-Pi) page explains how to get started, including which dependencies are required etc. A full clone of the repository, which can be done in your project directory comes in at almost 800MB, so I'd recommend a shallow clone if you don't want that size of a download:

```
git clone --depth 1 https://github.com/raysan5/raylib.git
```
This is a more modest 50MB~.

## Building Raylib
You have to make Raylib with the required options first, from the `raylib/src` directory:
```
make clean
```
Then depending whether you want to compile for desktop window or native full-screen,  either:
```
make PLATFORM=PLATFORM_DESKTOP GRAPHICS=GRAPHICS_API_OPENGL_ES3
```
Or:
```
make PLATFORM=PLATFORM_DRM GRAPHICS=GRAPHICS_API_OPENGL_ES3
```
This produces a `libraylib.a` file, which gets linked as a library in subsequent builds of the C code examples (or anything else). The `make clean` command is not required for the first run, but is required for any subsequent builds, which you might want to do. It does not take too long to do this.

As well as building for a windowed version, I also successfully ran examples in direct mode (aka native mode, aka full-screen) using the alternative `PLATFORM_DRM` option as above. You can initiate a virtual terminal as part of the run command from a terminal within the desktop environment (see the build/run script described below) if you want to try that way.

Though Raysan5's instruction page refers only to X11, I did not have to switch my desktop from Wayland to X11 to get the examples to work. I could freely switch between running the same example in windowed and full-screen modes, from the desktop, as long as both Raylib and the example were built/re-built with a matching `PLATFORM` option.

## Compiling Examples
In case you come across error output indicating undefined reference to GL functions such as `glActiveTexture` when compiling, this is something I would see if I tried compiling a C example with mis-matching `PLATFORM` settings in the respective builds - i.e. the example build not matching your build of `libraylib.a`.

The [Raylib Pi instructions](https://github.com/raysan5/raylib/wiki/Working-on-Raspberry-Pi) seem a bit out of date at time of writing. They speak of supporting version 2 variants of OpenGL on Bullseye, but you can actually work with version 3. The C example files I've seen seem to select between versions 3.30 and 1.00:
```
#if defined(PLATFORM_DESKTOP)
    #define GLSL_VERSION            330
#else   // PLATFORM_ANDROID, PLATFORM_WEB
    #define GLSL_VERSION            100
#endif
```
We don't have v3.3, but we can work with Pi 5's GL ES 3.0 capability, in any case. Actually the Pi 5 has GL ES 3.1 capability, but I think I'm just working with 3.0 so far. This `GLSL_VERSION` value is later used in the particular example code I analysed (`shaders_basic_lighting.c`), to select a directory for (shader) resources:
```
// Load basic lighting shader
    Shader shader = LoadShader(TextFormat("resources/shaders/glsl%i/lighting.vs", GLSL_VERSION),
                               TextFormat("resources/shaders/glsl%i/lighting.fs", GLSL_VERSION));
```
The trick for using Pi 5's GL 3.1 ES with these examples is to actually edit those resource files and set the version these resources are going to interact with, which I found in the 1st line of each file. To make compatible, we also need to specify a precision limitation (in line 2). If not done, the example may work partially, but without the shader functioning:
```
#version 310 es  // instead of 330!
precision mediump float;  // to compensate
```
Keep the setting to the '330' version in the C code though, so the directory with the resources can be found - unless you decide to create a `resources/shaders/glsl310es` directory for the modified shaders, for example. This worked out OK for `shaders_basic_lighting.c` for me. Also from the C file:
```
*   NOTE: This example requires raylib OpenGL 3.3 or ES2 versions for shaders support,
*         OpenGL 1.1 does not support shaders, recompile raylib to OpenGL 3.3 version
*
*   NOTE: Shaders used in this example are #version 330 (OpenGL 3.3)
```
There's no mention of version 2, and there are no v2 shader directories.

I found the example build process to be a bit awkward, requiring to delete the previous output, `cd ..` to make the example after editing in the example's directory, and to specify the `PLATFORM` and `GRAPHICS` parameters. I made a script for it, which also optionally runs the output according to the build type. Note that a `debug.txt` file is created in the examples directory you're in, for virtual terminal launches.

To use this script, you need to place it into the specific example directory you're working in (e.g. `raylib/examples/shaders`):

`brun.sh:`
```
#!/bin/bash

#
# ARK's Pi 5 Raylib Examples C Code Build & Run Script
#
# Place this script in the directory where the example code you want to try is.
#
# Specify platform type (must match that used for Raylib library build) and GL version here:
#
PLATFORM=PLATFORM_DESKTOP         # PLATFORM_DRM or PLATFORM_DESKTOP
GRAPHICS=GRAPHICS_API_OPENGL_ES3  # Requires examples to use resources for v3.30
RUN=true

# Get the current directory name (e.g., "shaders", "models", "core")
PARENT_DIR=$(basename "$PWD")
APP_NAME=$1  # e.g. shaders_basic_lighting
APP_NAME="${APP_NAME%.c}"  # Strip the .c extension if it exists using parameter expansion (e.g., "demo.c" -> "demo")
# Ensure we actually have a name
if [ -z "$APP_NAME" ]; then
  echo "Usage: ./brun.sh <app_name>"
  exit 1
fi
# Ensure the .c file exists
if [ ! -f "$APP_NAME.c" ]; then
  echo "$APP_NAME.c does not exist"
  exit 1
fi

# vcgencmd measure_temp  # VideoCore GENeral CoMmanD for displaying core temp

# Rebuild the app
echo "Removing output file: $APP_NAME"
rm -f "$APP_NAME"
cd ..
make_string='make PLATFORM='$PLATFORM' GRAPHICS='$GRAPHICS' '$PARENT_DIR'/'$APP_NAME
echo -e "\nExecuting: $make_string"
eval $make_string  # alternative to sh -c "$make_string"
# Check the build succeeded
if [ $? -ne 0 ]; then
  echo "Make failed for ${PLATFORM} mode build"
  exit 1
fi

# Return to original directory and run the app if wanted
cd "$PARENT_DIR"
if [ "$RUN" = true ]; then
  echo ''
  if [ "$PLATFORM" = "PLATFORM_DRM" ]; then
    echo "Running "$APP_NAME" in virtual terminal (full screen)..."
    sudo openvt -s -w -- sh -c "./$APP_NAME > debug.txt 2>&1"
    cat debug.txt  # So you can see the OPENGL output after exiting DRM (full screen) mode
  else
    echo "Running "$APP_NAME" in window..."
    ./$APP_NAME  # Assume we can otherwise run in the desktop GUI
  fi
fi

# vcgencmd measure_temp  # See how hot we go
```
Remember to `chmod +x brun.sh` before executing it from the example code's directory like in this example:
```
./brun.sh shaders_basic_lighting
```
![screenshot-20260712-182728 - Example Raylib PLATFORM_DESKTOP OpenGL ES3 on Raspberry Pi 5 Bookworm Output|690x330](upload://xt8uwJgUEjQYksdKGbEkz9aUilY.png)

Esc to exit the demo.

## Linking With Other Languages
When linking with my Ada program, it was a case of providing Raylib's source directory, specifying `libraylib.a` and linking other library features in an `-largs` option, just as with linking compiled C code:
```
gnatmake -f hello.adb -largs -L ./raylib/src -l:libraylib.a -lGL -lm -lpthread -ldl -lrt -lX11 -lXrandr
```
You may want to seek out some bindings for the particular language you want to use Raylib with. With Ada, I could manually create enough bindings to load and close a window, but that's as far as I got with it as yet.
