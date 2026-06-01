# chuks_ai

**Unified AI/LLM client for Chuks** — one package, every provider. Native streaming via channels, parallel inference with `spawn`, typed structured outputs, and function/tool calling.

- One API surface, six providers (OpenAI, Anthropic, Google Gemini, Mistral, Ollama, Groq).
- Streaming through `Channel<string>` — no async iterators, no callbacks.
- Real parallel inference with `spawn` (runs on real CPU cores, not a single thread).
- Strongly-typed result `dataType`s — no `any` soup at call sites.
- Built-in tool/function calling with auto-execution.

---

## Install

```bash
chuks add @chuks/ai
```

---

## Quick Start

```chuks
import { ai } from "pkg/@chuks/ai"

async function main(): Task<any> {
    var client = ai.openai("gpt-4o")
    var result = await client.complete("Explain async/await in Chuks")
    println(result.content)
}
```

---

## Exported Symbols

| Symbol             | Kind           | Purpose                                                   |
| ------------------ | -------------- | --------------------------------------------------------- |
| `ai`               | singleton `AI` | Entry point — `ai.openai(...)`, `ai.anthropic(...)`, etc. |
| `AI`               | class          | The facade type (rarely needed directly).                 |
| `OpenAIClient`     | class          | OpenAI client returned by `ai.openai(...)`.               |
| `AnthropicClient`  | class          | Anthropic client returned by `ai.anthropic(...)`.         |
| `GoogleClient`     | class          | Google Gemini client.                                     |
| `MistralClient`    | class          | Mistral client.                                           |
| `OllamaClient`     | class          | Local Ollama client.                                      |
| `GroqClient`       | class          | Groq client.                                              |
| `CompletionResult` | `dataType`     | Return shape of `.complete()` / `.completeWithTools()`.   |
| `EmbeddingResult`  | `dataType`     | Return shape of `.embed()`.                               |
| `ChatMessage`      | `dataType`     | One entry in the chat history array.                      |
| `ToolDefinition`   | `dataType`     | Schema descriptor for a registered tool.                  |
| `ToolCall`         | `dataType`     | A tool invocation requested by the model.                 |

Every provider client exposes the **same set of methods**. Swap providers by changing one line — `ai.openai(...)` → `ai.anthropic(...)` and your code keeps working.

---

## Exported Types

### `CompletionResult`

```chuks
dataType CompletionResult {
    content: string          // Response text
    model: string            // Model that generated the response
    usage: any               // Token usage: { prompt_tokens, completion_tokens, total_tokens }
    finishReason: string     // "stop" | "length" | "tool_calls" | "end_turn" | "content_filter"
    toolCalls: []ToolCall    // Empty if the model did not request tools
}
```

### `EmbeddingResult`

```chuks
dataType EmbeddingResult {
    embedding: []float       // The vector
    model: string            // Embedding model used
    dimensions: int          // Vector length
    usage: any               // Token usage (provider-specific)
}
```

### `ChatMessage`

```chuks
dataType ChatMessage {
    role: string             // "system" | "user" | "assistant" | "tool"
    content: string          // Message body
    name: string?            // Optional — function/tool name for role:"tool"
    tool_call_id: string?    // Required for role:"tool" — the id being replied to
    tool_calls: []ToolCall?  // Set on assistant turns that invoke tools
}
```

### `ToolDefinition`

```chuks
dataType ToolDefinition {
    name: string             // Unique tool name (matches what the model emits)
    description: string      // What the tool does (the LLM reads this)
    parameters: any          // JSON Schema describing the arguments
    handler: any             // fn(args: any) -> any — executed by completeWithTools
}
```

### `ToolCall`

```chuks
dataType ToolCall {
    id: string               // Provider-assigned call id
    name: string             // Tool name the model wants to invoke
    arguments: any           // Already-parsed JSON object (NOT a raw string)
}
```

---

## Provider Factories

All factories live on the `ai` singleton. The optional `config` argument is a plain object.

```chuks
ai.openai(model: string, config: any?): OpenAIClient
ai.anthropic(model: string, config: any?): AnthropicClient
ai.google(model: string, config: any?): GoogleClient
ai.mistral(model: string, config: any?): MistralClient
ai.ollama(model: string, config: any?): OllamaClient
ai.groq(model: string, config: any?): GroqClient
```

### Config shape (shared)

| Field          | Type   | Default          | Notes                                         |
| -------------- | ------ | ---------------- | --------------------------------------------- |
| `apiKey`       | string | env var          | Loaded from env if omitted (see table below). |
| `baseUrl`      | string | provider default | For proxies / Azure / self-hosted gateways.   |
| `temperature`  | float  | provider default | Sampling temperature.                         |
| `maxTokens`    | int    | provider default | Max tokens in the reply.                      |
| `systemPrompt` | string | `""`             | Default system message.                       |

Provider-specific extras:

| Provider  | Extra Config                                           |
| --------- | ------------------------------------------------------ |
| Anthropic | `apiVersion` (default `"2023-06-01"`)                  |
| Ollama    | No `apiKey` needed — talks to `http://localhost:11434` |

### Environment variables

| Provider  | Variable            |
| --------- | ------------------- |
| OpenAI    | `OPENAI_API_KEY`    |
| Anthropic | `ANTHROPIC_API_KEY` |
| Google    | `GOOGLE_API_KEY`    |
| Mistral   | `MISTRAL_API_KEY`   |
| Groq      | `GROQ_API_KEY`      |
| Ollama    | _(none — local)_    |

---

## Client API (every provider)

| Method              | Signature                                                                   | Description                                                  |
| ------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `complete`          | `complete(prompt: string, options: any?): Task<CompletionResult>`           | Single completion.                                           |
| `stream`            | `stream(prompt: string, ch: Channel<string>, options: any?): Task<any>`     | Streams tokens into `ch`; closes the channel when done.      |
| `structured`        | `structured(prompt: string, schema: any, options: any?): Task<any>`         | Returns parsed JSON matching `schema`.                       |
| `embed`             | `embed(input: string, model: string?): Task<EmbeddingResult>`               | One embedding.                                               |
| `embedBatch`        | `embedBatch(inputs: []string, model: string?): Task<[]EmbeddingResult>`     | Many embeddings in one request.                              |
| `registerTool`      | `registerTool(name: string, desc: string, params: any, handler: any): void` | Make a tool callable from the model.                         |
| `completeWithTools` | `completeWithTools(prompt: string, options: any?): Task<CompletionResult>`  | Auto-executes any tool calls and continues the conversation. |
| `system`            | `system(prompt: string): self`                                              | Fluent — set system prompt.                                  |
| `temperature`       | `temperature(t: float): self`                                               | Fluent — sampling temperature.                               |
| `maxTokens`         | `maxTokens(n: int): self`                                                   | Fluent — max reply tokens.                                   |
| `enableHistory`     | `enableHistory(maxPairs: int?): self`                                       | Track conversation history.                                  |
| `clearHistory`      | `clearHistory(): self`                                                      | Reset history.                                               |

The fluent setters return the client itself, so they chain:

```chuks
var client = ai.openai("gpt-4o")
    .system("You are a helpful Chuks programming assistant.")
    .temperature(0.3)
    .maxTokens(2048)
```

### Ollama-only methods

```chuks
ollama.list(): Task<[]string>       // installed models
ollama.pull(name: string): Task<any> // download a model
```

---

## Recipes

### Completion

```chuks
var client = ai.anthropic("claude-sonnet-4-20250514")
var result = await client.complete("What is Chuks?")
println(result.content)       // response text
println(result.model)         // "claude-sonnet-4-20250514"
println(result.finishReason)  // "end_turn"
println(result.usage)         // { input_tokens, output_tokens, ... }
```

### Streaming via channels

```chuks
import { ai } from "pkg/@chuks/ai"
import { Channel } from "std/channel"

async function main(): Task<any> {
    var client = ai.openai("gpt-4o")
    var ch: Channel<string> = new Channel<string>(0)
    spawn client.stream("Write a poem about Chuks", ch, null)

    for (var token: string? = ch.receive(); token != null; token = ch.receive()) {
        print(token)
    }
    println("")
}
```

### Parallel inference with `spawn`

```chuks
async function main(): Task<any> {
    var client = ai.anthropic("claude-sonnet-4-20250514")
    var prompts: []string = [
        "Explain variables in Chuks",
        "Explain functions in Chuks",
        "Explain classes in Chuks",
        "Explain async/await in Chuks",
    ]

    var tasks: []Task<CompletionResult> = []
    for (var p of prompts) { tasks.push(spawn client.complete(p, null)) }

    for (var t of tasks) {
        var r: CompletionResult = await t
        println("---")
        println(r.content)
    }
}
```

### Structured output

```chuks
var schema = {
    "name": "string",
    "age": "int",
    "skills": "[]string",
}

var profile = await client.structured(
    "Extract: John is 28 and knows Chuks and Go",
    schema,
    null
)
println(profile.name)    // "John"
println(profile.age)     // 28
println(profile.skills)  // ["Chuks", "Go"]
```

### Tool / function calling

```chuks
function getWeather(args: any): string {
    return "72°F and sunny in " + string(args.city)
}

async function main(): Task<any> {
    var client = ai.openai("gpt-4o")

    client.registerTool(
        "get_weather",
        "Get current weather for a city",
        {
            "type": "object",
            "properties": {
                "city":  { "type": "string", "description": "City name" },
                "units": { "type": "string", "enum": ["fahrenheit", "celsius"] },
            },
            "required": ["city"],
        },
        getWeather
    )

    var r = await client.completeWithTools("What's the weather in Lagos?", null)
    println(r.content)        // "The weather in Lagos is 72°F and sunny."
    println(r.toolCalls)      // []ToolCall — what was invoked along the way
}
```

### Embeddings

```chuks
var r = await client.embed("Chuks is a compiled language", null)
println(r.embedding.length)   // e.g. 1536
println(r.dimensions)         // 1536
```

### Conversation history

```chuks
var client = ai.openai("gpt-4o").enableHistory(20)
await client.complete("My name is Alice", null)
var r = await client.complete("What's my name?", null)
println(r.content)            // "Your name is Alice"
client.clearHistory()
```

### Local Ollama

```chuks
var client = ai.ollama("llama3")
var r = await client.complete("Explain concurrency in Chuks", null)
println(r.content)

var models = await client.list()
await client.pull("gemma:7b")
```

---

## Why chuks_ai

| Feature            | Node.js (Vercel AI SDK)       | **chuks_ai**             |
| ------------------ | ----------------------------- | ------------------------ |
| Parallel inference | `Promise.all` (single thread) | `spawn` (real CPU cores) |
| Streaming          | Async iterators / callbacks   | Typed `Channel<string>`  |
| Structured output  | Zod schemas (runtime)         | Native dataTypes         |
| Binary size        | Node.js runtime + deps        | Single native binary     |
| Cold start         | ~100 ms                       | ~1 ms                    |

---

## License

MIT
