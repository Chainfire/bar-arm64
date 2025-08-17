# Beyond all Reason native on Asahi

Tested on MacBook Pro M1 Max with Fedora Asahi Remix 42

Relevant repos:

- RecoilEngine fork: https://github.com/Chainfire/bar-RecoilEngine/tree/asahi

Discussions:

- https://github.com/beyond-all-reason/RecoilEngine/issues/936

@AdamDorwart 's sources, on which most of this is very heavily based:

- https://github.com/AdamDorwart/spring/pull/1

His commits are not included directly as I restructured them completely,
so it should no longer break the non-arm64 builds. The big brain stuff
on the FPU side is all him, though.

## A working game directory

Can't start work without it. I was not able to get the x86_64 Linux image to work directly with muvm and Fex (Electron/Node connectivity issue), but we can just install the Windows version with the Steam port the Asahi team has blessed us with.

Instructions, modified from chmod777's excellent advice on the BAR discord:

- Download the Windows installer
- Steam -> Games -> Add a non-Steam game to my library -> Browse -> select Windows installer
- In your game library right click on the newly added Beyond-All-Reason-*.exe file -> Properties... -> Compatibility -> check "Force the use of a specific Steam Play compatibility tool", select "Proton Experimental", close dialog
- "Play" the installer, go through the setup, but do not start the game at the end of the installer; close the installer
- Find your real install location, open a terminal, and do something like: `find ~/.steam/steam/ | grep -i "/beyond-all-reason.exe"`. The result should look something like `/home/<username>/.steam/steam/steamapps/compatdata/<id>/pfx/drive_c/users/steamuser/AppData/Local/Programs/Beyond-All-Reason/Beyond-All-Reason.exe`
- Go to the game's properties again, and paste the full path to that exe file in the Target box, strip off the trailing Beyond-All-Reason.exe and put the path in the Start In box, close the dialog

Now "Play" the game. This should start the BAR launcher, which will download and install all the game files we need. At this stage, you can actually play the game (in emulated mode) and it works pretty well!

Off-topic: if you're playing this in emulated mode and everything is very large or very tiny, change your scale settings in KDE's Display Configuration. If using multiple screens, be sure they all have the *same* scale!

## RecoilEngine

#### Dependencies

These are just the Fedora equivalents of the packages mentioned here: https://github.com/beyond-all-reason/RecoilEngine/wiki/Building-and-developing-engine-without-docker

```
sudo dnf install -y cmake gcc g++ ccache ninja-build clang clang++ lld git clangd socat

sudo dnf install -y doxygen SDL2-devel DevIL-devel libcurl-devel \
  p7zip p7zip-plugins openal-soft-devel libogg-devel libvorbis-devel libunwind-devel freetype-devel \
  glew-devel minizip-devel fontconfig-devel jsoncpp-devel

# had to add this one
sudo dnf install -y expat-devel
```

#### Clone the code

```
git clone https://github.com/Chainfire/bar-RecoilEngine.git -b asahi --recursive
cd bar-RecoilEngine
```

#### Toolchain setup

```
mkdir toolchain
cat >toolchain/gcc_aarch64-pc-linux-gnu.cmake <<EOF
SET(CMAKE_SYSTEM_NAME Linux)
SET(CMAKE_SYSTEM_PROCESSOR "aarch64")
SET(CMAKE_C_COMPILER "gcc")
SET(CMAKE_CXX_COMPILER "g++")
SET(CMAKE_EXE_LINKER_FLAGS_INIT "-fuse-ld=lld")
SET(CMAKE_MODULE_LINKER_FLAGS_INIT "-fuse-ld=lld")
SET(CMAKE_SHARED_LINKER_FLAGS_INIT "-fuse-ld=lld")
EOF
```

#### Build

```
rm -rf CMakeCache.txt CMakeFiles/
CI=1 cmake \
        -DCMAKE_TOOLCHAIN_FILE="toolchain/gcc_aarch64-pc-linux-gnu.cmake" \
        -DCMAKE_CXX_COMPILER_LAUNCHER=ccache \
        -DCMAKE_CXX_FLAGS_RELWITHDEBINFO="-O3 -g -DNDEBUG -fdiagnostics-color=always" \
        -DCMAKE_C_FLAGS_RELWITHDEBINFO="-O3 -g -DNDEBUG -fdiagnostics-color=always" \
        -DCMAKE_BUILD_TYPE=RELWITHDEBINFO \
        -DAI_TYPES=NATIVE \
        -DINSTALL_PORTABLE=ON \
        -DCMAKE_USE_RELATIVE_PATHS:BOOL=1 \
        -DBINDIR:PATH=./ \
        -DLIBDIR:PATH=./ \
        -DDATADIR:PATH=./ \
        -DCMAKE_INSTALL_PREFIX="$(pwd)/install" \
        -G Ninja
CI=1 ninja install
```

#### Run

The line below that finds the data directory inside Steam may not work for you, check it manually first

```
BAR_DATA=`find ~/.steam/steam/ | grep -i "/beyond-all-reason.exe" | sed 's/\/Beyond-All-Reason\.exe//g'`/data

./install/spring --write-dir $BAR_DATA --isolation --menu rapid://byar-chobby:test
```

## Notes

#### CpuTopology / Task Affinity

I have added code to detect the cpu topology that "should" work on all arm64 boards. It currently selects
all the performance cores to bind with, or all cores if no obvious difference between cores is detected.

I have however noticed that while this works for most threads, some threads appear to not get the task
affinity applied. I don't know if this is also the case for an x86_64 Linux or Windows builds, worth 
investigating.

Additionally, at least on my MacBook (this may be different on Minis and Studios), the game performs a lot 
better and stutters a lot less if after starting I manually taskset the process and all its threads to half
the performance cores (cores #2-#5 in my case) or even just one! This can be somewhat mitigated by setting
the scheduler to "performance" but that also turns the MacBook into a noisy space heater.

Monitoring `lscpu --extended` explains why :)

#### Desync testing

The patches have been tested against tag 2025.04.08 (took some extra work, current set applies to master)
and played this replay ( https://www.beyondallreason.info/replays?gameId=81eba168e6b9a20ad04beb03ed152b3e )
start to finish without a single desync.

## TODO

- byar-chobby, haven't looked at it yet
- FPUCheck, this is just patched out
- obviously there's no multiplayer (requires the correct engine version), and if there was, it could desync, needs ample testing
