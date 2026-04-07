# Agxr — Architecture

**Concept:** Claude Code on Android, fully local. No internet, no API key.
Local SLM + Alpine Linux VM + agent loop, all running on-device.

---

## Stack Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Flutter UI (Dart)                         │
│   ChatScreen   │   SettingsScreen   │   VM status indicator     │
└────────────────────────────┬────────────────────────────────────┘
                             │ Platform Channel
                             │ com.ai2th.agxr/agent
┌────────────────────────────▼────────────────────────────────────┐
│                       Kotlin Orchestrator                        │
│  AgentManager  │  LlmEngine  │  VmManager  │  VmBridge          │
│  AgentService (Foreground)                                       │
└──────────┬───────────────────────────┬──────────────────────────┘
           │ JNI                       │ ProcessBuilder
           ▼                           ▼
   ┌───────────────┐         ┌─────────────────────┐
   │  libllama.so  │         │  libqemu.so          │
   │  llama.cpp    │         │  QEMU aarch64        │
   │  Qwen 2.5 1.5B│         │  Minimal Alpine VM   │
   │  GGUF Q4_K_M  │         │  Serial console I/O  │
   └───────────────┘         └─────────────────────┘
```

---

## Components

### 1. Flutter UI

**ChatScreen**
- Conversation history (user + agent messages)
- Streaming token output — tokens appear as LLM generates
- Code blocks with syntax highlighting + copy button
- Tool call display — shows commands being executed and their output
- Send button / text input

**SettingsScreen**
- Model path (downloaded GGUF)
- RAM allocation for VM (256–2048 MB slider)
- vCPU count
- System prompt editor
- Download model button (first run)

**VM Status bar**
- Stopped / Starting / Running indicator
- Inline, does not take a full screen

---

### 2. Kotlin Layer

#### AgentManager.kt
Central orchestrator. Runs the agent loop:

```
user message
    → build prompt (system + history + tools)
    → LlmEngine.generate() [streaming]
    → parse tool calls from output
    → for each tool call:
        → VmBridge.execute(cmd)
        → append result to context
    → repeat until no more tool calls
    → return final response to Flutter
```

#### LlmEngine.kt
JNI wrapper around libllama.so:
```kotlin
fun loadModel(path: String)
fun generate(prompt: String, onToken: (String) -> Unit): String
fun unload()
```
Runs inference on a dedicated thread. Streams tokens back via callback.

#### VmManager.kt
Spawns and manages the QEMU process (derived from Linxr).
Modifications from Linxr:
- No SSH port forward (not needed)
- Serial console wired to VmBridge instead of being drained
- Smaller Alpine image (no OpenRC, no sshd)

#### VmBridge.kt
Reads/writes QEMU stdin/stdout (serial console):
```kotlin
fun runCommand(cmd: String, timeoutMs: Long = 30_000): CommandResult
```
Protocol:
- Write: `$cmd; echo __EXIT_$?__\n`
- Read stdout until `__EXIT_N__` marker
- Returns stdout + exit code

#### AgentService.kt
Android Foreground Service. Keeps LLM inference and QEMU alive.
Notification: "Agxr — Agent running"

---

### 3. Native Libraries (jniLibs/arm64-v8a/)

| Library | Source | Purpose |
|---------|--------|---------|
| `libllama.so` | llama.cpp (NDK build) | LLM inference engine |
| `libqemu.so` | QEMU (from Linxr) | VM execution |
| `libqemu_img.so` | qemu-img (from Linxr) | QCOW2 overlay creation |
| + deps | llama.cpp deps | ggml, metal (if applicable) |

Both libllama.so and libqemu.so use the same `exec_type` SELinux trick —
Android's package manager installs them into `nativeLibraryDir` with
the label that permits `execve()` without root.

---

### 4. LLM — Qwen 2.5 1.5B Instruct

| Property | Value |
|----------|-------|
| Model | Qwen2.5-1.5B-Instruct |
| Format | GGUF Q4_K_M |
| Size | ~900 MB |
| License | Apache 2.0 |
| Tool use | Native (JSON function calling) |
| Context | 32K tokens |
| Speed (S23, CPU) | ~20 tok/s |
| Speed (S23, Vulkan) | ~45 tok/s |

**Why Qwen 2.5 1.5B:**
- Best tool/function calling at this size
- Apache 2.0 — no restrictions
- Excellent at code generation and shell commands
- Multilingual
- Fits in ~900 MB RAM

**Model distribution:**
- NOT shipped in APK (too large)
- Downloaded on first run from HuggingFace
- Stored in `filesDir/models/qwen2.5-1.5b-q4.gguf`
- Progress shown in Settings screen

---

### 5. VM — Minimal Alpine Linux

Derived from Linxr's Alpine image, stripped down for agent use.

**Removed from Linxr image:**
- OpenRC init system
- sshd / openssh-server
- Boot sequence replaced with direct shell

**Kept:**
- busybox + ash shell
- apk package manager
- networking (SLIRP — for `apk add`)
- Python 3 (pre-installed)
- curl, wget

**Boot:**
```
kernel -append "... init=/bin/sh"
```
Drops directly into root shell on serial console (`/dev/ttyAMA0`).
Ready in ~3–5 seconds (no init system overhead).

**Storage:**
- `base.qcow2` — read-only rootfs backing file (~80 MB decompressed)
- `user.qcow2` — writable overlay (4 GB cap)
- QEMU command identical to Linxr minus SSH forward

**SLIRP networking (for package installs):**
- VM can reach the internet for `apk add`
- No ports exposed to host (no SSH needed)

---

### 6. Tool Definitions (System Prompt)

Qwen 2.5 receives these tools in the system prompt:

```json
[
  {
    "name": "run_command",
    "description": "Execute a shell command in the Linux VM. Returns stdout and exit code.",
    "parameters": {
      "command": { "type": "string" }
    }
  },
  {
    "name": "write_file",
    "description": "Write text content to a file path in the VM.",
    "parameters": {
      "path": { "type": "string" },
      "content": { "type": "string" }
    }
  },
  {
    "name": "read_file",
    "description": "Read the contents of a file in the VM.",
    "parameters": {
      "path": { "type": "string" }
    }
  },
  {
    "name": "install_package",
    "description": "Install an Alpine Linux package using apk.",
    "parameters": {
      "package": { "type": "string" }
    }
  }
]
```

---

## Agent Loop — Detailed

```
1. User sends: "write a python script that checks if a port is open"

2. AgentManager builds prompt:
   [system: you are a local AI assistant with access to a Linux VM. tools: ...]
   [user: write a python script that checks if a port is open]

3. LlmEngine generates (streaming to UI):
   "I'll write a Python script for that."
   <tool_call>{"name": "write_file", "parameters": {"path": "/tmp/portcheck.py", "content": "import socket\ndef check_port(host, port):\n    ..."}}

4. AgentManager detects tool call, calls VmBridge:
   VmBridge.runCommand("cat > /tmp/portcheck.py << 'EOF'\nimport socket\n...\nEOF")

5. VM executes, returns exit code 0

6. AgentManager appends to context:
   [tool_result: success]

7. LlmEngine continues:
   <tool_call>{"name": "run_command", "parameters": {"command": "python3 /tmp/portcheck.py google.com 80"}}

8. VmBridge runs command, returns output

9. LlmEngine continues:
   "The script works. Here's what it does: ..."

10. Final response streamed to ChatScreen
```

---

## RAM Budget

| Component | RAM |
|-----------|-----|
| Android OS | ~2.0 GB |
| Flutter app | ~150 MB |
| Qwen 2.5 1.5B Q4 | ~900 MB |
| llama.cpp context | ~200 MB |
| Alpine VM (minimal) | ~256 MB |
| **Total** | **~3.5 GB** |

Minimum device RAM: **6 GB**
Recommended: **8 GB**

---

## Project Structure

```
agxr/
├── android/
│   └── app/src/main/
│       ├── kotlin/com/ai2th/agxr/
│       │   ├── AgxrApp.kt          # Application singleton
│       │   ├── MainActivity.kt     # Platform channel
│       │   ├── AgentManager.kt     # Agent loop orchestrator
│       │   ├── LlmEngine.kt        # llama.cpp JNI wrapper
│       │   ├── VmManager.kt        # QEMU lifecycle (from Linxr)
│       │   ├── VmBridge.kt         # Serial console I/O
│       │   └── AgentService.kt     # Foreground service
│       ├── jniLibs/arm64-v8a/
│       │   ├── libllama.so         # llama.cpp
│       │   ├── libqemu.so          # QEMU
│       │   ├── libqemu_img.so      # qemu-img
│       │   └── [llama deps]
│       └── assets/
│           └── vm/
│               ├── base.qcow2.gz   # Minimal Alpine rootfs
│               ├── vmlinuz-virt
│               └── initramfs-virt
├── lib/
│   ├── main.dart
│   ├── screens/
│   │   ├── chat_screen.dart        # Main chat interface
│   │   └── settings_screen.dart   # Model + VM settings
│   ├── services/
│   │   └── agent_platform.dart    # Platform channel wrapper
│   └── widgets/
│       ├── message_bubble.dart    # Chat message + code blocks
│       ├── tool_call_card.dart    # Shows tool executions inline
│       └── vm_status_bar.dart     # VM state indicator
├── scripts/
│   ├── build_apk.sh
│   ├── build_llama.sh             # Compile llama.cpp for arm64
│   └── build_rootfs.sh            # Minimal Alpine image
├── pubspec.yaml
└── ARCH.md
```

---

## Build Pipeline

### libllama.so (new)
```bash
# Clone llama.cpp
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp

# Build for Android arm64 with NDK
cmake -B build-android \
  -DCMAKE_TOOLCHAIN_FILE=$NDK/build/cmake/android.toolchain.cmake \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-26 \
  -DLLAMA_VULKAN=ON \
  -DBUILD_SHARED_LIBS=ON
cmake --build build-android --target llama
```
Output: `libllama.so` → copy to `jniLibs/arm64-v8a/`

### libqemu.so (reuse from Linxr)
Symlink or copy from `../alpine/android/app/src/main/jniLibs/arm64-v8a/`

### Minimal Alpine rootfs (new)
Strip Linxr's `_build_rootfs.sh`:
- Remove OpenRC setup
- Remove sshd install
- Set `init=/bin/sh` in kernel append
- Pre-install: `python3 curl wget git`
- Target compressed size: ~4 MB

---

## Key Differences from Linxr

| | Linxr | Agxr |
|--|-------|------|
| Purpose | Run Alpine VM | Local AI agent |
| LLM | None | Qwen 2.5 1.5B |
| VM access | SSH (port 2222) | Serial console (stdin/stdout) |
| Alpine image | Full (OpenRC + sshd) | Minimal (shell + python only) |
| UI | Start/Stop + SSH terminal | Chat interface |
| Boot time | ~12s | ~4s (no init system) |
| APK size | ~63 MB | ~63 MB + model download |

---

*Architecture v1.0 — April 2026*
