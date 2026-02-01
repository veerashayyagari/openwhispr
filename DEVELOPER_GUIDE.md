# OpenWhispr Developer Guide

A beginner-friendly guide to understanding and working with the OpenWhispr codebase.

## Table of Contents

1. [What is OpenWhispr?](#what-is-openwhispr)
2. [The Big Picture: Electron Architecture](#the-big-picture-electron-architecture)
3. [Folder Structure](#folder-structure)
4. [How Recording Works](#how-recording-works-data-flow)
5. [The Two Windows](#the-two-windows)
6. [Key Files to Start With](#key-files-to-start-with)
7. [Common Patterns](#common-patterns)
8. [Configuration Reference](#configuration-reference)
9. [Adding New Features](#adding-new-features)
10. [Debugging Tips](#debugging-tips)
11. [Working with Claude Code](#working-with-claude-code)

---

## What is OpenWhispr?

OpenWhispr is a **desktop dictation app** - you press a hotkey, speak, and it converts your speech to text. Think of it like Siri or Google voice typing, but running locally on your computer with privacy options.

**Key features:**
- Press a hotkey → speak → text appears at your cursor
- Works with local AI models (private) or cloud APIs (faster)
- Supports "agent" commands like "Hey Jarvis, write an email..."

---

## The Big Picture: Electron Architecture

OpenWhispr uses **Electron**, which lets you build desktop apps using web technologies. Here's the mental model:

```
┌─────────────────────────────────────────────────────────────┐
│                       Your Computer                          │
│                                                              │
│  ┌────────────────────┐         ┌────────────────────────┐  │
│  │   MAIN PROCESS     │◄──IPC──►│   RENDERER PROCESS     │  │
│  │   (Node.js)        │         │   (React in Browser)   │  │
│  │                    │         │                        │  │
│  │ What it can do:    │         │ What it can do:        │  │
│  │ • Read/write files │         │ • Show UI (buttons)    │  │
│  │ • Run whisper.cpp  │         │ • Handle user clicks   │  │
│  │ • Access database  │         │ • Record audio         │  │
│  │ • System hotkeys   │         │ • Display settings     │  │
│  │ • Clipboard paste  │         │                        │  │
│  └────────────────────┘         └────────────────────────┘  │
│            ▲                                                 │
│            │                                                 │
│  ┌─────────┴──────────┐                                     │
│  │    preload.js      │  (Secure bridge - exposes only      │
│  │                    │   safe functions to the browser)    │
│  └────────────────────┘                                     │
└─────────────────────────────────────────────────────────────┘
```

### Why Two Processes?

**Security**: The browser window (renderer) can't directly access your files or run programs. This prevents malicious web content from harming your computer.

**Communication**: They talk through **IPC** (Inter-Process Communication) - like passing notes between them.

### The Three Key Files

| File | Role | Think of it as... |
|------|------|-------------------|
| `main.js` | Main process entry point | The "backend" server |
| `src/App.jsx` | React app entry | The "frontend" website |
| `preload.js` | Security bridge | The "API" between them |

---

## Folder Structure

```
openwhispr/
│
├── main.js                 # App starts here (main process)
├── preload.js              # Bridge between main & renderer
│
├── src/
│   ├── App.jsx             # Main dictation overlay UI
│   ├── main.jsx            # React entry point
│   │
│   ├── components/         # React UI components
│   │   ├── ControlPanel.tsx    # Settings window
│   │   ├── SettingsPage.tsx    # Settings form
│   │   ├── OnboardingFlow.tsx  # First-time setup wizard
│   │   └── ui/                 # Reusable UI pieces (buttons, cards)
│   │
│   ├── hooks/              # React hooks (state management)
│   │   ├── useAudioRecording.js  # Microphone recording logic
│   │   ├── useSettings.ts        # App settings state
│   │   └── useLocalStorage.ts    # Save data in browser
│   │
│   ├── helpers/            # Main process modules
│   │   ├── whisper.js          # Runs speech-to-text
│   │   ├── parakeet.js         # Alternative AI model
│   │   ├── audioManager.js     # Handles microphone
│   │   ├── clipboard.js        # Pastes text for you
│   │   ├── hotkeyManager.js    # Global keyboard shortcuts
│   │   ├── database.js         # SQLite for history
│   │   └── ipcHandlers.js      # Receives messages from UI
│   │
│   ├── services/           # Business logic
│   │   └── ReasoningService.ts # AI text processing
│   │
│   └── models/             # AI model definitions
│       └── ModelRegistry.ts    # Available AI models
│
├── resources/              # Native binaries
│   └── bin/                # whisper-cpp, ffmpeg, etc.
│
└── scripts/                # Build & download scripts
    ├── download-whisper-cpp.js
    └── download-sherpa-onnx.js
```

---

## How Recording Works (Data Flow)

Here's what happens when you press the hotkey and speak:

```
STEP 1: HOTKEY PRESSED
────────────────────────────────────────────────────────────────
  hotkeyManager.js detects the keypress
       │
       ▼
  Sends "toggle-recording" event via IPC to renderer
       │
       ▼
  App.jsx receives event, starts recording


STEP 2: RECORDING AUDIO
────────────────────────────────────────────────────────────────
  useAudioRecording.js uses browser's MediaRecorder API
       │
       ▼
  Audio chunks collect in memory as you speak
       │
       ▼
  Visual feedback shows in App.jsx (pulsing microphone)


STEP 3: HOTKEY PRESSED AGAIN (stop)
────────────────────────────────────────────────────────────────
  Recording stops
       │
       ▼
  Audio chunks → Blob → ArrayBuffer
       │
       ▼
  Sent to main process via IPC ("transcribe-audio" channel)


STEP 4: TRANSCRIPTION (in main process)
────────────────────────────────────────────────────────────────
  audioManager.js receives the audio data
       │
       ▼
  Writes audio to a temporary file
       │
       ▼
  whisper.js runs whisper-cpp (or cloud API)
       │
       ▼
  Returns transcribed text


STEP 5: OUTPUT
────────────────────────────────────────────────────────────────
  Text sent back to renderer via IPC
       │
       ▼
  If agent command detected → ReasoningService processes it
       │
       ▼
  clipboard.js copies text and simulates Ctrl+V
       │
       ▼
  Text appears where your cursor was!
```

---

## The Two Windows

OpenWhispr has two windows that share the same React codebase:

### 1. Main Window (App.jsx)
- **What**: Small floating overlay showing recording status
- **When**: Always visible when app is running
- **Features**: Draggable, stays on top, minimal UI

### 2. Control Panel (ControlPanel.tsx)
- **What**: Full settings and history interface
- **When**: Opens from tray icon or menu
- **Features**: Model selection, API keys, transcription history

**How it works**: Both load the same React app, but different components based on URL:
```javascript
// In main.jsx
if (window.location.search.includes('panel=control')) {
  // Show ControlPanel
} else {
  // Show main App overlay
}
```

---

## Key Files to Start With

If you want to understand the code, read in this order:

| Order | File | What You'll Learn |
|-------|------|-------------------|
| 1 | `main.js` | How the app boots up, creates windows |
| 2 | `src/App.jsx` | The main UI you interact with |
| 3 | `src/hooks/useAudioRecording.js` | How audio recording works |
| 4 | `src/helpers/whisper.js` | How speech becomes text |
| 5 | `src/helpers/ipcHandlers.js` | How frontend talks to backend |
| 6 | `preload.js` | The security bridge |

---

## Common Patterns

### Pattern 1: IPC Communication

Frontend and backend communicate through "channels":

```javascript
// STEP 1: In preload.js - Define the bridge
contextBridge.exposeInMainWorld('electronAPI', {
  transcribeAudio: (data) => ipcRenderer.invoke('transcribe-audio', data)
});

// STEP 2: In ipcHandlers.js - Handle the message
ipcMain.handle('transcribe-audio', async (event, audioData) => {
  const result = await whisper.transcribe(audioData);
  return result;
});

// STEP 3: In React component - Call it
const text = await window.electronAPI.transcribeAudio(audioData);
```

### Pattern 2: Settings Storage

Settings are stored in two places:

| Location | What's Stored | Why |
|----------|---------------|-----|
| `localStorage` | User preferences (model, language) | Quick access in browser |
| `.env` file | API keys | Persist across sessions |
| SQLite database | Transcription history | Searchable, structured |

```javascript
// In useSettings.ts - Using localStorage
const [whisperModel, setWhisperModel] = useLocalStorage("whisperModel", "base");

// API keys sync to .env file
setOpenaiApiKey(key);  // Saves to localStorage AND .env
```

### Pattern 3: React Hooks

Custom hooks encapsulate logic:

```javascript
// useAudioRecording.js
function useAudioRecording() {
  const [isRecording, setIsRecording] = useState(false);

  const startRecording = async () => { /* ... */ };
  const stopRecording = async () => { /* ... */ };

  return { isRecording, startRecording, stopRecording };
}

// Usage in App.jsx
const { isRecording, startRecording, stopRecording } = useAudioRecording();
```

---

## Configuration Reference

All configuration options and what they control:

### Transcription Settings

| Setting | localStorage Key | Default | What It Does |
|---------|-----------------|---------|--------------|
| Use Local Whisper | `useLocalWhisper` | `false` | `true` = process audio on your computer; `false` = send to cloud |
| Whisper Model | `whisperModel` | `"base"` | Which local model to use: `tiny`, `base`, `small`, `medium`, `large`, `turbo` |
| Local Provider | `localTranscriptionProvider` | `"whisper"` | Local engine: `"whisper"` (whisper.cpp) or `"nvidia"` (Parakeet) |
| Parakeet Model | `parakeetModel` | `""` | Which Parakeet model (e.g., `"parakeet-tdt-0.6b-v3"`) |
| Cloud Provider | `cloudTranscriptionProvider` | `"openai"` | Cloud service: `"openai"`, `"groq"`, `"custom"` |
| Cloud Model | `cloudTranscriptionModel` | `"gpt-4o-mini-transcribe"` | Which cloud model to use |
| Language | `preferredLanguage` | `"en"` | Language code for transcription (e.g., `"en"`, `"es"`, `"fr"`) |
| Custom Dictionary | `customDictionary` | `[]` | Array of words to improve recognition |

**Model Size Trade-offs:**
```
tiny   (75MB)  → Fastest, lowest quality
base   (142MB) → Good balance (recommended)
small  (466MB) → Better quality
medium (1.5GB) → High quality
large  (3GB)   → Best quality, slowest
turbo  (1.6GB) → Fast with good quality
```

### Reasoning/AI Settings

| Setting | localStorage Key | Default | What It Does |
|---------|-----------------|---------|--------------|
| Use Reasoning | `useReasoningModel` | `true` | Enable AI text cleanup and commands |
| Reasoning Model | `reasoningModel` | `""` | Which AI model for text processing |
| Reasoning Provider | `reasoningProvider` | (computed) | Derived from model: `"openai"`, `"anthropic"`, `"gemini"`, `"groq"`, `"local"` |

### Hotkey Settings

| Setting | localStorage Key | Default | What It Does |
|---------|-----------------|---------|--------------|
| Dictation Key | `dictationKey` | `""` (auto) | The hotkey to start/stop recording |
| Activation Mode | `activationMode` | `"tap"` | `"tap"` = press twice; `"push"` = hold to record |

**Platform Defaults:**
- macOS: Globe/Fn key (falls back to F8)
- Windows/Linux: Backtick (`) (falls back to F8)

### Microphone Settings

| Setting | localStorage Key | Default | What It Does |
|---------|-----------------|---------|--------------|
| Prefer Built-in Mic | `preferBuiltInMic` | `true` | Avoid Bluetooth mic switching issues |
| Selected Device | `selectedMicDeviceId` | `""` | Specific microphone device ID |

### API Keys

| Setting | localStorage Key | Environment Variable | What It Does |
|---------|-----------------|---------------------|--------------|
| OpenAI Key | `openaiApiKey` | `OPENAI_API_KEY` | For OpenAI transcription and GPT models |
| Anthropic Key | `anthropicApiKey` | `ANTHROPIC_API_KEY` | For Claude models |
| Gemini Key | `geminiApiKey` | `GEMINI_API_KEY` | For Google Gemini models |
| Groq Key | `groqApiKey` | `GROQ_API_KEY` | For Groq fast inference |

### Agent Settings

| Setting | localStorage Key | Default | What It Does |
|---------|-----------------|---------|--------------|
| Agent Name | `agentName` | `"Assistant"` | The name you use to address the AI (e.g., "Hey Jarvis") |

### Fallback Settings

| Setting | localStorage Key | Default | What It Does |
|---------|-----------------|---------|--------------|
| Allow OpenAI Fallback | `allowOpenAIFallback` | `false` | Fall back to cloud if local fails |
| Allow Local Fallback | `allowLocalFallback` | `false` | Fall back to local if cloud fails |
| Fallback Model | `fallbackWhisperModel` | `"base"` | Which local model to use as fallback |

### Onboarding

| Setting | localStorage Key | Default | What It Does |
|---------|-----------------|---------|--------------|
| Completed Onboarding | `hasCompletedOnboarding` | `false` | Skip first-time setup wizard |

---

## Adding New Features

### Adding a New Setting

1. **Add to useSettings.ts:**
```typescript
const [mySetting, setMySetting] = useLocalStorage("mySetting", defaultValue);
```

2. **Add UI in SettingsPage.tsx:**
```tsx
<Input value={mySetting} onChange={(e) => setMySetting(e.target.value)} />
```

3. **If it needs to reach main process, add to preload.js and ipcHandlers.js**

### Adding a New IPC Channel

1. **In preload.js:**
```javascript
myNewFunction: (data) => ipcRenderer.invoke('my-new-channel', data)
```

2. **In ipcHandlers.js:**
```javascript
ipcMain.handle('my-new-channel', async (event, data) => {
  // Handle the request
  return result;
});
```

3. **In React:**
```javascript
const result = await window.electronAPI.myNewFunction(data);
```

### Adding a New Helper Module

1. Create file in `src/helpers/myHelper.js`
2. Import and initialize in `main.js`
3. Export functions that can be called from ipcHandlers.js

---

## Debugging Tips

### Enable Debug Mode

```bash
# Option 1: Command line
npm run dev -- --log-level=debug

# Option 2: Add to .env file
OPENWHISPR_LOG_LEVEL=debug
```

### Log File Locations

| Platform | Path |
|----------|------|
| macOS | `~/Library/Application Support/OpenWhispr/logs/debug-*.log` |
| Windows | `%APPDATA%\OpenWhispr\logs\debug-*.log` |
| Linux | `~/.config/OpenWhispr/logs/debug-*.log` |

### Common Issues

| Problem | Check This |
|---------|------------|
| No audio detected | Microphone permissions, audio device selection |
| Transcription fails | whisper.cpp binary exists, model downloaded |
| Text not pasting | Accessibility permissions (macOS), paste tools (Linux) |
| Hotkey not working | Check for conflicts, try different key |

### Useful Debug Commands

```bash
# Check if whisper.cpp is found
ls resources/bin/

# Run with verbose output
npm run dev -- --log-level=debug

# Check database
sqlite3 ~/.config/OpenWhispr/openwhispr.db ".tables"
```

---

## Working with Claude Code

This project is configured for [Claude Code](https://claude.ai/claude-code) with custom agents and skills for code review and development workflows.

### Available Commands

| Command | Description |
|---------|-------------|
| `/review [PR#]` | Review a PR against project standards |
| `/review [PR#] --post` | Review and post comments to GitHub |
| `/pr-comments [PR#]` | Fetch PR comments and fix issues |

### Automated Behaviors

| Trigger | What Happens |
|---------|--------------|
| Claude completes code changes | Runs `npm run format:check` to validate |
| Code review task detected | Delegates to `code-reviewer` agent |
| `/review` command | Forks to code-reviewer, analyzes diff |
| `/review --post` | Posts structured review to GitHub PR |
| `/pr-comments` | Fetches comments, applies fixes |

### Code Review Standards

The `code-reviewer` agent enforces these patterns:

**Must Fix (blocking):**
- Async operations in constructors
- Missing IPC listener cleanup functions
- Unstructured error responses
- Direct Node.js usage in renderer

**Request Changes:**
- `console.log` instead of `debugLogger`
- Untyped React components (should be `.tsx`)
- Settings bypassing `useSettings` hook
- Missing types in `src/types/electron.ts`

### Review Workflow

```bash
# 1. Contributor opens PR #42

# 2. Review and post comments
/review 42 --post

# 3. Claude reads comments and fixes
/pr-comments 42

# 4. Re-review if needed
/review 42 --post
```

### Validation Commands

```bash
# Check for lint/format issues
npm run format:check

# Auto-fix lint/format issues
npm run format:js
```

### Configuration Files

```
.claude/
├── settings.json           # Permissions (gh, npm run)
├── agents/
│   └── code-reviewer.md    # Review standards & patterns
└── skills/
    ├── review/SKILL.md     # /review command
    └── pr-comments/SKILL.md # /pr-comments command
```

---

## Quick Reference: Data Flow

```
User speaks
    ↓
MediaRecorder API (browser)
    ↓
Audio Blob → ArrayBuffer
    ↓
IPC: "transcribe-audio"
    ↓
audioManager.js → temp file
    ↓
whisper.js → whisper.cpp binary
    ↓
Text result
    ↓
IPC: response
    ↓
(Optional) ReasoningService → AI cleanup
    ↓
clipboard.js → paste
    ↓
Text appears at cursor!
```

---

## Next Steps

1. **Run the app**: `npm run dev`
2. **Read main.js** to see how it starts
3. **Read App.jsx** to see the main UI
4. **Make a small change** and see it hot-reload
5. **Check the debug logs** to understand the flow

Happy coding!
