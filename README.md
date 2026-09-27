# adreno-llm

> **This is a field report, not software.** There is no code in this repository. It is a written record of measurements on real hardware, including the trap that defeats the obvious approach.
Local LLM inference on Snapdragon/Adreno Android devices — OpenCL backend guide for llama.cpp.

Field-tested on an Odin2 Portal (Snapdragon 8 Gen 2 / Adreno 740, 16GB unified RAM, Android 13)
via Termux. Qwen3-1.7B Q8_0: **20.2 tok/s generation** with the GPU OpenCL path vs **13.1 tok/s**
CPU-only — a 55% speedup, on hardware you already own.

## Why OpenCL (not Vulkan) on Adreno
llama.cpp's Vulkan backend requires a shaderc/glslang version that is often NEWER than what
Termux ships (symptom: `shaderc: internal error: ... Invalid capability operand: 5447` during
build). The **OpenCL backend compiles cleanly** and can use Qualcomm's *own* OpenCL driver
(usually faster than a Vulkan translation layer anyway).

## The trap everyone hits: llvmpipe fallback
After a successful OpenCL build, llama-server may silently fall back to **llvmpipe**
(a CPU software renderer) because Termux's ICD loader cannot see Qualcomm's driver.
Symptom in the log:
```
ggml_opencl: unsupported GPU 'llvmpipe (LLVM 21.1.8, 128 bits)'.
```
Check with `clinfo` — if the platform is llvmpipe instead of QUALCOMM, the GPU is NOT being used.

## The fix
Point `LD_LIBRARY_PATH` at the vendor's OpenCL driver when launching llama-server:
```bash
LD_LIBRARY_PATH=/system/vendor/lib64 ./llama-server -m model.gguf --port 8080
```
Then `clinfo` (with the same env) shows: `Platform Name: QUALCOMM Snapdragon(TM)`.

## Full setup (Termux)
```bash
pkg update
pkg install git cmake clang python opencl-headers vulkan-headers vulkan-tools vulkan-icd glslang shaderc ocl-icd clinfo
git clone --depth 1 https://github.com/ggml-org/llama.cpp
git clone --depth 1 https://github.com/KhronosGroup/SPIRV-Headers
cd SPIRV-Headers && cmake -B build -DCMAKE_INSTALL_PREFIX=$PREFIX && cmake --install build && cd ..
cmake -B build-llama llama.cpp -DGGML_OPENCL=ON -DLLAMA_CURL=OFF -DGGML_CCACHE=OFF   -DOpenCL_LIBRARY=/system/vendor/lib64/libOpenCL.so   -DOpenCL_INCLUDE_DIR=$PREFIX/include
cmake --build build-llama --target llama-server llama-cli -j2
# download a model, then:
LD_LIBRARY_PATH=/system/vendor/lib64 ./build-llama/bin/llama-server -m model.gguf --port 8080
```

## Notes learned the hard way
- `/tmp` does not exist on Android — build in `$HOME`.
- Wi-Fi SSH to a phone/handheld can time out mid-banner; USB (`adb forward tcp:8022 tcp:8022`) is stable.
- `pkill` from an SSH session can kill the session itself — use `setsid ... &` for servers.
- Keep the device on USB power for long runs (`svc power stayon usb`).

## License
MIT — see LICENSE.
