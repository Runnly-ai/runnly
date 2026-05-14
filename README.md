# Runnly-agent

<p align="center"><strong>Runnly-agent</strong> is an AI coding agent that runs locally on your computer and works with OpenAI-compatible LLM models.</p>

<p align="center">A fork of OpenAI's Codex, designed to support multiple OpenAI-compatible LLM providers including OpenAI, Azure OpenAI, Anthropic Claude, and other compatible APIs.</p>

---

## Features

- 🤖 **Multi-Model Support**: Works with any OpenAI-compatible API
- 💻 **Local Execution**: Runs entirely on your machine
- 🔧 **Extensible**: Built on the robust Codex architecture
- 🔐 **Flexible Authentication**: Support for API keys from multiple providers

## Quickstart

### Installing and running Runnly-agent

#### Building from source:

```shell
# Clone the repository
git clone the project
cd runnly-agent

# Build the project
cd codex-rs
cargo build --release

# Run the agent
cargo run --release -p codex-cli
# Or run the binary directly after building:
# ./codex-rs/target/release/codex
```

#### Pre-built binaries:

<details>
<summary>Download from the <a href="https://github.com/[your-username]/runnly-agent/releases/latest">latest GitHub Release</a> (when available)</summary>

Pre-built binaries will be available for:

- macOS (Apple Silicon/arm64 and x86_64)
- Linux (x86_64 and arm64)
- Windows

Extract the archive and run the binary directly.

</details>

### Configuration

Runnly-agent works with any OpenAI-compatible LLM provider using API keys. **No OpenAI account required!**

#### Quick Start with Environment Variables

Set your API key as an environment variable:

```shell
# For OpenAI
export OPENAI_API_KEY=your-api-key-here

# For DeepSeek (recommended - fast and cost-effective)
export DEEPSEEK_API_KEY=your-api-key-here

# For other providers (see examples below)
export CUSTOM_API_KEY=your-api-key-here
```

Then run:
```shell
cd codex-rs
cargo run --release -p codex-cli
```

#### Supported Providers

**DeepSeek** (fast, cost-effective, built-in):
```shell
export DEEPSEEK_API_KEY=your-api-key
cargo run --release -p codex-cli -- --model-provider deepseek --model deepseek-v4-flash
```

Or create `~/.codex/config.toml`:
```toml
model_provider = "deepseek"
model = "deepseek-v4-flash"
```

See [deepseek-config.toml](./deepseek-config.toml) for a complete example.

Available models:
- `deepseek-v4-flash` - Fast and cost-effective
- `deepseek-v4-pro` - More capable reasoning

**OpenAI** (default):
```shell
export OPENAI_API_KEY=sk-...
cargo run --release -p codex-cli
```

**Anthropic Claude** (via OpenAI-compatible endpoint):
```toml
# ~/.codex/config.toml
model_provider = "anthropic"
model = "claude-3-5-sonnet-20241022"

[model_providers.anthropic]
name = "Anthropic"
base_url = "https://api.anthropic.com/v1"
env_key = "ANTHROPIC_API_KEY"
```

**Azure OpenAI**:
```toml
# ~/.codex/config.toml
model_provider = "azure"

[model_providers.azure]
name = "Azure OpenAI"
base_url = "https://your-resource.openai.azure.com/openai/deployments/your-deployment"
env_key = "AZURE_OPENAI_API_KEY"
```

**Ollama** (local models):
```toml
# ~/.codex/config.toml
model_provider = "ollama"
model = "llama3:latest"

[model_providers.ollama]
name = "Ollama"
base_url = "http://localhost:11434/v1"
```

**Any OpenAI-compatible endpoint**:
```toml
# ~/.codex/config.toml
model_provider = "custom"
model = "your-model-name"

[model_providers.custom]
name = "My Custom LLM"
base_url = "https://api.your-llm.com/v1"
env_key = "YOUR_API_KEY_ENV_VAR"
```

**See [example-config.toml](./example-config.toml) for a complete configuration reference with all providers.**

For more details, see the [config documentation](./docs/config.md).

## Documentation

- [**Provider Configuration Guide**](./PROVIDERS.md) - Configure different LLM providers
- [**Configuration Guide**](./docs/config.md)
- [**Contributing**](./docs/contributing.md)
- [**Building from Source**](./docs/install.md)
- [**Original Codex Documentation**](https://developers.openai.com/codex) (upstream reference)

## About

Runnly-agent is a fork of [OpenAI's Codex](https://github.com/openai/codex), extended to work with any OpenAI-compatible LLM API. This project maintains compatibility with the original Codex architecture while adding flexibility to use your preferred AI model provider.

To sync updates from the upstream OpenAI Codex repository, see the [contributing guide](./docs/contributing.md).

This repository is licensed under the [Apache-2.0 License](LICENSE).
