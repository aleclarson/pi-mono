# Tauri OAuth Integration Guide

When embedding `pi-coding-agent` in a desktop application built with [Tauri](https://tauri.app/), you likely want to handle authentication flows natively within your app rather than requiring users to drop into a CLI to run `/login`.

This guide explains how to implement the `OAuthLoginCallbacks` interface and use the `AuthStorage` class to provide a seamless GUI-based login experience for OAuth providers like Anthropic, GitHub Copilot, or custom extensions.

## Understanding the Flow

The `pi-coding-agent` manages credentials (both API keys and OAuth tokens) using the `AuthStorage` class.

To initiate an OAuth login, you call `AuthStorage.login(providerId, callbacks)`. The `callbacks` object must implement `OAuthLoginCallbacks` from `@mariozechner/pi-ai`, which the underlying provider will use to interact with the user (e.g., opening a browser, showing a device code, or prompting for input).

## 1. Setting up AuthStorage

By default, `AuthStorage` reads and writes to `~/.pi/agent/auth.json`. In a Tauri app, you might want to use the app's standard data directory.

```typescript
import { AuthStorage } from "@mariozechner/pi-coding-agent";
import { appDataDir, join } from "@tauri-apps/api/path";

// Get the native app data directory
const dataDir = await appDataDir();
const authPath = await join(dataDir, "auth.json");

// Create AuthStorage instance pointing to your app's directory
const authStorage = AuthStorage.create(authPath);
```

## 2. Implementing OAuthLoginCallbacks in Tauri

When you call `login()`, the provider will invoke methods on your callbacks object. In a Tauri app, you can use Tauri's APIs and your frontend framework to handle these natively.

First, ensure you have the necessary Tauri plugins installed, particularly the shell plugin for opening URLs:

```bash
npm run tauri add shell
# or your package manager equivalent
```

Here's an example implementation of the callbacks:

```typescript
import { open } from "@tauri-apps/plugin-shell";
import type { OAuthLoginCallbacks } from "@mariozechner/pi-ai";

function createTauriAuthCallbacks(
  // You might pass in UI state setters here, e.g., React/Vue state hooks
  showDeviceCodeDialog: (code: string, url: string) => void,
  showPromptDialog: (message: string, isSecret: boolean) => Promise<string>,
  updateStatus: (message: string) => void
): OAuthLoginCallbacks {
  return {
    // Called when the user needs to authorize via a standard OAuth browser flow
    onAuth: async ({ url, instructions }) => {
      updateStatus(instructions || "Opening browser for authentication...");
      // Use Tauri's shell plugin to open the system default browser
      await open(url);
    },

    // Called when the provider uses the OAuth Device Flow
    onDeviceCode: async ({ userCode, verificationUri, instructions }) => {
      // Show a UI component with the device code and a button to open the URL
      showDeviceCodeDialog(userCode, verificationUri);
      updateStatus(instructions || `Please enter code ${userCode} in your browser.`);
    },

    // Called when the provider needs to prompt the user for input (e.g., a CLI token)
    onPrompt: async ({ message, isSecret }) => {
      // Show a native Tauri dialog or a custom modal in your frontend
      return await showPromptDialog(message, isSecret);
    },

    // Called to provide progress updates during the login flow
    onProgress: (message: string) => {
      updateStatus(message);
    }
  };
}
```

## 3. Initiating the Login

With your `AuthStorage` and callbacks ready, you can wire them up to a "Login" button in your Tauri frontend.

```typescript
async function handleLogin(providerId: string) {
  const callbacks = createTauriAuthCallbacks(
    // your state management functions...
    (code, url) => { setDeviceCodeState({ code, url }); },
    (msg, secret) => { return askUserViaDialog(msg, secret); },
    (msg) => { setStatusMessage(msg); }
  );

  try {
    setStatusMessage(`Starting login for ${providerId}...`);

    // This will initiate the flow, invoke your callbacks, and save the resulting
    // credentials into the auth.json file managed by AuthStorage.
    await authStorage.login(providerId, callbacks);

    setStatusMessage(`Successfully logged in to ${providerId}!`);
  } catch (error) {
    setStatusMessage(`Login failed: ${error.message}`);
  }
}
```

## 4. Custom Storage Backends (Optional)

If your app requires storing credentials in a secure system keychain rather than a plain text JSON file, you can implement the `AuthStorageBackend` interface and pass it to a new `AuthStorage` instance instead of using `AuthStorage.create()`.

```typescript
import { AuthStorage, type AuthStorageBackend, type AuthStorageData } from "@mariozechner/pi-coding-agent";
import { invoke } from "@tauri-apps/api/core";

class TauriKeychainBackend implements AuthStorageBackend {
  // Implement withLock and withLockAsync to read/write from Tauri's secure storage
  withLock<T>(fn: (current: string | undefined) => { result: T; next?: string }): T {
    // Sync keychain operations (if available)
    throw new Error("Synchronous storage not supported by this backend");
  }

  async withLockAsync<T>(fn: (current: string | undefined) => Promise<{ result: T; next?: string }>): Promise<T> {
    // Read from your secure storage via Tauri commands
    const current = await invoke<string>("read_secure_credentials");

    const { result, next } = await fn(current || undefined);

    if (next !== undefined) {
      // Write back to secure storage
      await invoke("write_secure_credentials", { data: next });
    }

    return result;
  }
}

// Usage:
const secureAuthStorage = new AuthStorage(new TauriKeychainBackend());
```

## Summary

By providing a custom `OAuthLoginCallbacks` implementation using Tauri's native APIs and integrating with `AuthStorage`, your application can offer a fully integrated, GUI-driven authentication experience that eliminates the need for the CLI `/login` command.
