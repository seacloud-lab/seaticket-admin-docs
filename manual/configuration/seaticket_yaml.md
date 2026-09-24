# seaticket_config.yaml

Besides [`.env`](./environment_variables.md) for general settings, the `seaticket_config.yaml` is used to specify the detailed configurations of the all components in SeaTicket docker.

## File structure of seaticket_config.yaml

`seaticket_config.yaml` consists of the following sections (i.e., `global` and name of each component):

```yaml
global:
  # ... configurations for all components
seaqa-ai:
  # ... special configurations for seaqa-ai
seaqa-events:
  # ... special configurations for seaqa-events
seaqa-indexer:
  # ... special configurations for seaqa-indexer
seaqa-notification:
  # ... special configurations for seaqa-notification
seaqa-web:
  # ... special configurations for seaqa-web
```

## Global settings

The global section is generally used to specify configuration items that are common to all SeaTickets components.

### Internal server URLs used for communication between SeaTicket components (cluster only)

In cluster mode, you need to specify the URLs used for internal communication between components via `seaticket_config.yaml`:

```yaml
global:
  # ... other configurations
  SEATICKET_AI_INNER_SERVER_URL: http://<your seaqa-ai server host>:8887
  SEATICKET_INDEXER_INNER_SERVER_URL: http://<your seaqa-indexer server host>:8888
  SEATICKET_EVENTS_INNER_SERVER_URL: http://<your seaqa-events server host>:6001
```

### LLMs configurations

!!! warning "To ensure the SeaTicket service functions normally, please ensure that `seaticket_config.yaml` contains **at least one valid LLM**"

For LLMs configurations, you should configure a list option `LLM_MODELS` in `seaticket_config.yaml`:

```yaml
global:
  LLM_MODELS:
    # <model 1 configuration>
    - type: 'your_llm_provider' # required, please refer https://docs.litellm.ai/docs/providers for the details
      model: 'your_model_id' # required
      url: 'llm_endpoint' # required for some provider (or type)
      key: 'llm_secret_key' # required for non-self deploy LLM server (including OpenAI proxy)
      label: 'display_name' # optional, display name on SeaTicket chat page, default to option `model` if not set
      hidden: false # optional, hidden on SeaTicket chat page but still can be used with API calling and other components, default to `false`
      disable: false # optional, disable this LLM, default to `false`
      tier: # optional, accepts one of three values — `low`, `medium`, or `high` — and is used to specify the model's inference strength, allowing for flexible invocation across different tasks.
      price: # required only need to specify this when you need to track user token usage.
        input_tokens: 1 # dollars / 1 M tokens (non-cached)
        output_tokens: 10 # dollars / 1 M tokens

    # <model 2 configuration>
    # - type: ...

    # <model 3 configuration>
    # - type : ...
```

!!! note "Futher information about `tier` of a model"
    - When multiple LLMs have the same `tier`, the system will use only the LLM that explicitly specifies that `tier`, with priority given to the first entry in the `LLM_MODELS` list.
    - If the tier specified for a service in [AI_UTILS_TIER](#seaqa-ai-settings) is not defined in `LLM_MODELS`, then the default model in `LLM_MODELS` or the first valid model in the list will be used.

#### LLMs reference configuration

=== "OpenAI"

    ```yaml
    global:
    # ...
    LLM_MODELS:
      - type: openai
        model: <your model-id>  # e.g., gpt-4o-mini
        key: <your openai LLM access key>
    ```

=== "Gemini"

    ```yaml
    global:
    # ...
    LLM_MODELS:
      - type: gemini
        model: <your model-id>  # e.g., gemini-2.5-flash
        key: <your gemini LLM access key>
    ```

=== "Qwen"

    ```yaml
    global:
    # ...
    LLM_MODELS:
      - type: dashscope
        model: <your model-id>  # e.g., qwen3-max
        key: <your qwen LLM access key>
    ```

=== "Deepseek"

    ```yaml
    global:
    # ...
    LLM_MODELS:
      - type: deepseek
        model: <model_id> # e.g. deepseek-chat
        key: <your LLM access key>
    ```

=== "Azure OpenAI"

    ```yaml
    global:
    # ...
    LLM_MODELS:
      - type: azure
        model: <your deployment name>
        url: # your deployment url, leave blank to use default endpoint
        key: <your LLM access key>
    ```

=== "Ollama"

    ```yaml
    global:
    # ...
    LLM_MODELS:
      - type: ollama
        model: <your model-id>
        url: <your LLM endpoint>
        key: <your LLM access key>
    ```

=== "HuggingFace"

    ```yaml
    global:
    # ...
    LLM_MODELS:
      - type: huggingface
        model: <model provider>/<model-id>
        url: <your huggingface API endpoint>
        key: <your huggingface API key>
    ```
=== "Other"
    SeaTicket AI utilizes [LiteLLM](https://docs.litellm.ai/docs/) to interact with LLM services. For a complete list of supported LLM providers, please refer to [this documentation](https://docs.litellm.ai/docs/providers). Then fill the following fields in your `seaticket_config.yaml`:

    For example, if you are using a LLM service with ***OpenAI-compatible endpoints***, you should set `type` to `other`, and set other LLM configuration items accurately.

### Embedding model

!!! warning "To ensure the SeaTicket service functions normally, please ensure that `seaticket_config.yaml` has specified embedding model"

The configuration of embedding models is similar to that of an element in the `LLM_MODELS` list:

```yaml
global:
  EMBEDDING_MODEL:
    type: ...
    model: embedding_model_id
    url: embedding_model_endpoint # required for some provider (or type)
    key: embedding_model_secret_key # required for non-self deployment LLM server (including OpenAI-proxy)
    price: 0.02 # dollars / 1 M input tokens, required only need to specify this when you need to track user token usage.
```

### Thought process

The thought process is used to display detailed debug information about LLM calls, including system prompts, context, tool call history, and cost statistics. It will be displayed in the project's Chat and Agent modules.

```yml
global:
  THOUGHT_PROCESS: # Configure which content to record in the thought process.
    enabled: false # set to `true` to record in the thought process (global switcher)
    tasks:
      enabled: true
      system_prompt: false
      user_input: true
      user_input_raw: false
    context: false
    tool_details: false
```

### Thirdparty integrations

Please refer [here](../integrations/overview.md) for the details.

## seaqa-ai settings

The following are the default values, normally, you don't need to make any additional settings for this part

```yaml
seaqa-ai:
  COMPLETION_MAX_RETRIES: 2 # Maximum number of chating completion retries with LLM
  COMPLETION_RETRY_INTERVAL: 0.0 # Retry interval when AI chating completion fails
  TOOL_CALL_MAX_RETRIES: 2 # Maximum number of executing tool retries with LLM
  TOOL_CALL_RETRY_INTERVAL: 1.0 # Retry interval when tool executes fails
  AI_UTILS_TIER: # The tier level required to configure AI functionality (must be one of `low`, `medium`, and `high`) will be matched in `LLM_MODELS`.
    generate_ticket: low
    generate_summary: low
    rerank_relevance: medium
```

!!! tip "Model selection"
    - For the chat and agent, the models typically used include:
        - Main resonance chain:
            - Chat: The model selected for the webpage (`LLM_MODELS.hidden = false`)
            - Agent: The defualt model in `LLM_MODELS`
        - An embedding model for vector search
        - A model specified by `AI_UTILS_TIER.rerank_relevance` for reranking search results
    - For other services (e.g., summary generation), will use the model specified in `AI_UTILS_TIER`

### Agent configurations

```yaml
seaqa-ai:
  AGENT:
    # --- Max records counter ---
    max_tickets_per_run: 100 # Maximum tickets analyzed per agent run
    max_github_issues_per_run: 100 # Maximum GitHub issues analyzed per agent run
    max_analysis_steps: 10 # Maximum analysis steps per agent task
    max_handling_steps: 5 # Maximum handling steps per agent task

    # --- Agent manager timing (seconds) ---
    enqueue_pending_events_check_intetval: 5 # Interval to check and flush pending events
    enqueue_debounce_interval: 600 # Minimum seconds between two enqueues for the same record
    enqueue_ticket_initial_debounce: 300 # Initial debounce seconds after first ticket event
    timed_scan_interval: 3600 # Timed scanner execution interval
    timed_scan_project_cooldown: 86400 # Cooldown before re-scanning the same project
```

## seaqa-events settings

```yaml
seaqa-events:
  AI_STATS: # optional
    enabled: true # Enable AI tokens usage statistics (default is `true`)
```

## seaqa-web notification settings

The Docker examples set this value through `.env`.

```yaml
seaqa-web:
  ENABLE_NOTIFICATION_SERVER: false
```

## Portal domain settings

Set `PORTAL_SERVICE_ROOT_DOMAIN` to provide each project with a portal subdomain, such as `example.seaticket-portal.ai`:

```yaml
seaqa-web:
  PORTAL_SERVICE_ROOT_DOMAIN: seaticket-portal.ai
```

The root domain must not include a scheme, port, wildcard, or path. Public Portal domains must be routed to a separate Web instance running with `SEAQA_APP_MODE=portal`. DNS, service topology, reverse proxy, TLS, and custom domain configuration are described in [Portal Domain](./portal_domain.md).
