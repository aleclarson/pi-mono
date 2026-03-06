# Automatic Offline Fallback to Local Models

This guide explains how to configure Pi to automatically detect when you are offline (e.g., when the internet is down or unavailable) and automatically switch to a local Ollama model to ensure you can keep working without interruptions.

This setup relies on two features in Pi:
1. **Custom Models:** Registering your local Ollama instance in Pi.
2. **Extensions:** Intercepting the agent session startup to dynamically check internet connectivity and switch the model if needed.

## Step 1: Configure Ollama in `models.json`

First, Pi needs to know about your local Ollama models. You can add them in the `~/.pi/agent/models.json` configuration file.

Create or edit `~/.pi/agent/models.json` and add your local models:

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [
        {
          "id": "qwen2.5-coder:7b",
          "name": "Qwen 2.5 Coder 7B (Local)",
          "reasoning": false,
          "input": ["text"]
        },
        {
          "id": "llama3.1:8b",
          "name": "Llama 3.1 8B (Local)",
          "reasoning": false,
          "input": ["text"]
        }
      ]
    }
  }
}
```

*Note: You must have [Ollama](https://ollama.com/) running with the corresponding models pulled (`ollama run qwen2.5-coder:7b`).*

## Step 2: Create the Offline Fallback Extension

Next, we'll create an extension that runs right before the agent starts processing your prompt. It will ping a reliable host (like `1.1.1.1` or `8.8.8.8`) and, if it fails, switch Pi's current model to the local one.

Create a new file `~/.pi/agent/extensions/offline-fallback.ts` with the following content:

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";

// Replace these with your preferred local provider and model ID
const FALLBACK_PROVIDER = "ollama";
const FALLBACK_MODEL = "qwen2.5-coder:7b";

export default function (pi: ExtensionAPI) {
  pi.on("before_agent_start", async (_event, ctx) => {
    try {
      // Fast check: fetch a known reliable endpoint with a short timeout
      const controller = new AbortController();
      const timeoutId = setTimeout(() => controller.abort(), 1500); // 1.5 seconds

      await fetch("https://1.1.1.1", {
        method: "HEAD",
        signal: controller.signal,
      });

      clearTimeout(timeoutId);
      // Internet is up; do nothing and continue with the current model
    } catch (error) {
      // Internet appears to be down
      const currentModel = ctx.model;

      // Don't switch if we are already using a local model
      if (currentModel?.provider === FALLBACK_PROVIDER) {
         return;
      }

      const fallback = ctx.modelRegistry.find(FALLBACK_PROVIDER, FALLBACK_MODEL);
      if (fallback) {
        ctx.ui.notify(`Internet unreachable. Falling back to local model: ${fallback.name}`, "warning");
        await pi.setModel(fallback);
      } else {
        ctx.ui.notify(`Internet unreachable, and fallback model ${FALLBACK_PROVIDER}/${FALLBACK_MODEL} not found.`, "error");
      }
    }
  });
}
```

## How It Works

1. Whenever you send a prompt, the extension listens to the `before_agent_start` event.
2. It attempts a fast `HEAD` request to `1.1.1.1` with a 1.5-second timeout.
3. If the request succeeds, Pi continues using your current cloud model (e.g., Anthropic Claude).
4. If the request fails or times out, the extension catches the error.
5. It then searches the `modelRegistry` for your configured local model and calls `pi.setModel(fallback)`, smoothly transferring the current session to your local Ollama instance before the LLM request even starts.

If the internet comes back online, you can manually switch back to your cloud model using the `/model` command, or simply hit `Ctrl+L`.
