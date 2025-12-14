# Introduction
![image](images/miyoo-mini.jpg) ![image](images/miyoo-mini-plus.jpg)

&nbsp;

This repository contains the SDL v2.0 source code, ported for the following handheld devices:  
- Miyoo Mini (Plus)  
  Utilizes SigmaStar MI GFX for rendering and is supported on both Stock OS and Onion v4.3.1-1.

&nbsp;

# Miyoo Mini Plus SDL2 (steward-fu) — Build + Deploy Notes

This README captures a working workflow to:

- Reset Ubuntu in WSL
- Install build dependencies
- Download/install the Miyoo Mini toolchain
- Build `steward-fu/sdl2` (cfg/gpu/sdl2)
- Cross-build SDL2 addon dependencies (SDL2_* + json-c)
- Stage runtime `.so` files for the target
- Deploy examples to the Miyoo Mini Plus via `rsync`
- Run examples over SSH
- Work around MI_AO (SigmaStar) audio ownership issues for audio examples

---

## PowerShell (Windows / WSL)

```powershell
# Ensure WSL itself is installed and up to date:
# - Installs WSL if missing;
# - Enables required Windows features;
# - Updates the WSL kernel if already installed.
wsl --install

# Remove the existing Ubuntu distribution (hard reset)
# WARNING: This permanently deletes all files and user data in the Ubuntu environment.
wsl --unregister Ubuntu

# Install the current Ubuntu LTS release
# NOTE
# - Passing in "Ubuntu" always points to the latest LTS available
# - As of today, this resolves to Ubuntu 24.04.3 LTS
wsl --install -d Ubuntu

# Explicitly set Ubuntu as the default WSL distribution
# (useful when multiple distros are installed)
wsl --set-default Ubuntu

# List all installed WSL distributions and confirm default
wsl --list --verbose

# Start the WSL distribution and open a shell in '~'
wsl -d Ubuntu --cd ~
```

---

## Bash (Ubuntu in WSL)

### System packages

```bash
sudo apt update -y && sudo apt upgrade -y

# Install core build toolchain and development utilities, including:
# - build-essential : gcc, g++, make, libc headers
# - cmake / ninja   : modern build systems
# - autotools       : autoconf, automake, libtool
# - pkg-config      : dependency discovery
# - gettext, m4     : internationalization & macro processing
# - perl, python3   : required by many build systems
# - unzip, patch    : source archive handling and patching
sudo apt install -y \
  build-essential \
  cmake \
  ninja-build \
  autoconf \
  automake \
  libtool \
  pkg-config \
  gettext \
  m4 \
  perl \
  python3 \
  unzip \
  patch

# Clean up cached packages and remove unused dependencies
sudo apt autoclean -y && sudo apt autoremove -y
```

### Miyoo Mini toolchain

```bash
# Download the Miyoo Mini toolchain (around 700 MB)
URL="https://github.com/steward-fu/website/releases/download/miyoo-mini/mini_toolchain-v1.0.tar.gz"
FILE="mini_toolchain-v1.0.tar.gz"

wget -O "$FILE" "$URL"

# Extract it into a TEMPDIR folder
TEMPDIR="$(mktemp -d)"
tar -xvf "$FILE" -C "$TEMPDIR"

# Move the extracted folders into /opt
sudo mv "$TEMPDIR/mini"     /opt/
sudo mv "$TEMPDIR/prebuilt" /opt/

# Cleanup the TEMPDIR folder
rm -rf "$TEMPDIR"

# Verify
ls -1 /opt | grep -E '^(mini|prebuilt)$'
# - Expecting:
# mini
# prebuilt
```

### Clone + build steward-fu/sdl2

```bash
# Create a folder for 'steward-fu' and clone the SDL2 repo into it (around 270 MB)
mkdir -p steward-fu
cd steward-fu/
git clone --depth 1 https://github.com/steward-fu/sdl2
cd sdl2/

# Build steps provided by the SDL2 repo:
make -j"$(nproc)" cfg CFLAGS="-Wno-psabi" CXXFLAGS="-Wno-psabi"
# - Expecting:
# -- Build files have been written to: /home/luspin/steward-fu/sdl2/swiftshader/build

make -j"$(nproc)" gpu CFLAGS="-Wno-psabi" CXXFLAGS="-Wno-psabi"
# on my machine, this takes around 20 minutes
ls swiftshader/build/lib*
# - Expecting:
# swiftshader/build/libEGL.so  swiftshader/build/libGLESv2.so

make -j"$(nproc)" sdl2 CFLAGS="-Wno-psabi" CXXFLAGS="-Wno-psabi"
# on my machine, this takes around 2 minutes
ls sdl2/build/.libs/libSDL2-2.0.so.0*
# - Expecting:
# sdl2/build/.libs/libSDL2-2.0.so.0  sdl2/build/.libs/libSDL2-2.0.so.0.18.2
```

---

## Cross-build SDL2 dependency libraries

```bash
# Build the provided SDL2 dependencies
cd ~/steward-fu/sdl2/sdl2/dependency

for archive in *.tar.gz; do
  echo "Extracting: $archive"
  tar -xvf "$archive"
done

# Compilation Environment Variables
export TOOLCHAIN=/opt/mini/bin
export PATH="$TOOLCHAIN:$PATH"

# Target triplet for the Miyoo Mini toolchain
export TARGET=arm-linux-gnueabihf

# Where all cross-built dependencies will be placed
export PREFIX="$HOME/steward-fu/sdl2/deps-out"
mkdir -p "$PREFIX"

# Toolchain binaries
export CC="${TARGET}-gcc"
export CXX="${TARGET}-g++"
export AR="${TARGET}-ar"
export RANLIB="${TARGET}-ranlib"
export STRIP="${TARGET}-strip"

# Confirm Miyoo Mini toolchain is being used
command -v "$CC" >/dev/null || { echo "ERROR: $CC not found on PATH"; exit 1; }
"$CC" --version | head -n 1

# 'pkg-config' paths for dependency discovery
export PKG_CONFIG=pkg-config
export PKG_CONFIG_PATH="$PREFIX/lib/pkgconfig:$PREFIX/share/pkgconfig"

# Common include/lib search paths
export CPPFLAGS="-I$PREFIX/include"
export LDFLAGS="-L$PREFIX/lib"
```

### Build SDL2_* addons (Autotools)

```bash
# Build SDL2_* dependencies (leveraging Autotools)
for dir in \
  SDL2_net-2.2.0 \
  SDL2_gfx-1.0.4 \
  SDL2_image-2.8.1 \
  SDL2_mixer-2.6.3 \
  SDL2_ttf-2.20.2
do
  echo "=== Building $dir ==="

  cd "$dir"
  make distclean >/dev/null 2>&1 || true

  # Configure Arguments
  cfg_args=(--host="$TARGET" --prefix="$PREFIX")

  # Specifically for SDL2_gfx: avoid x86/MMX checks when cross-compiling
  if [[ "$dir" == SDL2_gfx-* ]]; then
    cfg_args+=(--disable-mmx)
  fi

  ./configure "${cfg_args[@]}"
  make -j"$(nproc)" V=0
  make install
  cd ..
done
```

### Build json-c (CMake)

```bash
# A different approach is required for the JSON-C dependency
cd ~/steward-fu/sdl2/sdl2/dependency/json-c-0.15

rm -rf build-arm

cmake -S . -B build-arm \
  -DCMAKE_SYSTEM_NAME=Linux \
  -DCMAKE_C_COMPILER="$CC" \
  -DCMAKE_AR="$AR" \
  -DCMAKE_RANLIB="$RANLIB" \
  -DCMAKE_STRIP="$STRIP" \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX="$PREFIX" \
  -DBUILD_SHARED_LIBS=ON \
  -DBUILD_STATIC_LIBS=OFF

cmake --build build-arm -j"$(nproc)"
cmake --install build-arm

# Take note of where .so libraries are installed
find "$PREFIX/lib" -maxdepth 1 -type f -name '*.so*' -o -name '*.a' | sort
# - Expecting:
# /home/luspin/steward-fu/sdl2/deps-out/lib/libSDL2_gfx-1.0.so.0.0.2
# /home/luspin/steward-fu/sdl2/deps-out/lib/libSDL2_gfx.a
# /home/luspin/steward-fu/sdl2/deps-out/lib/libSDL2_image-2.0.so.0.800.1
# /home/luspin/steward-fu/sdl2/deps-out/lib/libSDL2_image.a
# /home/luspin/steward-fu/sdl2/deps-out/lib/libSDL2_mixer-2.0.so.0.600.3
# /home/luspin/steward-fu/sdl2/deps-out/lib/libSDL2_mixer.a
# /home/luspin/steward-fu/sdl2/deps-out/lib/libSDL2_net-2.0.so.0.200.0
# /home/luspin/steward-fu/sdl2/deps-out/lib/libSDL2_net.a
# /home/luspin/steward-fu/sdl2/deps-out/lib/libSDL2_ttf-2.0.so.0.2000.2
# /home/luspin/steward-fu/sdl2/deps-out/lib/libSDL2_ttf.a
# /home/luspin/steward-fu/sdl2/deps-out/lib/libjson-c.so.5.1.0
```

---

## Build + stage SDL2 examples for the Miyoo

```bash
# Build the repository's SDL2 examples
cd ~/steward-fu/sdl2/examples

# If the folder already contains libs from a previous run, archive them - timestamped so reruns don't clobber your backup
STAMP="$(date +%Y%m%d-%H%M%S)"
mkdir -p "prebuiltLibs/$STAMP"

# Move any existing shared libs into the timestamped 'prebuiltLibs' subfolder
# NOTE: shopt is bash-only; Ubuntu's default shell is typically bash, so this is OK here.
shopt -s nullglob
mv -f lib*.so* "prebuiltLibs/$STAMP/" || true
shopt -u nullglob

# Create a dedicated runtime 'libs' directory
mkdir -p libs

# Bring back a couple of prebuilt libraries that aren't built anywhere else yet
cp -av prebuiltLibs/$STAMP/libpng16.so.16 ./libs/
cp -av prebuiltLibs/$STAMP/libz.so.1      ./libs/

# Copy the newly built SDL2 and related libraries to it
# 1) Core SDL2 library
cp -av ~/steward-fu/sdl2/sdl2/build/.libs/libSDL2-2.0.so.0* ./libs/

# 2) SwiftShader runtime
cp -av ~/steward-fu/sdl2/swiftshader/build/libEGL.so    ./libs/
cp -av ~/steward-fu/sdl2/swiftshader/build/libGLESv2.so ./libs/
# Provide the SONAME expected by some loaders/apps (symlink locally)
ln -sf libGLESv2.so ./libs/libGLESv2.so.2

# 3) SDL2 dependency libraries
cp -av ~/steward-fu/sdl2/deps-out/lib/libjson-c.so.5*            ./libs/
cp -av ~/steward-fu/sdl2/deps-out/lib/libSDL2_gfx-1.0.so.0*      ./libs/
cp -av ~/steward-fu/sdl2/deps-out/lib/libSDL2_net-2.0.so.0*      ./libs/
cp -av ~/steward-fu/sdl2/deps-out/lib/libSDL2_image-2.0.so.0*    ./libs/
cp -av ~/steward-fu/sdl2/deps-out/lib/libSDL2_ttf-2.0.so.0*      ./libs/
cp -av ~/steward-fu/sdl2/deps-out/lib/libSDL2_mixer-2.0.so.0*    ./libs/

# Clean, then compile all example files in parallel
make clean
make -j"$(nproc)"
```

### Deploy to device via rsync

```bash
# Copy all files to a directory on the target device using 'rsync' over SSH
# NOTE:
# - The Miyoo filesystem may not permit symlinks.
# - --copy-links copies symlink targets as real files.
# - --no-links avoids attempting to create symlinks on the device.
rsync -rvz --progress --delete \
  --copy-links --no-links \
  --no-owner --no-group \
  -e "ssh -o MACs=hmac-sha1" \
  --rsync-path=/mnt/SDCARD/.tmp_update/bin/rsync \
  . onion@192.168.1.233:/mnt/SDCARD/App/SDL/
```

---

## SSH

```bash
# Connect to the Miyoo Mini Plus via SSH and land directly in the app folder
ssh -t -o MACs=hmac-sha1 onion@192.168.1.233 'cd /mnt/SDCARD/App/SDL && exec sh'

# Run an application while pausing 'MainUI'
# NOTE: if you background the app (&), MainUI will resume immediately.
kill -STOP `pidof MainUI`
LD_LIBRARY_PATH=./libs:/config/lib:/customer/lib ./hello &
kill -CONT `pidof MainUI`
```

---

## Audio examples (MI_AO / SigmaStar)

### Symptom

Most examples run, but audio playback ones fail with MI_AO errors like:

- `MI_AO_SetPubAttr ... failed ... Dev0 busy`
- `Audio device hasn't been opened`
- `Invalid audio device ID`

### Summary / why it happens

Per the MI AO lifecycle requirements (SigmaStar MI AO API docs), the ordering is roughly:

1. `MI_SYS_Init()`
2. `MI_AO_SetPubAttr(AoDevId, &stAttr)`
3. `MI_AO_GetPubAttr(AoDevId, &stAttr)` (optional verification)
4. `MI_AO_Enable(AoDevId)`
5. `MI_AO_EnableChn(AoDevId, AoChn)`
6. `MI_AO_SendFrame(...)`
7. …
8. `MI_AO_DisableChn`
9. `MI_AO_Disable`
10. `MI_SYS_Exit()`

If any process has already enabled AO (e.g. `MainUI`, `audioserver`), then:

- `MI_AO_SetPubAttr()` fails with Dev0 busy
- Even killing later can leave AO partially initialized

Conclusion: the launcher must ensure **exclusive AO ownership for the full init window**, not just “before exec”.

---

## MI_AO launcher: `mi_ao_launch.sh`

Save this to `/mnt/SDCARD/App/SDL/mi_ao_launch.sh` on the device.

```sh
#!/bin/sh
# mi_ao_launch.sh
#
# On this firmware, MI_AO (audio output) is effectively single-owner.
# 'MainUI' or 'audioserver' often keep Dev0 open, so programs that call MI_AO_SetPubAttr() fail
# unless those owners are stopped/killed first.
#
# NOTE:
# - The audio "pop" is usually a hardware/amp re-init transient; muting helps but may not eliminate it.
# - This script logs program output to debug_<prog>.log in the same folder.

APPDIR="$(cd "$(dirname "$0")" && pwd)"
PROG="$1" # first argument

# Basic Argument Validation
[ -z "$PROG" ] && { echo "Usage: $0 <program> [args...]"; exit 2; }
[ ! -x "$APPDIR/$PROG" ] && { echo "ERROR: '$APPDIR/$PROG' not found or not executable"; exit 2; }

# Runtime Library path preferences:
# - Prefer app-local ./libs directory first.
# - Then, the app directory itself.
# - Lastly, firmware libraries.
export LD_LIBRARY_PATH="$APPDIR/libs:$APPDIR:/config/lib:/customer/lib"

# Per-program log file (stdout + stderr)
LOG="$APPDIR/debug_${PROG}.log"

# runtime.sh often supervises MainUI/audioserver; pausing it prevents immediate respawns
RUNTIME_PID="$(pidof runtime.sh 2>/dev/null)"

#######################
# MI_AO /proc Helpers #
#######################
# Write a command to the MI_AO /proc interface if it exists.
# This is a best-effort tweak: failures are tolerated.
ao_cmd() {
  [ -e /proc/mi_modules/mi_ao/mi_ao0 ] || return 1
  echo "$1" > /proc/mi_modules/mi_ao/mi_ao0
}

# Hard Mute right now (best-effort)
ao_mute_now() {
  ao_cmd "set_ao_mute 1" 2>/dev/null || true
}

# Apply a "quiet" profile during transitions:
# 1. Mute
# 2. Drop volume on both channels to reduce transient energy ("pop") during re-init
ao_quiet_profile() {
  ao_cmd "set_ao_mute 1" 2>/dev/null || true
  ao_cmd "set_ao_volume 0 -30dB" 2>/dev/null || true
  ao_cmd "set_ao_volume 1 -30dB" 2>/dev/null || true
}

###########
# Cleanup #
###########
cleanup() {
  # Prevent audio pop on return-to-UI:
  # Mute before letting runtime/MainUI re-init audio
  ao_mute_now
  sleep 0.1

  # Resume runtime.sh supervisor (may restart MainUI/audioserver)
  [ -n "$RUNTIME_PID" ] && kill -CONT "$RUNTIME_PID" 2>/dev/null

  # Unmute after the UI likely finished bringing audio back.
  ( sleep 0.8; ao_cmd "set_ao_mute 0" 2>/dev/null || true ) &
}

trap cleanup EXIT INT TERM

###########################
# Take ownership of MI_AO #
###########################
# Pause runtime to avoid respawn races while we kill/reconfigure MI_AO owners
[ -n "$RUNTIME_PID" ] && kill -STOP "$RUNTIME_PID" 2>/dev/null

# Pre-emptive mute to reduce audio pop while tearing down MI_AO owners
ao_mute_now

# Stop Audio Services
if [ -f /mnt/SDCARD/.tmp_update/script/stop_audioserver.sh ]; then
  /mnt/SDCARD/.tmp_update/script/stop_audioserver.sh
else
  killall audioserver 2>/dev/null
  killall audioserver.mod 2>/dev/null
fi

# MainUI might also own MI_AO; killing it frees Dev0 reliably on this firmware
killall -9 MainUI 2>/dev/null
sleep 0.5

# Once MI_AO proc exists, apply quiet profile while our program initializes audio.
# We wait briefly because the proc node may appear slightly later than process teardown.
start_time=$(date +%s)

while [ ! -e /proc/mi_modules/mi_ao/mi_ao0 ]; do
  sleep 0.1
  elapsed=$(( $(date +%s) - start_time ))
  [ "$elapsed" -ge 3 ] && break
done

ao_quiet_profile

# Unmute after things settle - adjust the 0.6 seconds delay if needed
# NOTE:
# This is best-effort; some pops are hardware/amp re-init related.
( sleep 0.6; ao_cmd "set_ao_mute 0" 2>/dev/null || true ) &

###############
# Run Program #
###############
# Remove the program name from args so we don't accidentally pass "./music music"
shift

cd "$APPDIR" || exit 1

# Log launcher header + capture program output (stdout/stderr)
{
  echo "=== $(date) ==="
  echo "Running: ./$PROG $*"
  "./$PROG" "$@"
  echo "Exit: $?"
} >>"$LOG" 2>&1

exit $?
```

### Fix CRLF line endings (if required)

If you edited the script on Windows, ensure Unix line endings:

```sh
sed -i 's/\r$//' /mnt/SDCARD/App/SDL/mi_ao_launch.sh
```

### Make executable

```sh
chmod +x /mnt/SDCARD/App/SDL/mi_ao_launch.sh
```

### Usage

```sh
./mi_ao_launch.sh music &
```
