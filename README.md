# Offline Qwen - on-device chat app

A from-scratch Android app that runs a **GGUF** model (e.g. `Qwen3-1.7B-Q4_K_M.gguf`)
fully offline using **llama.cpp**, with voice input, PDF export, chat history, and
adjustable generation settings. Built from source via GitHub Actions - no PC needed.

## ⚠️ Please read before building
This is real native (C++/JNI) code, not a template. I wrote it carefully against
llama.cpp's current API, but **I could not compile or run it in my own environment**
(no Android SDK/NDK, no network access there). llama.cpp's C API changes over time,
so the very first GitHub Actions build may fail with a compiler error if something
shifted upstream. That's normal for this kind of project - paste me the **Actions
build log** (the red step) and I'll fix the exact line. Expect 1-3 rounds of this
before it's green.

## Get the model file (~1.1 GB)
Download `Qwen3-1.7B-Q4_K_M.gguf` from Hugging Face (search "Qwen3 1.7B GGUF",
e.g. a Bartowski or Unsloth quant) using your phone's browser, save it to your
Downloads folder. No account/token needed for this one (unlike gated models).

## Build WITHOUT a PC (GitHub Actions)
1. Create a new, empty GitHub repo.
2. In Termux (or any shell with git):
   ```
   cd OfflineQwen
   git init && git add . && git commit -m "init"
   git branch -M main && git remote add origin <your repo url> && git push -u origin main
   ```
3. On GitHub: **Actions** tab → "Build APK" → **Run workflow** → wait
   (first build compiles llama.cpp from source: ~10-20 min) → open the run →
   **Artifacts** → download `OfflineQwen-debug-apk`, unzip, install `app-debug.apk`
   (allow "install unknown apps" when prompted).
4. Open the app → **Pick model file (.gguf)** → select the Qwen3 file you downloaded.

## Build with Android Studio instead
File → Open → select this folder → wait for Gradle sync (first sync also compiles
llama.cpp, can take a while) → Run.

## How it works
- `app/src/main/cpp/CMakeLists.txt` - pulls llama.cpp source straight from GitHub
  at build time (`FetchContent`) and compiles it for `arm64-v8a`, CPU-only.
- `app/src/main/cpp/llama-bridge.cpp` - the JNI bridge: loads the model, tokenizes,
  runs the decode loop, samples (temperature/top-p/top-k), streams text back token
  by token.
- `LlamaEngine.kt` - Kotlin side of the JNI bridge.
- `ModelManager.kt` - copies a picked `.gguf` file into the app's private storage.
- `ChatViewModel.kt` - app state, settings persistence (SharedPreferences), builds
  the ChatML-style prompt Qwen expects, drives generation.
- `ChatHistoryStore.kt` - saves/loads the conversation as JSON (no database needed).
- `VoiceInputHelper.kt` - launches Android's built-in speech-recognition screen.
- `PdfExporter.kt` - renders the transcript to a PDF with Android's built-in
  `PdfDocument` (no extra library) and shares it via `FileProvider`.
- `ui/ChatScreen.kt` / `ui/SettingsScreen.kt` - Jetpack Compose UI.

## Features included
- Pick any `.gguf` model file from storage (not hardcoded to Qwen3 - any GGUF
  model small enough for your RAM will work).
- Streaming responses, chat history saved across app restarts, "New chat".
- Voice input (mic button fills the text box; availability depends on the
  device's installed speech service, not guaranteed 100% offline on every phone).
- Export the current chat to a PDF and share/save it.
- Settings: temperature, top-p, top-k, max tokens per reply, context size,
  CPU thread count, system prompt. Context size/threads reload the model.
- Single active model at a time; "Delete model" frees the storage.

## Honest limits for your phone (4 GB RAM)
- CPU-only inference (no GPU delegate) - this is the safest choice for a 4 GB
  device, but it means "fast" is relative: expect a handful of tokens per
  second on a 1.7B Q4 model, not instant replies.
- Keep `maxTokens` and `contextSize` modest (defaults: 512 / 2048) - pushing
  these up raises RAM/CPU load and the real ceiling depends on how much RAM
  is actually free when you run it.
- Close other apps before loading the model for best results.

## Known things NOT done (tell me if you want them next)
- No GPU/NNAPI acceleration - adds real complexity and isn't a safe default
  on a 4 GB device.
- No multi-model switching UI beyond "Change model" (single slot).
- No automatic context-window truncation yet - a very long chat can eventually
  exceed `contextSize` and generation will degrade or fail; "New chat" resets it.
