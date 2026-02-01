# Azure AI Foundry Integration

Technical specification for adding Azure AI Foundry support to OpenWhispr.

## Overview

This document describes the implementation plan for integrating Azure AI Foundry as a transcription and reasoning provider, enabling users to use Azure-hosted models like `gpt-4o-mini-transcribe` and `gpt5-nano`.

## Requirements

### Goal
Add Azure AI Foundry as a first-class provider for:
- **Transcription:** gpt-4o-mini-transcribe (or other Azure STT deployments)
- **Reasoning:** gpt5-nano (or other Azure chat deployments)

### Key Differences from OpenAI

| Aspect | OpenAI | Azure AI Foundry |
|--------|--------|------------------|
| Endpoint | `https://api.openai.com/v1/audio/transcriptions` | `{ENDPOINT}/openai/deployments/{DEPLOYMENT}/audio/transcriptions?api-version=2024-02-01` |
| Auth Header | `Authorization: Bearer {key}` | `api-key: {key}` |
| Model | In request body: `model: "gpt-4o-mini-transcribe"` | In URL path as deployment name |
| API Version | Not needed | Required: `?api-version=2024-02-01` |

### New Settings

| Setting | localStorage Key | Description | Default |
|---------|-----------------|-------------|---------|
| Azure Endpoint | `azureEndpoint` | Resource URL (e.g., `https://your-resource.openai.azure.com`) | `""` |
| Azure API Key | `azureApiKey` | Azure OpenAI API key | `""` |
| Transcription Deployment | `azureTranscriptionDeployment` | Deployment name for STT | `""` |
| Reasoning Deployment | `azureReasoningDeployment` | Deployment name for text cleanup | `""` |
| API Version | `azureApiVersion` | Azure API version | `"2024-02-01"` |

## Architecture

### Design Decision: First-Class Provider

Azure is implemented as a **separate provider** (not extending "custom") because:

1. **Unique authentication:** `api-key` header vs `Authorization: Bearer`
2. **Unique URL structure:** Deployment name in path, api-version query param
3. **Future-proofing:** Clean path for adding Managed Identity support later
4. **User experience:** Dedicated UI with Azure-specific field labels

## Implementation

### Stage 1: Backend Infrastructure

**Files:**
- `src/helpers/environment.js` - Add `getAzureKey()`, `saveAzureKey()`
- `src/helpers/ipcHandlers.js` - Add IPC handlers
- `preload.js` - Expose to renderer

### Stage 2: Settings Management

**Files:**
- `src/hooks/useSettings.ts` - Add 5 Azure settings with IPC sync
- `src/config/constants.ts` - Add `API_VERSIONS.AZURE`

### Stage 3: Transcription Logic

**File:** `src/helpers/audioManager.js`

Key changes:
```javascript
// Header construction
if (provider === "azure") {
  headers["api-key"] = apiKey;
} else if (apiKey) {
  headers.Authorization = `Bearer ${apiKey}`;
}

// Endpoint construction for Azure
const url = `${endpoint}/openai/deployments/${deployment}/audio/transcriptions?api-version=${apiVersion}`;

// Skip model in FormData for Azure (it's in the URL)
if (provider !== "azure") {
  formData.append("model", model);
}
```

### Stage 4: Reasoning Logic

**File:** `src/services/ReasoningService.ts`

- Add `processWithAzure()` method
- Use `api-key` header
- Build Azure-style endpoint for chat completions

### Stage 5: UI Components

**Files:**
- `src/components/TranscriptionModelPicker.tsx` - Azure tab and config
- `src/components/ReasoningModelSelector.tsx` - Azure option
- `src/components/SettingsPage.tsx` - Pass Azure props
- `src/models/modelRegistryData.json` - Add Azure provider

## Files Summary

| File | Changes |
|------|---------|
| `src/helpers/environment.js` | Azure key getter/setter, env persistence |
| `src/helpers/ipcHandlers.js` | Azure key IPC handlers |
| `preload.js` | Expose Azure key methods |
| `src/hooks/useSettings.ts` | 5 new Azure settings with IPC sync |
| `src/config/constants.ts` | `API_VERSIONS.AZURE` |
| `src/helpers/audioManager.js` | Azure endpoint, headers, skip model |
| `src/services/ReasoningService.ts` | `processWithAzure()`, Azure auth |
| `src/models/ModelRegistry.ts` | Azure provider detection |
| `src/models/modelRegistryData.json` | Azure transcription provider |
| `src/components/TranscriptionModelPicker.tsx` | Azure tab and config UI |
| `src/components/ReasoningModelSelector.tsx` | Azure option and config UI |
| `src/components/SettingsPage.tsx` | Pass Azure props |

## Testing

### Manual Testing Checklist

1. **Settings UI:**
   - [ ] Azure tab appears in Transcription settings
   - [ ] Azure option appears in AI Models/Reasoning settings
   - [ ] All Azure fields save correctly
   - [ ] API key persists after app restart

2. **Transcription Flow:**
   - [ ] Select Azure provider → enter config
   - [ ] Record audio → transcription completes
   - [ ] Verify `api-key` header in debug logs
   - [ ] Verify URL format matches Azure pattern

3. **Reasoning Flow:**
   - [ ] Configure Azure reasoning deployment
   - [ ] Agent-addressed commands work correctly

4. **Error Handling:**
   - [ ] Missing endpoint shows clear error
   - [ ] Missing deployment shows clear error
   - [ ] Invalid API key shows auth error

5. **Regression Testing:**
   - [ ] OpenAI transcription still works
   - [ ] Groq transcription still works
   - [ ] Custom provider still works

### Debug Mode

```bash
OPENWHISPR_LOG_LEVEL=debug npm run dev
```

## Future Enhancements

- **Managed Identity:** Use `DefaultAzureCredential` instead of API keys
- **Deployment Discovery:** Query Azure for available deployments
- **Multi-region:** Support multiple Azure endpoints

## References

- [Azure OpenAI Whisper Quickstart](https://learn.microsoft.com/en-us/azure/ai-foundry/openai/whisper-quickstart)
- [Azure OpenAI Audio API](https://learn.microsoft.com/en-us/azure/ai-foundry/openai/concepts/audio)
