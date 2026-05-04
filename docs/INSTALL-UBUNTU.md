# Building HyperHDR-10bit on Ubuntu 22.04 / 24.04

This fork is the [johnneerdael/HyperHDR-10bit](https://github.com/johnneerdael/HyperHDR-10bit) tree, which adds experimental P010 (10-bit) FlatBuffer wire-format support to upstream HyperHDR. These are focused step-by-step instructions for compiling and running it on Ubuntu 22.04 LTS (jammy) or 24.04 LTS (noble). For other platforms or build methods (Docker, Windows, macOS, LibreELEC) see [`BUILDING.md`](../BUILDING.md) at the repo root.

Tested with:

| Ubuntu | gcc | cmake | Qt6 | FlatBuffers |
|---|---|---|---|---|
| 22.04 LTS (jammy) | 11.4 | 3.22 | 6.2.4 | 2.0.6 |
| 24.04 LTS (noble) | 13.2 | 3.28 | 6.4.2 | 23.5.26 |

Both are fine — HyperHDR's CMake requires `>= 3.16` and Qt6, both well below the Ubuntu 22.04 floor.

---

## 1. Install build prerequisites

```bash
sudo apt update
sudo apt install -y \
  build-essential cmake ninja-build pkg-config patchelf chrpath git wget unzip \
  python3 \
  flatbuffers-compiler libflatbuffers-dev libssl-dev liblzma-dev libzstd-dev \
  qt6-base-dev qt6-serialport-dev \
  libasound2-dev libayatana-appindicator3-dev \
  libegl-dev libgl-dev libglvnd-dev libgtk-3-dev libx11-dev \
  libpipewire-0.3-dev libsystemd-dev \
  libftdi1-dev libusb-1.0-0-dev
```

The TurboJPEG dev headers are packaged under different names on the two releases — 22.04 (jammy) ships `libturbojpeg0-dev`, 24.04 (noble) ships `libturbojpeg-dev`. Install whichever your release has:

```bash
sudo apt install -y libturbojpeg0-dev || sudo apt install -y libturbojpeg-dev
```

Optional — Raspberry Pi CEC support:
```bash
sudo apt install -y libcec-dev libp8-platform-dev libudev-dev
```

---

## 2. Clone the fork

```bash
cd ~
git clone --recursive -b p010-wire-format \
  https://github.com/johnneerdael/HyperHDR-10bit.git hyperhdr
cd hyperhdr
```

`--recursive` is required — HyperHDR has several git submodules under `dependencies/external/` that the build needs.

If you already cloned without `--recursive`:
```bash
git submodule update --init --recursive
```

### Confirm the P010 commits are on HEAD

```bash
git log --oneline -n 6 | grep -iE "p010"
```

Expected:
```
5a266c5 docs: manual P010 round-trip test recipe
48d9f16 docs: CHANGELOG entry for P010 flatbuffer variant
82117d9 feat(flatbuf): handle P010 frames in server dispatch
22faa3a feat(flatbuf): parse incoming P010Image frames
d9d6244 feat(flatbuf): add P010 to FLATBUFFERS_IMAGE_FORMAT enum
3ad034d feat(flatbuf): add P010Image variant to ImageType union
```

If those six commits aren't there, you're on the wrong branch. `git checkout p010-wire-format` and try again.

---

## 3. Configure and build

```bash
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
```

Build takes ~5–15 minutes on a modern x86_64 PC, longer on Raspberry Pi.

If `make` is killed mid-build with `Killed` or no output, the build ran out of RAM (common in 1–2 GB VMs). Reduce parallelism:
```bash
make -j2
```

You can also bump VM swap or close other apps. The compiler peaks at ~1.5 GB per parallel job.

---

## 4. Run it

### Option A — Run straight from the build dir (no install)

```bash
./bin/hyperhdr -d
```

Useful for quick iteration. The web UI starts at `http://<host>:8090`. The FlatBuffer server (which is what the Android client talks to) listens on `19400`.

### Option B — Build a `.deb` and install system-wide

```bash
cpack
# you should now see HyperHDR-<version>-Linux-amd64.deb in the build dir
sudo apt install ./HyperHDR-*.deb
```

After install, HyperHDR runs as a systemd user service:
```bash
systemctl --user enable --now hyperhdr
journalctl --user -fu hyperhdr
```

To run as a system service instead, see upstream HyperHDR's documentation.

---

## 5. Verify the P010 path is reachable

The P010 patches add a new `P010Image` variant to the FlatBuffer schema's `ImageType` union and a parser branch in `FlatBuffersServer.cpp`. The path is reachable as soon as a client sends a `P010Image`-typed frame — there is no separate enable flag.

To verify end-to-end with the included synthetic test client, follow [`docs/manual-tests/01-flatbuffer-p010-roundtrip.md`](manual-tests/01-flatbuffer-p010-roundtrip.md). On the first P010 frame, HyperHDR's debug log emits a one-shot line:

```
[FlatBufferServer] Debug: Received first P010 frame. First plane size: <N> (stride: <S>). Second plane size: <M> (stride: <S>). Image size: <T> (<W> x <H>)
```

If you see that line: parser, dispatch, and image-decode paths all work. If you instead see:

```
[FlatBufferServer] Error: Unsupported flatbuffers image format
```

then the parser branch isn't matching — most likely your build is from upstream `awawa-dev/HyperHDR` and not this fork. Double-check `git log` for the six P010 commits above.

To exercise the path against a real Android device, build and install [HyperHDR-android v1.1.0+](https://github.com/johnneerdael/HyperHDR-android), enable Settings → HDR → "10-bit HDR transport (experimental)", and start capture pointing at this server.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `cmake: command not found` | apt list above incomplete | `sudo apt install cmake` |
| `Could NOT find Qt6` | Qt6 dev packages missing | `sudo apt install qt6-base-dev qt6-serialport-dev` |
| `flatc: command not found` during build | flatbuffers-compiler missing | `sudo apt install flatbuffers-compiler libflatbuffers-dev` |
| `make` killed with no error | Out-of-memory | Use `make -j2` or add swap |
| Build succeeds but server says "Unsupported flatbuffers image format" on P010 frame | Built from upstream master, not the fork | Re-clone with `-b p010-wire-format` |
| `libpipewire-0.3-dev` not found on a server-only headless install | No Pipewire on the target | Pass `-DENABLE_PIPEWIRE=OFF` to cmake |
| `qt6-base-dev` not found on Ubuntu < 22.04 | Pre-jammy Ubuntu only ships Qt5 | Use Ubuntu 22.04 or newer; this fork doesn't support Qt5 |

---

## Optional CMake flags

For a slimmer headless server build (no GUI grabbers, no audio capture):

```bash
cmake -DCMAKE_BUILD_TYPE=Release \
      -DENABLE_X11=OFF \
      -DENABLE_PIPEWIRE=OFF \
      -DENABLE_FRAMEBUFFER=OFF \
      -DENABLE_SOUNDCAPLINUX=OFF \
      -DENABLE_SYSTRAY=OFF \
      ..
```

The full list of build options is in [`BUILDING.md`](../BUILDING.md) under "Available CMake HyperHDR configuration options".
