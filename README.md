# llm-chat

[![CI](https://github.com/Mattbusel/llm-chat/actions/workflows/ci.yml/badge.svg)](https://github.com/Mattbusel/llm-chat/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
![C++17](https://img.shields.io/badge/C%2B%2B-17-blue.svg)
![Single header](https://img.shields.io/badge/single-header-green.svg)

Multi-turn chat for C++: conversation history, token-budget trimming, streaming and save/restore in one header.

> Part of **[llm-cpp](https://github.com/Mattbusel/llm-cpp)**, a family of 26 single-header C++ libraries for building on LLM APIs. Each one stands alone: copy one header, include it, done.

A chat feature needs more than a single completion call. You have to keep the history, pin the system prompt, drop old turns before the context overflows, and ideally stream the reply. llm-chat's `Conversation` class does all of that on top of the OpenAI chat completions API.

## Features

- `Conversation` keeps the full message history and sends it with each turn
- System prompt pinned at the top and never trimmed
- Token budget (default 4096, estimated as characters / 4): oldest non-system messages are dropped when over
- `chat_stream()` delivers text deltas to a callback as they arrive (server-sent events) and returns the full reply
- `serialize()` / `deserialize()` to save and restore history as JSONL
- `reset()` clears the conversation but keeps the system prompt

## Quick start

Requirements: a C++17 compiler and libcurl (`apt install libcurl4-openssl-dev`, `brew install curl`, or `vcpkg install curl`).

1. Copy [`include/llm_chat.hpp`](include/llm_chat.hpp) into your project.
2. In exactly one `.cpp` file, `#define LLM_CHAT_IMPLEMENTATION` before including it. Other files just `#include "llm_chat.hpp"`.

```cpp
#define LLM_CHAT_IMPLEMENTATION
#include "llm_chat.hpp"
#include <cstdlib>
#include <iostream>

int main() {
    llm::ChatConfig cfg;
    cfg.api_key       = std::getenv("OPENAI_API_KEY");
    cfg.system_prompt = "You are a concise assistant.";
    cfg.token_budget  = 4096;  // oldest turns are dropped past this; the system prompt stays

    llm::Conversation convo(cfg);
    std::cout << convo.chat("My name is Ada. Remember it.") << "\n";

    // Streaming: text arrives through the callback as it is generated
    convo.chat_stream("What is my name?", [](const std::string& delta) {
        std::cout << delta << std::flush;
    });
    std::cout << "\n" << convo.message_count() << " messages\n";

    std::string saved = convo.serialize();  // JSONL; restore later with deserialize(saved)
}
```

Build and run:

```bash
g++ -std=c++17 -I include example.cpp -o example -lcurl
export OPENAI_API_KEY=sk-...
./example
```

## API

Everything lives in namespace `llm`.

| Function / type | What it does |
|---|---|
| `Conversation(ChatConfig)` | Create a conversation with API key, model (default `gpt-4o-mini`), system prompt, max tokens, temperature, token budget |
| `chat(msg)` | Send a user message, return the assistant reply |
| `chat_stream(msg, on_delta)` | Same, streaming each delta to a callback |
| `history()`, `message_count()`, `reset()` | Inspect or clear history |
| `serialize()` / `deserialize(jsonl)` | Persist history as JSONL |

## How it works

Each call appends the user message, trims the history to the token budget, POSTs the full message list to `/v1/chat/completions` through libcurl, and appends the assistant reply. Streaming sets `"stream": true` and parses the SSE `data:` lines as they arrive.

## Examples

The [`examples/`](examples) folder has runnable programs:

- [`basic_chat.cpp`](examples/basic_chat.cpp)
- [`persistent_chat.cpp`](examples/persistent_chat.cpp)
- [`token_budget.cpp`](examples/token_budget.cpp)

Build the examples with CMake (needs libcurl):

```bash
cmake -B build
cmake --build build
```

Examples that call the API read `OPENAI_API_KEY` from the environment.

## Limitations

- Token counts are a characters/4 estimate, not a real tokenizer.
- OpenAI chat completions endpoint only; no retries.

## License

MIT. See [LICENSE](LICENSE).
