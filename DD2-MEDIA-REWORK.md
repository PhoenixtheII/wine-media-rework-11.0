# dwproton-ffmpeg8-dd2

Notes, patches, and a launcher for a custom **DWProton 11.0-15** build using **FFmpeg 8.1.3** and the **GE-Proton 11-6 media rework**, made to fix Dragon's Dogma 2 movie playback on Linux.

## Result

Dragon's Dogma 2 now gets past the Media Foundation movie path and plays the opening cinematic / intro without the previous crash:

```text
Failed to create SourceReader with registered byte stream handler.
error(c00d36bb)
```

The game was tested with:

- DWProton 11.0-15 source
- FFmpeg n8.1.3
- Valve Wine 11.0 at commit `9358696fe9a2261329f4a83aa6a65fd436106154`
- GE-Proton 11-6 `patches/ge-video-rework` media rework
- REFramework 01422 + `intro_skip.lua`
- OnlineFix with `EmulateTicket=true`
- `GAMEID=umu-480`
- NVIDIA RTX 3080 Ti, driver 615.78.08
- 2800x1752 display

## Root cause

The DWProton FFmpeg configuration enabled libvpx:

```text
--enable-libvpx
```

That made `libavcodec.so.62` depend on `libvpx.so.9`. The Proton runtime only provides `libvpx.so.6`, so the media stack could not load its codec library and DD2 failed while creating the SourceReader.

Removing `--enable-libvpx` and rebuilding FFmpeg 8.1.3 fixes it. Native VP8/VP9 decoders remain available; only the external libvpx wrapper is lost.

The final `libavcodec` dependencies are:

```text
libswresample.so.6
libavutil.so.60
libm.so.6
liblzma.so.5
libvorbis.so.0
libz.so.1
libva.so.2
libc.so.6
```

No `libvpx.so.9` dependency remains.

## Contents

- `CHANGELOG.md` — detailed change summary and test history
- `patches/dwproton-ffmpeg-no-libvpx.patch` — removes the libvpx dependency from DWProton's FFmpeg build
- `patches/wine-media-rework.patch` — combined Valve Wine 11.0 + GE-Proton 11-6 media rework patch
- `scripts/patch-python39.py` — compatibility patches for the Steam Runtime Python 3.9
- `scripts/steamit-dd2-custom` — launcher for the custom build
- `NOTICE.md` — upstream attribution and licensing notes

## Rebuilding

The build was made with DWProton's Docker build system. The general sequence is:

```bash
# 1. DWProton source
git clone --recurse-submodules https://dawn.wine/dawn-winery/dwproton.git
cd dwproton
git -C ffmpeg checkout n8.1.3
git apply /path/to/patches/dwproton-ffmpeg-no-libvpx.patch

# 2. Valve Wine 11.0 base used by GE-Proton 11-6
git clone https://github.com/ValveSoftware/wine.git ../valve-wine
git -C ../valve-wine checkout 9358696fe9a2261329f4a83aa6a65fd436106154

# 3. Apply the media rework and use it as DWProton's Wine tree
git -C ../valve-wine apply /path/to/patches/wine-media-rework.patch
rm -rf wine
cp -a ../valve-wine wine

# 4. Build
mkdir build && cd build
../configure.sh \
  --build-name=dwproton-ffmpeg8 \
  --container-engine=docker \
  --enable-ccache \
  --without-tts
make -j"$(nproc)" redist
```

The tested build used `HOST_CFLAGS=-O3 -march=native` from the configure environment. That creates a CPU-specific build. For a portable build, use the configure default:

```bash
CFLAGS="-O2 -march=nocona -mtune=core-avx2" \
RUSTFLAGS="-Copt-level=3 -Ctarget-cpu=nocona" \
../configure.sh ...
```

After extracting the resulting tarball, patch the Python 3.9 incompatibilities in the Proton scripts:

```bash
python3 scripts/patch-python39.py /path/to/dwproton-ffmpeg8
```

Then run with:

```bash
PROTONPATH=/path/to/dwproton-ffmpeg8 \
WINEPREFIX=$HOME/games/dd2-media \
./scripts/steamit-dd2-custom
```

## Notes / limitations

- The Wine tree is a **hybrid**: Valve Wine 11.0 plus GE-Proton 11-6's media rework. It is not the stock DWProton Wine tree.
- The full GE `ge-video-rework` patch series was applied. The unrelated RSX3D shim patch was skipped. Minimal DualSense-haptic helper stubs were added so the audio patches compile.
- This is a local, CPU-specific build. It is not an official DWProton or GE-Proton release.
- No game files, OnlineFix binaries, or other proprietary content are included in this repository.
