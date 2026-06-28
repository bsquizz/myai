# myai: My personal collection of servives for running local AI workflows.

A Docker-based collection of services for running local AI workflows.

I run this at home on a machine that has an AMD Radeon 7900XTX with 24GB of VRAM.

You'll be able to use locally hosted models to chat with Open WebUI, work with opencode, and run other agentic workflows with hermes.

In addition, when web searches are used by the agents, they are configured to use a locally hosted searxng instance.

## Services

| Service | Port | Description |
|---|---|---|
| llama.cpp server | 8080 | Local LLM inference backend |
| Open WebUI | 3000 | Chat interface |
| SearXNG | 9000 | Private, metasearch engine for web search tool calls |

## Getting Started

```bash
docker compose up -d
```

Services will be available at:

- **LLM API:** `http://localhost:8080`
- **Web UI:** `http://localhost:3000`
- **Search:** `http://localhost:9000`

## Models

Models are configured via `models.ini` and autoloaded by the llama.cpp server. The current setup includes:

| Model | Alias | Context | Quantization |
|---|---|---|---|
| Qwen3.6-27B | `qwen3.6-27b` | 120k tokens | Q4_K_XL |
| Qwen3.6-35B-A3B (MoE) | `qwen3.6-35b-a3b` | 250k tokens | Q4_K_XL |
| Gemma 4-26B-A4B (MoE) | `gemma-4-26b-a4b` | 250k tokens | Q4_K_XL |
| Gemma 4-31B | `gemma-4-31b` | 40k tokens | Q4_K_XL |

The context was determined after some experimentation to stay within 24GB VRAM limits, leaving ~2.5-3.5GB for desktop overhead.

Models will be downloaded into `models/` by llama-server using the Hugging Face API. After the initial download, they will persist in the 'models/' directory. Only one model is loaded at a time by llama-server (`--models-max 1`), but they are "hot swappable" and do not require a server restart to change to a different one.

## Integrations

### Modifying SearXNG settings

SearXNG settings are stored in `data/searxng-core-config/settings.yml`. After the first `docker compose up`, edit this file directly to customize engines, search behavior, and other options. Then, restart the container.

#### Recommended settings.yml

```yaml
use_default_settings:
  engines:
    keep_only:
      - google
      - duckduckgo
      - bing
      - brave

general:
  enable_metrics: true

server:
  limiter: true
  secret_key: search123
  bind_address: 0.0.0.0
  port: 8080

search:
  safe_search: 0
  formats:
    - html
    - json
```

### Open WebUI

To set up Open WebUI, navigate to http://localhost:3000 and set up an admin username/password. Then follow the below instructions.

#### llama.cpp integration

1. In Open WebUI, go to **Admin Panel > Settings > Connections > OpenAI API**
2. Enable the OpenAI API integration
3. Set the **API Base URL** to: `http://llama-server:8080/v1` with no authentication.
4. Check the box to enable **Direct Connections** 
5. Check the box to enable **Cache Base Model List**
6. Click **Save**
7. Navigate to **Models** on the left navigation. (you should now see 4 models)
8. Enable or disable your models as desired. For each model you want to enable, I'd recommend disabling thinking for a better chat experience:
    - Click "Edit" icon next to the model
    - Next to "Advanced Params" click "Show"
    - Add a custom parameter with name `chat_template_kwargs` and value `{"enable_thinking":false}` 

#### SearXNG integration

1. In Open WebUI, go to **Admin Panel > Settings > Web Search**
2. Set **Search Engine** to `SearXNG`
3. Set **SearXNG Query URL** to: `http://searxng-core:8080/search?q=<query>`
4. Configure **Result Count** to 10 and **Concurrent Requests** to 10
5. Check the box to enable **Bypass Embedding and Retrieval**, **Bypass Web Loader** and **Trust Proxy Environment**
6. Click **Save**

### Use with Opencode

This project includes an example `opencode.json` configuration that connects [opencode](https://github.com/opencode-ai/opencode) to the local llama.cpp server. Models defined in `models.ini` are available as providers with their respective context limits.

### Use with Hermes

#### Add models

Add the following config to `~/.hermes/config.yaml`:

```
custom_providers:
  - name: local
    base_url: http://localhost:8080/v1
    api_key: none
    format: openai
```

Then type `hermes model` and scroll down until you see the `local` provider. You should see the 4 models running on llama.cpp show up here. Select one of them.

#### Use searxng for web search tool calls

1. Add `SEARXNG_URL=http://localhost:9000` to the bottom of `~/.hermes/.env`
2. Modify the 'web' section of `~/.hermes/config.yaml`:

    ```
    web:  
      backend: searxng
      search_backend: searxng
    ```
3. Run `hermes tools enable web`

You should now see web search tool calls utilize your local searxng instance.

