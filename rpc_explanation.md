# Understanding Pi's RPC Mode (`--mode rpc`)

Pi provides a Remote Procedure Call (RPC) mode that allows it to operate headlessly via a JSON protocol over `stdin` and `stdout`. This mode is designed for embedding the agent into other applications, IDEs, or custom UIs.

## Starting RPC Mode

To start the agent in RPC mode, use the following command:

```bash
pi --mode rpc [options]
```

**Common Options:**
*   `--provider <name>`: Set the LLM provider (e.g., anthropic, openai, google).
*   `--model <pattern>`: Model pattern or ID (e.g., `anthropic/claude-sonnet-4-20250514`).
*   `--no-session`: Disable session persistence.
*   `--session-dir <path>`: Custom session storage directory.

---

## Protocol Overview

The protocol operates using single-line JSON objects:
1.  **Commands**: JSON objects sent to `stdin`, one per line.
2.  **Responses**: JSON objects with `type: "response"` indicating command success/failure sent to `stdout`.
3.  **Events**: Agent events streamed to `stdout` as JSON lines.

All commands support an optional `id` field for request/response correlation.

---

## Key Commands

### Prompting

*   **`prompt`**: Send a user prompt to the agent.
    ```json
    {"id": "req-1", "type": "prompt", "message": "Hello, world!"}
    ```
    If the agent is already streaming, specify `streamingBehavior` (`"steer"` to interrupt, `"followUp"` to queue).

*   **`steer`**: Interrupt the agent mid-run. Delivered after current tool execution.
    ```json
    {"type": "steer", "message": "Stop and do this instead"}
    ```

*   **`follow_up`**: Queue a message to be processed after the agent finishes.
    ```json
    {"type": "follow_up", "message": "After you're done, also do this"}
    ```

*   **`abort`**: Abort the current agent operation.
    ```json
    {"type": "abort"}
    ```

### State and Configuration

*   **`get_state`**: Retrieve the current session state, including model, thinking level, and configuration.
*   **`get_messages`**: Get all messages in the conversation.
*   **`set_model`**: Switch to a specific model.
    ```json
    {"type": "set_model", "provider": "anthropic", "modelId": "claude-sonnet-4-20250514"}
    ```
*   **`set_thinking_level`**: Set reasoning level (`"off"`, `"low"`, `"medium"`, `"high"`, etc.).

---

## Events Streaming

Events are streamed to `stdout` during agent operation. They do not include an `id` field.

### Event Lifecycle

1.  **`agent_start`**: Agent begins processing.
2.  **`turn_start` / `turn_end`**: A turn (assistant response + tool calls/results) begins/ends.
3.  **`message_start` / `message_end`**: Message block begins/ends.
4.  **`message_update`**: Streaming updates for text, thinking, or tool call deltas.
    ```json
    {
      "type": "message_update",
      "assistantMessageEvent": {
        "type": "text_delta",
        "delta": "Hello"
      }
    }
    ```
5.  **`tool_execution_start` / `tool_execution_update` / `tool_execution_end`**: Tracks tool usage and partial results (like bash output).
6.  **`agent_end`**: Agent completes and returns all generated messages.

---

## Extension UI Protocol

Extensions can request user interaction via a sub-protocol.

*   **Dialog Methods** (`select`, `confirm`, `input`, `editor`): Emits an `extension_ui_request` on `stdout`. Blocks until the client replies with an `extension_ui_response` on `stdin`.
    ```json
    // Request (stdout)
    {"type": "extension_ui_request", "id": "uuid-1", "method": "confirm", "title": "Clear session?"}

    // Response (stdin)
    {"type": "extension_ui_response", "id": "uuid-1", "confirmed": true}
    ```
*   **Fire-and-forget Methods** (`notify`, `setStatus`, `setWidget`): Emits information to `stdout` without expecting a response.

---

## Example Usage (Node.js)

```javascript
const { spawn } = require("child_process");
const readline = require("readline");

const agent = spawn("pi", ["--mode", "rpc", "--no-session"]);

readline.createInterface({ input: agent.stdout }).on("line", (line) => {
    const event = JSON.parse(line);

    // Stream text deltas
    if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
        process.stdout.write(event.assistantMessageEvent.delta);
    }
});

// Send a prompt
agent.stdin.write(JSON.stringify({ type: "prompt", message: "Hello" }) + "\n");
```

*(For Node.js/TypeScript integrations, it is highly recommended to use the `AgentSession` or `RpcClient` classes from `@mariozechner/pi-coding-agent` instead of spawning a raw subprocess).*
