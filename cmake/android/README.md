# Android source-build adapter

This optional CMake entry is maintained by tencentmalos, independently of the
FFmpeg build system. The initial branch is based on upstream `n7.1.5`
(`3a0867c2bfda4a4d4309ca1a8cbdc6175e67f587`). Consumers pin the submodule commit;
the branch name is not a reproducibility lock.

Call `add_subdirectory(path/to/ffmpeg/cmake/android ffmpeg-android)` in an
Android NDK project and link `FFmpeg::ffmpeg`. The adapter currently supports
`arm64-v8a`; validation uses native API 33. Build out of tree, with a separate
CMake build directory per NDK/API/profile. Installed headers and all six archives
come from that build. No prebuilt archive, host pkg-config package, or external
codec source is downloaded.

The default builds software decoders, parsers, demuxers, filters, scaling and
resampling, using NDK zlib. It excludes programs, encoders, muxers, device
capture, hardware accelerators and network protocols. Network support and
explicit decoder/demuxer allowlists are CMake options; changing them changes the
media capability profile. This is not the configuration for an encoder or
Android MediaCodec integration.

The six PIC archives are private to the consuming shared library. The aggregate
target hides their exports with narrowly scoped `--exclude-libs` options, which
also bind AArch64 assembly table references locally. The adapter does not apply
`-Bsymbolic` globally to the host, or allow duplicate definitions. Individual
`FFmpeg::avcodec` etc. targets expose archive paths; use the aggregate target for
the complete cyclic link group, system libraries and visibility requirements.

`FFMPEG_ANDROID_JOBS` controls make workers. No C++ runtime is bundled by FFmpeg;
the embedding application owns its NDK/STL profile and runtime packaging.

Validation in the shadPS4 integration repository uses a real shared library and
an Android executable, checks header/runtime versions and software H.264 decode,
COPY_OPAQUE propagation, scale/resample and the audio decoder registry. This does
not establish full player behavior, all codec compatibility or APK acceptance.
