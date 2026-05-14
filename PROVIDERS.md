# Provider Configuration Guide

Runnly-agent supports any OpenAI-compatible LLM provider. No OpenAI account required!

**Quick Start**: See [example-config.toml](./example-config.toml) for ready-to-use configuration examples for all providers.

## Configuration Hierarchy

Runnly-agent looks for configuration in:
1. Command-line arguments: `--model gpt-4`
2. Environment variables: `OPENAI_API_KEY`
3. Config file: `~/.codex/config.toml`
4. Built-in defaults

## Provider Examples

### DeepSeek

DeepSeek is a fast, cost-effective provider with strong reasoning capabilities. **Built-in support - no config.toml needed!**

**Using environment variable** (recommended):
```bash
export DEEPSEEK_API_KEY=your-key-here
cargo run --release -p codex-cli -- --model-provider deepseek --model deepseek-v4-flash
```

**Available models**:
- `deepseek-v4-flash` - Fast and economical
- `deepseek-v4-pro` - More capable reasoning
- `deepseek-chat` (deprecated July 24, 2026)
- `deepseek-reasoner` (deprecated July 24, 2026)

**Using config file**:
```toml
# ~/.codex/config.toml
model_provider = "deepseek"
model = "deepseek-v4-flash"
```

Get your API key from [DeepSeek Platform](https://platform.deepseek.com/api_keys).

### OpenAI

**Using environment variable** (recommended):
```bash
export OPENAI_API_KEY=sk-your-key-here
cargo run --release -p codex-cli
```

**Using config file**:
```toml
# ~/.codex/config.toml
model_provider = "openai"
model = "gpt-4o"
```

### Anthropic Claude

```bash
export ANTHROPIC_API_KEY=sk-ant-your-key-here
```

```toml
# ~/.codex/config.toml
model_provider = "anthropic"
model = "claude-3-5-sonnet-20241022"

[model_providers.anthropic]
name = "Anthropic"
base_url = "https://api.anthropic.com/v1"
env_key = "ANTHROPIC_API_KEY"
wire_api = "responses"
```

### Azure OpenAI

```bash
export AZURE_OPENAI_API_KEY=your-azure-key
```

```toml
# ~/.codex/config.toml
model_provider = "azure"
model = "gpt-4"

[model_providers.azure]
name = "Azure OpenAI"
base_url = "https://your-resource.openai.azure.com/openai/deployments/your-deployment"
env_key = "AZURE_OPENAI_API_KEY"
wire_api = "responses"

[model_providers.azure.http_headers]
api-version = "2024-02-15-preview"
```

### Ollama (Local Models)

```toml
# ~/.codex/config.toml
model_provider = "ollama"
model = "llama3:latest"

[model_providers.ollama]
name = "Ollama"
base_url = "http://localhost:11434/v1"
wire_api = "responses"
```

No API key needed for local Ollama!

### LM Studio (Local Models)

```toml
# ~/.codex/config.toml
model_provider = "lmstudio"
model = "local-model"

[model_providers.lmstudio]
name = "LM Studio"
base_url = "http://localhost:1234/v1"
wire_api = "responses"
```

### Custom OpenAI-Compatible Provider

```bash
export MY_LLM_API_KEY=your-key-here
```

```toml
# ~/.codex/config.toml
model_provider = "custom"
model = "your-model-name"

[model_providers.custom]
name = "My Custom LLM"
base_url = "https://api.your-provider.com/v1"
env_key = "MY_LLM_API_KEY"
wire_api = "responses"
```

### Groq

```bash
export GROQ_API_KEY=gsk_your-key-here
```

```toml
# ~/.codex/config.toml
model_provider = "groq"
model = "llama-3.1-70b-versatile"

[model_providers.groq]
name = "Groq"
base_url = "https://api.groq.com/openai/v1"
env_key = "GROQ_API_KEY"
wire_api = "responses"
```

### Together AI

```bash
export TOGETHER_API_KEY=your-key-here
```

```toml
# ~/.codex/config.toml
model_provider = "together"
model = "meta-llama/Meta-Llama-3.1-70B-Instruct-Turbo"

[model_providers.together]
name = "Together AI"
base_url = "https://api.together.xyz/v1"
env_key = "TOGETHER_API_KEY"
wire_api = "responses"
```

## Provider Requirements

For a provider to work with Runnly-agent, it must:

1. **Support OpenAI-compatible API format**
   - REST API at `/v1/chat/completions` or `/v1/responses`
   - Streaming responses (SSE)
   - Function calling (for tool usage)

2. **Authentication**
   - Bearer token via `Authorization: Bearer <token>` header
   - Set via environment variable defined in `env_key`

3. **Response Format**
   - Follow OpenAI's chat completion response format
   - Support streaming with `data:` prefix

## Advanced Configuration

### Multiple Headers

```toml
[model_providers.custom.http_headers]
X-Custom-Header = "value"
X-Another-Header = "another-value"
```

### Environment-Based Headers

```toml
[model_providers.custom.env_http_headers]
X-Api-Key = "MY_SECRET_KEY_ENV_VAR"
```

### Retry Configuration

```toml
[model_providers.custom]
request_max_retries = 3
stream_max_retries = 5
stream_idle_timeout_ms = 300000
```

### Command-Based Authentication

For providers requiring dynamic token generation:

```toml
[model_providers.custom.auth]
command = "aws codewhisperer get-token --output json"
json_path = "$.token"
```

## Troubleshooting

### "Authentication required" Error

Make sure your API key is set:
```bash
echo $OPENAI_API_KEY
```

If empty, set it:
```bash
export OPENAI_API_KEY=your-key-here
```

### "Provider not found" Error

Check your `model_provider` value matches a defined provider in `config.toml`.

### "Model not supported" Error

Verify the model name is valid for your provider:
```bash
# Test with curl
curl https://api.your-provider.com/v1/models \
  -H "Authorization: Bearer $YOUR_API_KEY"
```

### Connection Errors

Check the `base_url` is correct and accessible:
```bash
curl https://api.your-provider.com/v1/chat/completions \
  -H "Authorization: Bearer $YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "your-model", "messages": [{"role": "user", "content": "test"}]}'
```

## Next Steps

- See [config.md](./docs/config.md) for full configuration reference
- See [README.md](./README.md) for general usage
- Report issues: [GitHub Issues](https://github.com/[your-username]/runnly-agent/issues)
