# Installing inferzero

(Moved from the README on 2026-10-31.)

Everything below runs in **Windows PowerShell** from the project folder (the folder that contains `build.ps1`).
If PowerShell refuses to run the scripts ("running scripts is disabled on this system"), start them as
`powershell -ExecutionPolicy Bypass -File <script> <arguments>`, or allow local scripts once for your user with
`Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`.

## Ready-to-use package (no compiler, no Python, no terminal)

The package (`inferzero-<version>-win64.zip`, 9 MB, **download: https://github.com/rizagp/inferzero/releases/latest**)
holds the programs already built. On a laptop with an Intel
Core Ultra (Arc graphics) and Intel's graphics driver, nothing else is needed:

1. Unpack the zip anywhere (right-click, "Extract All...", e.g. to `C:\inferzero`).
2. Double-click `zero-launcher.cmd`. Windows may ask once whether to run it (the zip came from the internet: "More
   info", "Run anyway"); the launcher then unblocks its own files. A package from before the rename (up to
   2026.11.10.3, `arcinferzero-<version>-win64.zip`) has `arc-launcher.cmd` instead.
3. The first time, a window offers a model: the one that suits the laptop's RAM is ticked (4B on 16 GB, the 35B on
   32 GB), each with a line on what it is good at; speech to text (recordings, meetings, voice messages) is an
   add-on, and a desktop shortcut is offered. "Download and start" fetches it with a progress bar (a cut connection
   goes on where it stopped), starts it, and opens the web UI (`http://127.0.0.1:8090/`) in the browser.

Later: double-click the shortcut (or `zero-launcher.cmd`), select a model, Start; "Get models" adds more. Without
the Intel driver the launcher says so and offers Intel's download page.

Behind it: the programs carry the C++ runtime inside (`/MT`) and ship the OpenMP runtime `vcomp140.dll` next to them
(no Visual C++ Redistributable); models download with `scripts\get-models.ps1` / `scripts\hf-get.ps1` (`curl.exe`,
part of Windows; every file's SHA256 checked) into the Hugging Face cache, from the list in
`scripts\model-catalog.ps1`. `setup.cmd` works too (the same models, plus Pi); `bin\PREBUILT.txt` marks the package,
so setup keeps the shipped programs (`setup.cmd -Rebuild` builds them from the sources in the package: step 3).
The package is made with `scripts\make-package.ps1` (into `dist\`).

## Linux

The headless server and a terminal launcher (no window), on Ubuntu 24.04 or newer (other distributions with glibc
2.39+; users run CachyOS too). The steps:

1. **Intel's GPU runtime.** Ubuntu's own packages: `sudo apt install intel-opencl-icd libze-intel-gpu1 libze1
   ocl-icd-libopencl1 libgomp1 curl`. Battlemage cards (B580, Pro B60 / B70) and Arrow Lake H need a newer
   compute runtime than Ubuntu 24.04 ships: Intel's packages (the `kobuk-team/intel-graphics` PPA their GPU pages
   point to), see [linux-port.md](linux-port.md).
2. **Access to the GPU.** `sudo usermod -aG render $USER`, then log in again.
3. **The package.** Download `inferzero-<version>-linux-x64.tar.gz` from the
   [latest release](https://github.com/rizagp/inferzero/releases/latest), then:
   ```
   tar -xzf inferzero-<version>-linux-x64.tar.gz
   cd inferzero-<version>-linux-x64
   ./zero-launcher.sh --check
   ./zero-launcher.sh
   ```
   `--check` lists what the system has and lacks. The menu offers the catalog's models (the ones that suit the RAM),
   downloads one with curl (resumes, checks SHA-256) into the Hugging Face cache, starts
   `bin/serverz` and opens the web UI at `http://127.0.0.1:8090/` when it is ready. Ctrl+C stops it.
4. **Without the menu:** `./zero-launcher.sh --model 4b` (a catalog key, see `--list`, or a path to a `.gguf`),
   `--download 4b`, `--port N`, `--host H`, `--gpus N`, `--no-browser`, and `-- <serverz options>`.
5. **As a service:** `./zero-launcher.sh --model 4b --install-service` writes a systemd user unit
   (`~/.config/systemd/user/inferzero.service`, with the cache variables you have set); then
   `systemctl --user daemon-reload`, `systemctl --user enable inferzero`, `systemctl --user restart inferzero`; its
   log: `journalctl --user -u inferzero -f`.
   An existing unit is left unchanged. For a separate test instance, pass
   `--service-name inferzero-test` and use that name in the printed commands.
   Use `--replace-service` only when you intend to replace an existing unit;
   replacement swaps the unit file itself rather than writing through a symlink.
   The user service starts when its user manager starts. To start the manager at boot before you log in and keep it
   running after logout, enable lingering for your account: `sudo loginctl enable-linger "$USER"`.
6. **In Docker** (run on Unraid with an Arc B580, no `ipc: host` or `privileged` needed: tenken, issue #40; the Kubernetes file is not tested yet): [docker/](../docker) holds a `Dockerfile` (Ubuntu 24.04, Intel's GPU runtime from
   the PPA, the latest release's Linux package) and a `compose.yaml`. The host needs only the GPU's kernel driver:
   ```
   cd docker
   docker compose up -d
   docker compose logs -f
   ```
   The web UI is then at `http://127.0.0.1:8090/`. `compose.yaml` sets the model (`MODEL: 4b`, a catalog key or a
   `.gguf` path), `GPUS`, `SERVER_ARGS`, and `OPTIONS`, the speed and accuracy options of [options.md](options.md) (e.g. `OPTIONS: "copy_draft,xcpu=off,budget=12"`; exactly these at each start, none kept). It passes `/dev/dri` through and mounts the Hugging Face cache and
   `~/.inferzero` (compiled kernels, expert files) from the host, so models and kernels are not fetched or built
   again. The container runs as root, so the files it writes there belong to root on the host. On Kubernetes,
   `docker/kubernetes.yaml` sets up a Deployment, a Service and a volume for the cache. The GPU comes from Intel's
   device plugin (`gpu.intel.com/i915`), and the image must be pushed to your own registry first (the file says
   how). Neither is tested yet: please report on GitHub how they run. If the GPU is not usable inside the
   container, `compose.yaml` has two commented lines to try: `ipc: host` first, `privileged: true` only if that is
   not enough (issue #40).
   On Unraid there is also a community template, kept by catduckgnaf, not by this project: [the Inferzero
   template](https://github.com/catduckgnaf/catduck-unraid-templates/blob/main/templates/inferzero.xml) and [its setup
   notes](https://github.com/catduckgnaf/catduck-unraid-templates#inferzero), with an image built from release
   2026.11.21.18 (issue #46). Its maintainer has started it on an Intel GPU host but not run a model with it yet.

From the sources instead: `./build.sh` (g++ 13 or newer) builds `runz`, `serverz`, `mgpu_probe`, `ref_forward` (the f32
reference for KL checks) and `gpu_bench` into `bin/`. The benchmarks have Linux versions in bash: `bench/website_bench.sh`,
`bench/gpu_sweep.sh` (the `.ps1` files on Windows). Nothing on Linux needs Python.
`scripts/make-package-linux.sh` makes the package (into `dist/`). Not on Linux yet: the window launcher, speech
recording, the NPU (built, not run), `web_view` (Edge); audio formats other than WAV and scanned-PDF pages need
`ffmpeg` / `pdftoppm` if installed ([linux-port.md](linux-port.md)).

## Quick setup

From the sources: once the graphics driver, Visual Studio Build Tools (C++) and optionally Pi are installed (steps
1–3 and 9.1 below), one command does the rest:

```powershell
.\setup.cmd                    # checks, builds, asks which models to download, connects Pi
.\setup.cmd -Models 27b,4b     # download these without asking (27b 35b 35b-gsq flash flash-ar4 9b 4b g12 g31 e4b, whisper asr: speech to text; all, none)
.\setup.cmd -Tests             # also build the test tools and run the regression tests (needs the 4B)
```

It checks the requirements (and says where to get what is missing), builds `bin\serverz.exe`, `runz.exe`,
`clinfo.exe`, `transcribez.exe` and `zero_recorder.exe` when they are missing or older than the sources, checks the GPU (OpenCL 3.0, DP4a, sub-groups, GPU
memory), downloads the chosen models into the Hugging Face cache without Python (Gemma with its drafter and image
projector; `35b-gsq` is converted by `bin\convert_gsq.exe`, `flash-ar4` imported by `bin\import_autoround_experts.exe`),
and
connects Pi (`zero-pi`, `zero-server`, Pi's model list with each model's context). Running it again only does what
is missing: a model the server already finds (in the Hugging Face cache, as a package in
`%USERPROFILE%\.inferzero\zero\models` or in a folder of `model-dirs.txt`, which `zero-server model-dirs -Add D:\ai\models` or the launcher's **More > Model folders...** adds) is not downloaded again. Double-clicked,
its window stays open at the end so the result can be read; it ends with setup's exit code (1 when a step failed).
Afterwards the easiest start is `zero-launcher.cmd` (see the package's steps above). Steps 1–10 describe the same by
hand.

### Flash Next with Intel's AutoRound INT4 experts (`flash-ar4`)

[Intel/Qwen3.8-Flash-Next-W4A16-AutoRound](https://huggingface.co/Intel/Qwen3.8-Flash-Next-W4A16-AutoRound) is
safetensors, not GGUF, so it is imported once (pull request #22). `setup -Models flash-ar4`, or on its own
`powershell -File scripts\get-autoround.ps1`, does all of it:

1. downloads the checkpoint's expert shards at the revision the importer checks (~78 GB; the shards without experts
   are skipped) and the file list with every file's SHA-256 (`model-info.json`, from the Hugging Face API);
2. imports the INT4 experts into an expert file (`bin\import_autoround_experts.exe`, ~64 GB), checking every code
   and scale;
3. downloads Unsloth's UD-Q4_K_XL Flash Next (~111 GB) for everything but the experts, and its image projector;
4. writes the package (`bin\import_autoround_pack.exe`, ~32 GB) next to the expert file in
   `%USERPROFILE%\.inferzero\zero\models\Qwen3.8-Flash-Next-AutoRound-INT4\`, where the server lists it as
   `qwen3.8-flash-next-autoround` (shortcut `flash-ar4`).

Then it asks whether to delete the AutoRound shards, which inferzero no longer reads (`-DeleteSource`: without
asking). Disk: ~290 GB at the most, ~207 GB kept with the shards deleted (the UD-Q4_K_XL stays: it is listed as a
model of its own). The model runs in Flash Next's precise mode by itself, without `--precise`; other models are not
affected. A run stopped halfway goes on where it stopped (the downloads resume; a finished expert file is kept).

## 1. Requirements

| | Needed | Tested with |
|---|---|---|
| OS | Windows 11, 64-bit | Windows 11 Home 25H2 (build 26200) |
| GPU | Intel Core Ultra with Intel Arc integrated graphics: Meteor Lake / Arrow Lake (Xe-LPG), and Lunar Lake / Panther Lake (Xe2 / Xe3: the 8-lane kernels are built with 16 lanes there, and the prompt GEMMs use XMX after a start-up check; a user's Arc B390 runs the 27B's 4K prompt at 583-592 tok/s, the 35B's at 1,370). The engine needs OpenCL 3.0 with `cl_intel_subgroups` and `cl_khr_integer_dot_product` (DP4a). Discrete Arc cards work experimentally, one or several for one model (`--gpus N`): the B-series (B580, Pro B60, Pro B70) and the A-series (Alchemist: A770, A750, A580). A dense model that nearly fills a card keeps part of its weights in RAM (a B580 12 GB runs the 27B that way): [xe2-tuning.md](xe2-tuning.md), [multi-gpu.md](multi-gpu.md). The kernels are compiled through OpenCL, so a card OpenCL does not list (an eGPU without Resizable BAR, as a user reports) cannot run them unless another card of the same kind is listed. | Core Ultra 5 125H, Arc iGPU 112 EU; users: Arc Pro B60 x1-4, Pro B70, B580, A770 x1-2, Arc 140T, Arc B390 |
| Memory | 16 GB works for the 4B / E4B models; 32 GB for the 12B–35B models and long contexts. The iGPU can use part of the RAM (`bin\clinfo.exe`, "global mem": set in Intel Graphics Software; 20.4 GiB of 32 GB on the test laptop now); inferzero uses all of it but 0.5 GiB for the driver, the rest of the RAM stays with Windows. Each model's context follows that setting. | 32 GB LPDDR5x-6400 |
| CPU | AVX2 (every Core Ultra has it; the build uses `/arch:AVX2`). On Linux, a CPU without AVX2 / FMA (Ivy Bridge, an older Xeon) runs a build made on it: `bash build.sh` builds for the CPU it runs on there (issue #50) | |
| Graphics driver | Intel Arc & Iris Xe graphics driver (it contains the OpenCL runtime; nothing else to install) | 32.0.101.9033 |
| Compiler | Visual Studio 2026 Build Tools with the **Desktop development with C++** workload (MSVC + Windows SDK); not for the ready-to-use package | VS Build Tools 18 |
| Python | not needed: downloads use `curl.exe`, the `35b-gsq` conversion and the `flash-ar4` import are exes of the package | |
| Pi (optional) | the Pi coding agent, which runs on Node.js | Node.js 24 |
| Disk | 3–13 GB per model | |

## 2. Graphics driver

Install or update the Intel graphics driver from Intel's download center ("Intel Arc & Iris Xe Graphics - Windows")
or with the Intel Driver & Support Assistant, then restart. The OpenCL runtime comes with the driver; there is no
SDK to install: inferzero loads `OpenCL.dll` at run time and the driver compiles the kernels.

## 3. Compiler

Install **Visual Studio 2026 Build Tools** (free, from visualstudio.microsoft.com, "Tools for Visual Studio" →
"Build Tools for Visual Studio 2026") and select the workload **Desktop development with C++** (it includes MSVC
and the Windows SDK). `build.ps1` finds the compiler itself (`vswhere`: any Visual Studio or Build Tools
installation with `VC\Auxiliary\Build\vcvars64.bat`, Community and Professional included); to use a particular
one, set `$env:ARC_VCVARS` to its `vcvars64.bat`.

## 4. Get the code

Copy or clone the `inferzero` folder anywhere (the scripts find everything relative to their own location). It
already contains everything it needs to build (`third_party\` holds minja and nlohmann/json).

## 5. Build

```powershell
.\build.ps1 zero\serverz.cpp zero\runz.cpp tools\clinfo.cpp zero\transcribez.cpp tools\zero_recorder.cpp
```

This takes a few minutes and writes `bin\serverz.exe` / `bin\runz.exe` (inferzero: the HTTP server and the
command-line tool for runs, benchmarks and accuracy checks) and `bin\clinfo.exe`. `zero\serverz.cpp` compiles
`tools\server.cpp` (the HTTP API) on the Level Zero engine. Run the programs from the project folder, because they
read `zero\kernels\*.cl` and `webui\` from there. They are built with the C++ runtime inside (`/MT`) and the OpenMP
runtime `vcomp140.dll`, which `build.ps1` copies into `bin\` from Visual Studio's redistributable folder: the folder
runs on a PC without Visual Studio or the Visual C++ Redistributable.

For discrete cards, the package also holds `mgpu_probe.exe`, `event_probe.exe` and `bench\replay_probe.exe` (what
moving data and syncing between GPUs, and launching kernels, cost). To build them:
`.\build.ps1 zero\mgpu_probe.cpp zero\event_probe.cpp bench\replay_probe.cpp`.

Check that the GPU is usable:

```powershell
.\bin\clinfo.exe
```

It should list `Intel(R) Arc(TM) Graphics`, `OpenCL 3.0 NEO`, `cl_khr_integer_dot_product yes`,
`cl_intel_subgroups yes` and sub-group sizes `8 16 32`.

Optional tools (tests and benchmarks):

```powershell
.\build.ps1 tools\tok_test.cpp tools\chat_render.cpp tools\ref_forward.cpp tools\gguf_dump.cpp tools\tool_stream_test.cpp
.\build.ps1 zero\vision_testz.cpp tools\gemma4v_ref.cpp tools\gemma4a_ref.cpp
.\build.ps1 bench\gemv_bench.cpp bench\attn_dec_bench.cpp bench\attn_bench.cpp bench\bw_bench.cpp
```

`zero\vision_testz` times an image through a model's encoder (Gemma 4 or Qwen3-VL `mmproj`, `ARC_PROF=1` per kernel)
and with `ARC_VIS_LM=file` writes the model's log-probabilities for 32 tokens after the image, to compare two encoder
builds or against `tools\gemma4v_ref` (the Gemma 4 encoder in f32 on the CPU); an audio clip with E4B's `mmproj`
the same against `tools\gemma4a_ref` (its audio conformer in f32; `ARC_AUDIO_EMB` runs the model on those rows). `runz --bench-batch N --prof-gen`
shows each kernel's GPU time of a decoding step alone and batched; `ARC_KLIST=1` prints the cache file and build
options of every kernel, to read the driver's own assembly with Intel's `ocloc disasm`
(docs/kernel-audit-2026-10-27.md). The source builds warning-free at
`/W4 /permissive-`; `.github/workflows/ci.yml` builds every exe and runs the model-free unit tests on each push.

## 6. Download models

No Python: `powershell -File scripts\get-models.ps1 4b asr` downloads models by the keys of `setup.cmd -Models`
(`scripts\model-catalog.ps1`; a Gemma model with its drafter copied next to it), and
`powershell -File scripts\hf-get.ps1 <org/name> <file or folder> ...` any repository's files. Both use curl.exe,
resume after a cut, check each file's SHA256 and put the files in the Hugging Face cache
(`%USERPROFILE%\.cache\huggingface\hub`, or `HF_HUB_CACHE` / `HF_HOME`), which is where `zero-server` / `zero-pi`
look for them, so no paths have to be configured. For the Gemma models, the drafter (`mtp-*.gguf`) and the image
projector (`mmproj-*.gguf`) must sit in the same folder as the model. (Hugging Face's own `hf download` fills the
same cache, if you have it.)

| Alias | Model | Command | Size |
|---|---|---|---|
| `27b` | Qwen3.8 27B GSQ-RCO (with MTP head) | `powershell -File scripts\get-models.ps1 27b` | 12.1 GB |
| `9b` | Qwen3.5 9B (with MTP head) | `powershell -File scripts\get-models.ps1 9b` | 6.1 GB |
| `4b` | Qwen3.5 4B (with MTP head) | `powershell -File scripts\get-models.ps1 4b` | 3.0 GB |
| `g12` | Gemma 4 12B QAT (Google, Q4_0) + image projector; drafter from Unsloth, copied into the model's snapshot folder | `powershell -File scripts\get-models.ps1 g12` | 7.0 + 0.2 + 0.3 GB |
| `g31` | Gemma 4 31B QAT (Google, Q4_0) + image projector; drafter from Unsloth, copied into the model's snapshot folder. On a 32 GB laptop part of its weights go into host memory (see USAGE.md) | `powershell -File scripts\get-models.ps1 g31` | 17.7 + 1.2 + 0.3 GB |
| `e4b` | Gemma 4 E4B QAT (Google, Q4_0); drafter from Unsloth, copied into the model's snapshot folder | `powershell -File scripts\get-models.ps1 e4b` (images, audio) | 5.2 + 0.1 GB |
| `g26` | Gemma 4 26B-A4B QAT (Google, Q4_0) + image projector; drafter `mtp-gemma-4-26B-A4B-it.gguf` from Unsloth's Gemma 4 26B-A4B QAT GGUF repository, copied next to the model (optional: ~25 tok/s with it) | `powershell -File scripts\hf-get.ps1 google/gemma-4-26B-A4B-it-qat-q4_0-gguf gemma-4-26B_q4_0-it.gguf gemma-4-26B-it-mmproj.gguf` | 14.4 + 1.2 GB |
| `35b` | Qwen3.6 35B-A3B UD-IQ4_XS (Unsloth's MTP repository: MTP block + image projector) | `powershell -File scripts\get-models.ps1 35b` | 18.2 GB + projector |
| `35b-mtp` | Qwen3.6 35B-A3B UD-Q3_K_M of the same repository | `powershell -File scripts\hf-get.ps1 unsloth/Qwen3.6-35B-A3B-MTP-GGUF Qwen3.6-35B-A3B-UD-Q3_K_M.gguf mmproj-BF16.gguf` | |
| (`35b`) | Qwen3.6 35B-A3B, 2-bit GSQ, no MTP (what `setup.cmd -Models 35b` downloads) | see step 7 (download 12.6 GB, then convert) | 11.7 GB |
| `flash` | Qwen3.8 Flash Next (inferzero only); images: an image projector copied into the model's folder (the server looks only next to the model; with a package: next to the package) | `powershell -File scripts\get-models.ps1 flash`; projector `powershell -File scripts\hf-get.ps1 bartowski/Qwen3.8-Flash-Next-GGUF mmproj-Qwen3.8-Flash-Next-f16.gguf` | 54.8 + 28.8 GB + 0.9 GB |
| | DFlash2 drafters (optional, see `--dflash` below) | `powershell -File scripts\hf-get.ps1 z-lab/Qwen3.8-27B-DFlash2 config.json model.safetensors` / `powershell -File scripts\hf-get.ps1 incoai/Qwen3.6-35B-A3B-DFlash2 config.json model.safetensors` | 3.85 / 1.05 GB |
| | Speech to text (optional, `zero-server start -Whisper auto`): Whisper large-v3, and Qwen3-ASR 1.7B (the default speech model; its mmproj next to it) | `powershell -File scripts\get-models.ps1 whisper asr` | 2.9 + 2.4 GB |

With their `mmproj` file next to the model, Gemma 4 12B and E4B read images and audio, and Gemma 4 26B, Qwen3.5
4B / 9B, Qwen3.8 27B and Qwen3.6 35B read images (all encoders run on the GPU).
Transcripts use Qwen3-ASR (the default), Whisper large-v3 (turbo only when large-v3 is not cached) or Gemma 4 E4B
([docs/qwen3-asr.md](qwen3-asr.md#using-it)); Gemma 4 12B is refused for transcripts: it has no audio encoder
(its projector maps raw 40-ms waveform frames straight into the model), and on noisy recordings of mixed Indonesian
and English it made up sentences or repeated a word where E4B's conformer transcribed them.

Shortcuts (`zero-pi <shortcut>`, `zero-server start -Model <shortcut>`): `27b`, `35b`, `35b-s`, `35b-mtp`, `flash`,
`9b`, `4b`, `g12`, `e4b`, `g26`. Other GGUFs of these architectures get an id from their file name (`zero-pi --pick`
lists them). `35b` is the UD-IQ4_XS file above, or UD-Q3_K_M / the GSQ file when that is the one
downloaded (with several, the first found keeps the shortcut); `35b-s` is Unsloth's UD-Q3_K_S. Variant suffixes:
`-mtp` (MTP, no images), `-vision` (images, no MTP), `-std` (neither), and for a model with a DFlash2 drafter
`-nodf` / `-df` (see `--dflash` below).

## 7. Qwen3.6-35B-A3B (optional): convert the GSQ checkpoint

ISTA-DASLab publishes this model only as a safetensors checkpoint (compressed-tensors: 2-bit routed experts
`{-2,-1,0,1} x bf16 scale per 128`, int8 attention / DeltaNet / shared experts / lm_head). Convert it once
(`setup.cmd -Models 35b-gsq` does all of this; by hand, with `bin\convert_gsq.exe` of the package, or built with
`.\build.ps1 tools\convert_gsq.cpp`; no Python). The tokenizer metadata comes from any GGUF with the same
vocabulary, e.g. the 4B from step 6:

```powershell
. .\scripts\hf-get.ps1
$repo = "ISTA-DASLab/Qwen3.6-35B-A3B-2Bit-GSQ"
$files = @(Invoke-RestMethod "https://huggingface.co/api/models/$repo/tree/main?recursive=1" | ForEach-Object { $_ } | Where-Object { $_.type -eq "file" } | ForEach-Object { $_.path })
$snap = Get-HfFiles $repo $files
$vocab = (Get-ChildItem "$env:USERPROFILE\.cache\huggingface\hub" -Recurse -Filter "Qwen3.5-4B-UD-Q4_K_XL.gguf" | Select-Object -First 1).FullName
New-Item -ItemType Directory -Force "$env:USERPROFILE\.inferzero\models" | Out-Null
bin\convert_gsq.exe $snap "$env:USERPROFILE\.inferzero\models\Qwen3.6-35B-A3B-GSQ2-arc.gguf" --vocab-from $vocab
```

The int8 matrices become Q8_0 with the same int8 values (f16 scales), the experts keep their exact 2-bit codes in
the layout of `zero/kernels/moe.cl`, and the tensor transforms follow llama.cpp's converter (DeltaNet V-head order,
norm weights + 1, A = -exp(A_log)). The checkpoint's MTP block has no routed experts, so this model runs without
speculative decoding. `zero-server` / `zero-pi` find the converted file in `%USERPROFILE%\.inferzero\models`.

## 8. First run

```powershell
.\scripts\zero-server.ps1 start -Model e4b      # or 27b, 35b, flash, 9b, 4b, g12, g26
```

This starts inferzero. The first start of each model compiles its GPU
kernels (up to a minute or two); they are cached in `%USERPROFILE%\.inferzero\kernels`, so later starts take 5–40 s
(mostly reading the model file). The first inferzero start of a mixture-of-experts model also writes its expert
file once (35B UD-IQ4_XS: 16 GB, about 2.5 minutes; Qwen3.8 Flash Next: 56 GB, about 9.5 minutes; later loads through the
server ~21–26 s) to `%USERPROFILE%\.inferzero\zero\`; it can be deleted at any time and is written again when missing. Then open the web
UI at http://127.0.0.1:8090/ or call the OpenAI API at `http://127.0.0.1:8090/v1`. `zero-server status` shows what
is running and `zero-server stop` frees the GPU memory. Without Pi the command is `zero-server.cmd` in the inferzero
folder (setup offers to put the folder on PATH); `zero-server help` lists the commands (`list`, `logs`, `doctor`,
`version`, `pull`, presets, `-DryRun`).

**Launcher window.** `zero-launcher.cmd` (double-click it in the project folder) opens a window to pick a model,
set its options (with presets, and whether they fit the GPU memory) and start or stop the server; it follows
Windows' light / dark mode and waits in the tray when minimized: [USAGE.md, section 3](../USAGE.md#3-managing-the-server).
When something does not work, `zero-server doctor` (or the launcher's More > "Check this PC") makes a report to
paste into a GitHub issue.

## 9. Pi (optional)

1. Install Pi by following the instructions on https://pi.dev and run it once, so that `%USERPROFILE%\.pi\agent`
   exists (with `bin` on the PATH).
2. Add inferzero to Pi (one time, `setup.cmd` does it too; it builds `bin\serverz.exe` if it is missing, adds the
   `inferzero` provider with every model it finds and its context for the current GPU memory to
   `~/.pi/agent/models.json` (the previous file is kept as `models.json.bak`; `zero-pi` refreshes the list), and
   installs the `zero-pi` and `zero-server` commands):

   ```powershell
   powershell -ExecutionPolicy Bypass -File scripts\install-pi.ps1
   ```

3. Open a new terminal and start Pi from any project folder:

   ```powershell
   zero-pi                  # the model the server runs or used last (the first time: choose from a list)
   zero-pi 27b              # a model by its shortcut (step 6);  zero-pi --pick = choose from every downloaded model
   zero-pi --help           # every option
   ```

   Inside Pi, `/model` switches between all of them (the server loads the chosen one). Every zero-pi and
   zero-server command: [USAGE.md, sections 2 and 3](../USAGE.md#2-using-pi).

## 10. Check the installation (optional)

With the test tools from step 5 built, the regression tests compare every kernel with a CPU reference, the
tokenizer and chat templates with the reference output, the model's next-token distributions with an f32 CPU
implementation, and the server API:

```powershell
powershell -ExecutionPolicy Bypass -File tests\run_tests.ps1               # inferzero, models found automatically
powershell -ExecutionPolicy Bypass -File tests\run_tests.ps1 -SkipGpu      # only the checks without a GPU / model
powershell -ExecutionPolicy Bypass -File tests\run_tests.ps1 -ListModels   # which model each check would use
```

It runs the model-free checks first (HTTP parser, API formats, JSON, sampler, expert-cache pools, tool-call
streaming, attention and routing kernels, chat templates against the GGUFs' Jinja templates), then per model found
(4B, 27B, 35B, Flash Next, Gemma 4 12B / E4B / 26B) the KL and greedy checks and MTP / DFlash2 =
plain decoding, then a server smoke test; a missing model is a SKIP, not a failure. It ends with `ALL TESTS PASSED`. Stop `zero-server` first: the
test uses port 8091 and needs the GPU memory.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `cannot load OpenCL.dll` / `no OpenCL GPU found` | Install or update the Intel graphics driver (step 2) and restart. |
| `build failed` right away / `No Visual Studio C++ compiler found` | Visual Studio's C++ workload is missing (step 3); to use a particular installation, set `$env:ARC_VCVARS` to its `vcvars64.bat`. `setup.cmd` checks this. The compiler output of `setup.cmd` is in `%USERPROFILE%\.inferzero\build.log`. |
| `running scripts is disabled on this system` | Run the script through `powershell -ExecutionPolicy Bypass -File ...` (see the top of this section). |
| `Model file ... not found` | Download it with `scripts\get-models.ps1` or `scripts\hf-get.ps1` (step 6); `zero-server` searches `%USERPROFILE%\.inferzero\models` and the Hugging Face cache (models on another drive: [USAGE.md, section 8](../USAGE.md#8-troubleshooting)). |
| The first start of a model takes minutes | The driver compiles the kernels once; later starts use `%USERPROFILE%\.inferzero\kernels`. |
| Gemma does not use its drafter | `mtp-gemma-4-*-it.gguf` must be in the same folder as the model GGUF (download it from the same repository). |
| Later problems (the server stops while loading, Windows gets slow, Pi's `web_search`, slow first sessions, models on another drive) | [USAGE.md, section 8](../USAGE.md#8-troubleshooting). |
