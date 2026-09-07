# osmo-trx for macOS ARM64

[osmo-trx](https://osmocom.org/projects/osmo-trx) port to macOS ARM64
(Apple Silicon), Darwin 26+. Osmocom GSM transceiver: the software PHY
between osmo-bts-trx and an SDR. This port builds the UHD backend,
`osmo-trx-uhd`, for Ettus B2xx class devices such as the LibreSDR B220.

Upstream version: 1.8.0. Eight patches applied, five in the build system
and three in the daemon. Testsuite on macOS 26.6.2, Apple M5 Pro, UHD
4.10.0.0 from Homebrew: 7 pass, 1 skipped (LMSDeviceTest, the LimeSuite
backend is not built), 0 fail.

Upstream README preserved as [README.upstream.md](README.upstream.md).

## Prerequisites

- [libosmocore](https://github.com/AndreiGosman/libosmocore-macos-arm64) >= v0.2.3 to build, >= v0.2.4 to run under a BTS (see below)
- UHD >= 4.0 with the B2xx images (Homebrew: `brew install uhd`, then
  `uhd_images_downloader` if the images are missing)
- Boost (Homebrew: `brew install boost`; UHD pulls it in)
- FFTW single precision (Homebrew: `brew install fftw`); the multi-ARFCN
  code is enabled by default upstream and needs `fftw3f`

libosmocore v0.2.3 is a hard requirement. Its `cpu_sched_vty.c` is Linux
only and the port stubs the two public functions of that file. v0.2.2
stubbed only the initialiser; osmo-trx is the first daemon in this series
that calls `osmo_cpu_sched_vty_apply_localthread()` from its worker
threads, and against v0.2.2 `osmo-trx-uhd` fails to link with that symbol
undefined.

libosmocore v0.2.4 is needed as soon as osmo-bts-trx sends POWERON. The
rate counter timers in `CommonLibs/trx_rate_ctr.cpp` disarm their
timerfd from the read callback without a `read()`; the Darwin timerfd
emulation up to v0.2.3 left the descriptor readable in that case, so the
main thread spun at 100 % CPU and logged "Main thread is updating
Transceiver counters" about 260000 times per second. v0.2.4 makes
`timerfd_settime()` reset the pending count as Linux does. The fix is in
`libosmocore.dylib`; this daemon does not need to be rebuilt.

## Build

```bash
git clone https://github.com/AndreiGosman/osmo-trx-macos-arm64.git
cd osmo-trx-macos-arm64
autoreconf -fi
mkdir -p build && cd build
../configure --prefix=$HOME/sdr-lab/local --with-uhd
make -j$(sysctl -n hw.ncpu)
make check
make install
```

All dependencies are found through pkg-config; the prefix and the
Homebrew directories have to be on `PKG_CONFIG_PATH`. `--disable-doxygen`
is not an osmo-trx option. The other backends (`--with-lms`,
`--with-bladerf`, `--with-usrp1`, `--with-ipc`) and the MS side
(`--with-mstrx`) were not built; see "Not covered".

configure reports `whether g++ supports C++17 features with
-std=gnu++17... yes` and prints an autoconf warning that C++17 "is not
yet standardized", which comes from the age of the bundled macro and can
be ignored.

## SIMD on Apple Silicon

None. The build has two SIMD trees: `arch/x86` with SSE3 and SSE4.1
kernels, selected by `ax_sse.m4` from `$host_cpu`, and `arch/arm` with
NEON kernels, selected by `--with-neon`. The NEON kernels are ARMv7
assembly (`vld1.32`, `vmul.f32`, `bx lr`, compiled with `-mfpu=neon`),
which does not assemble for AArch64. So the build goes through
`arch/x86` with no SSE conditionals set and uses the generic C
convolution and conversion routines, autovectorised by clang. Patch 008
is what makes that path link on macOS.

`show trx` and the startup log report the SIMD state; the
`__builtin_cpu_supports` probe in configure answers "no" on arm64, so
runtime detection is off as well. Throughput under a real BTS load was
not measured; see "Not covered". Native AArch64 NEON intrinsics for the
four kernels (convert, convolve, scale, mult) would be the next step if
the generic path proves too slow.

## Running

The VTY listens on 127.0.0.1:4237, CTRL on 4236, and the transceiver
protocol for osmo-bts-trx on UDP 5700 and up on the configured
`bind-ip`.

```bash
osmo-trx-uhd -C doc/examples/osmo-trx-uhd/osmo-trx-uhd.cfg
```

Two things in the upstream example configuration do not apply here.
The `cpu-sched` block (`policy rr 18`) is rejected, because the
libosmocore port has no `cpu-sched` VTY node: the underlying
`sched_setaffinity` and `sched_setscheduler` are Linux only. Remove the
block, or on Darwin set a real-time policy by other means. And
`clock-ref external` expects a 10 MHz reference on the device; use
`internal` unless one is connected. The deprecated `rt-prio` option
(patch 007) applies `SCHED_RR` to the main thread only on Darwin.

Signals are handled through a pipe instead of `signalfd(2)` (patch
006); SIGINT and SIGTERM shut the daemon down, SIGUSR1 and SIGUSR2 print
talloc reports and SIGHUP reopens the logs, as on Linux.

## Hardware check

Checked against a LibreSDR B220 Mini (serial MAE8DOY, identifies as a
B210) with dummy loads on both TRX ports, using a configuration without
the `cpu-sched` block and with `clock-ref internal`. The daemon comes up
with 10 threads, opens VTY and CTRL, and UHD discovers and initialises
the device: `Detected Device: B210`, `Operating over USB 3`, both
register loopback tests pass, the master clock is set to 26 MHz, the
rates are configured for 4 SPS, gain ranges are read (Tx 0 to 89.75 dB,
Rx 0 to 76 dB) and the log ends with `Transceiver active with 1
channel(s)`. `show trx` on the VTY reports the configuration, and SIGINT
shuts the daemon down cleanly with exit code 0, which exercises the
signal pipe of patch 006.

One caveat for anyone repeating this: while UHD is inside the device
open the daemon does not answer on the VTY and ignores SIGINT, because
the select loop that serves both starts only after the device is up. A
device that hangs in the FX3 firmware load (it then shows on USB as
`WestBridge` 0x2500:0x0020) blocks `uhd_usrp_probe` the same way, and a
replug clears it.

The full loop with osmo-bts-trx and osmo-bsc is documented in the
[osmo-bts port](https://github.com/AndreiGosman/osmo-bts-macos-arm64):
POWERON acknowledged, 26 MHz master clock held through 2.5 minutes of
streaming, no device underrun, this daemon at 37 % CPU on an M5 Pro.

## Patches applied

| Patch | Upstream file | Issue | Fix |
|-------|---------------|-------|-----|
| 001 | `configure.ac` | The UHD 4.10 public headers use `std::optional`, `std::is_same_v` and `std::is_arithmetic_v`; configure asks for gnu++11 and `UHDDevice.cpp` fails to compile | Require C++17 with the bundled `AX_CXX_COMPILE_STDCXX` macro; the sources compile unchanged |
| 002 | `configure.ac` | The `uhd < 004.002` check that adds `-lboost_thread` reuses the `UHD` prefix of the first `PKG_CHECK_MODULES`, whose preset variables make it answer "yes" for every UHD version; on macOS the library is not on the link path and the build stops | Use `PKG_CHECK_EXISTS`, which asks pkg-config and sets nothing; the workaround still applies to libuhd < 4.2 |
| 003 | `Transceiver52M/arch/common/Makefile.am` | `fft.c` is compiled without `FFTWF_CFLAGS`, so `fftw3.h` is only found when it sits in a default include directory | Add `$(FFTWF_CFLAGS)` to `AM_CFLAGS` |
| 004 | 12 files under `Transceiver52M` | `<malloc.h>` and `memalign()` are glibc extensions; Darwin has neither | `<stdlib.h>` and `posix_memalign()` at the three aligned allocations, same 16 byte alignment |
| 005 | `CommonLibs/Threads.cpp` | `pthread_setname_np()` takes the thread as first argument on Linux and only the name on Darwin | Call the Darwin form there; the function only ever names the calling thread |
| 006 | `configure.ac`, `Transceiver52M/osmo-trx.cpp` | `signalfd(2)` is Linux only | Probe for `<sys/signalfd.h>`; without it, a handler writes the signal number into a pipe whose read end is registered with the select loop, so `sig_handler()` still runs from the main loop |
| 007 | `Transceiver52M/osmo-trx.cpp` | `sched_setscheduler(2)` is Linux only | Keep it on Linux; elsewhere `pthread_setschedparam()` on the calling thread for the deprecated `rt-prio` option |
| 008 | `Transceiver52M/arch/x86/Makefile.am` | `libarch_sse_3.la` and `libarch_sse_4_1.la` are declared unconditionally but get sources only under `HAVE_SSE3`/`HAVE_SSE4_1`; Apple `ar` refuses an archive with no members | Declare the two libraries inside the same conditionals as their sources |

A ninth commit tracks `.tarball-version` with the upstream version so
that `osmo-trx-uhd --version` reports 1.8.0 rather than the fork's tag.

Patches 001 to 004 and 008 are portability fixes with no effect on
GNU/Linux and are worth sending upstream; 001 and 002 also affect Linux
hosts with a recent UHD. Patches 005 to 007 add Darwin branches.

## Not covered

No BTS was attached, so the transceiver protocol (CLOCK, CTRL, DATA
sockets) was not exercised, no POWERON was issued and no burst went
through the signal processing chain under load; the device was opened
and configured but not streamed. `osmo-bts-trx` comes next in the series.

The MS side of the tree (`Transceiver52M/ms`, built with `--with-mstrx`)
uses `eventfd(2)`, `cpu_set_t` and `pthread_attr_setaffinity_np`, all
Linux only, and was not ported. The IPC backend links `-lrt`, which
Darwin does not have, and was not built either. LimeSuite, bladeRF and
USRP1 backends were not built for lack of hardware.

## Dependency cascade

Ports enabled by this repository:

- osmo-bts-trx, which drives this transceiver over UDP

## Prior ports in this series

1. libosmocore-macos-arm64 v0.2.3
2. libsctp-compat-macos-arm64 v0.3.1
3. srsRAN-4G-macos-arm64 v0.1.0
4. kraken-macos-arm64
5. libosmo-netif-macos-arm64 v0.1.2
6. libosmo-abis-macos-arm64 v0.1.1
7. libosmo-sigtran-macos-arm64 v0.1.0
8. osmo-hlr-macos-arm64 v0.1.0
9. osmo-mgw-macos-arm64 v0.1.0
10. libsmpp34-macos-arm64 v0.1.0
11. osmo-msc-macos-arm64 v0.1.0
12. osmo-bsc-macos-arm64 v0.1.0
13. This repository

## License

As upstream, per `debian/copyright`: AGPL-3.0-or-later for osmo-trx,
LGPL-2.1-or-later for `Transceiver52M/arch/arm/*`, GPL-3.0-or-later for
`CommonLibs/Makefile.am`, the `config/ax_*.m4` macros and `debian/*`.
The patches in this repository carry the license of the files they
modify.

## Credits

Port developed by Andrei Gosman ([@agoarchitecture](https://linkedin.com/in/agoarchitecture))
in collaboration with Claude Code CLI (Anthropic). All commits authored by
Andrei; Claude assisted with pattern analysis, debugging, and iteration.
