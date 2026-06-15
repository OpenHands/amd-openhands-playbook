<!--
Copyright Advanced Micro Devices, Inc.

SPDX-License-Identifier: MIT
-->

<!-- @github-only -->
> [!IMPORTANT]
> This playbook uses AMD Playbooks comment tags that are interpreted by the
> AMD Playbooks site. GitHub renders the Markdown content, but not the device,
> OS, variable, or hidden-test directives.
<!-- @github-only:end -->

# Build a GitHub-to-Slack Development Digest with Agent Canvas and a Local LLM

## Overview

Software engineering is full of repetitive tasks that still require context,
judgment, and careful follow-through. Automations are useful whenever a team
repeats the same context-gathering, decision, and notification loop: checking
activity, summarizing changes, triaging issues, monitoring health signals, or
posting updates. They reduce manual polling, make handoffs more consistent, and
let important work run on a schedule or in response to events.

[OpenHands automations](https://docs.openhands.dev/openhands/usage/automations/overview)
run these workflows as full agent conversations with access to your configured
LLM, stored secrets, tools, and integrations.
[Agent Canvas](https://github.com/OpenHands/agent-canvas) provides a local,
self-hostable UI and backend stack for connecting an OpenHands agent server,
configuring the LLM, installing MCP servers, and creating or testing
automations.

In this playbook, you will build one concrete automation: a scheduled
GitHub-to-Slack development digest powered by Agent Canvas, GitHub and Slack MCP
servers, and a local model served by Lemonade Server. Because the model runs
locally through an OpenAI-compatible API, the workflow context and prompt stay
on your AMD system. SGLang is also covered as an optional local endpoint when
you need grammar-constrained OpenAI tool calling.

![Architecture diagram showing GitHub MCP, OpenHands automation, Lemonade Server, and Slack MCP](screenshots/00-architecture-overview.png)

<!-- @device:stx,krk -->
> [!NOTE]
> Coding-agent workflows benefit from a larger model and context window. Use at
> least 32 GB of system memory, and prefer 64 GB or more for larger GGUF models.
<!-- @device:end -->

## Prerequisites

<!-- @os:linux -->
<!-- @require:lemonade,nodejs -->
<!-- @os:end -->

<!-- @os:windows -->
<!-- @require:lemonade,nodejs -->
<!-- @os:end -->

You need:

- Lemonade Server installed.
- Agent Canvas checked out locally, using a recent `main` checkout.
- A recent OpenHands Agent Server / `software-agent-sdk` build with
  schema-driven agent settings, `LLMSummarizingCondenserSettings.max_tokens`,
  and LLM `custom_tokenizer` support.
- The Python `transformers` package available in the Agent Server environment.
  It is required for chat-template token counting when `custom_tokenizer` is
  set.
- A GitHub token with read access to the repository you want summarized.
- A Slack bot token (`xoxb-...`) with `chat:write` and channel read access.
- A Slack team ID (`T...`).
- A Slack channel ID (`C...`) where the digest should be posted.

Invite the Slack app to the target channel before testing the automation.

## Variables Used in This Playbook

<!-- @device:halo,halo_box,stx,krk -->
<!-- @var:id=lemonade_model value="Qwen3.6-35B-A3B-GGUF" -->
<!-- @device:end -->

```bash
export LEMONADE_BASE_URL="http://127.0.0.1:13305/api/v1"
export LEMONADE_MODEL="Qwen3.6-35B-A3B-GGUF"
export QWEN_CUSTOM_TOKENIZER="Qwen/Qwen3.6-35B-A3B"
export CONDENSER_MAX_TOKENS="56000"

export SGLANG_BASE_URL="http://127.0.0.1:13306/v1"
export SGLANG_MODEL="Qwen3-8B"

export AGENT_CANVAS_URL="http://localhost:8000"
export AGENT_SERVER_API_URL="http://127.0.0.1:18000/api"
export AUTOMATION_API_URL="http://127.0.0.1:18001/api/automation"
export AGENT_CANVAS_API_KEY="$(cat ~/.openhands/agent-canvas/api-key.txt)"

export GITHUB_REPO_FILTER="your-org/your-repo"
export SLACK_DIGEST_CHANNEL="C0123456789"
export DIGEST_TIMEZONE="America/New_York"
```

Use an explicit `owner/repo` value for `GITHUB_REPO_FILTER`. Broad organization
wildcards can return too much MCP context for local models.

Agent Canvas local development uses one session key for both local backends:
send it as `X-Session-API-Key` to Agent Server, and as
`Authorization: Bearer ...` to the automation backend. The key is stored at
`~/.openhands/agent-canvas/api-key.txt`.

## 1. Start Lemonade Server

Start the model from the Lemonade CLI:

```bash
lemonade config set llamacpp.backend=vulkan
lemonade config set ctx_size=65536
lemonade run "${LEMONADE_MODEL}"
```

Lemonade exposes an OpenAI-compatible API at:

```text
http://127.0.0.1:13305/api/v1
```

If you are developing Lemonade from a local source checkout and need to restart
the server in the background, run:

```bash
cd ~/work/lemonade
setsid -f build/lemond --host 127.0.0.1 --port 13305 \
  </dev/null > ~/.openhands/agent-canvas/lemonade.log 2>&1
```

Optional: if Agent Canvas or the automation runner is not on the same machine,
publish the Lemonade endpoint through a secure tunnel and use the HTTPS URL as
the LLM base URL:

```bash
ngrok http 13305 --url YOUR_NGROK_DOMAIN.ngrok-free.dev
```

<!-- @test:id=lemonade-version timeout=60 hidden=True -->
```bash
lemonade --version
```
<!-- @test:end -->

<!-- @test:id=lemonade-health-linux timeout=120 hidden=True -->
```bash
set -euo pipefail

for i in $(seq 1 120); do
  if curl -fsS "http://127.0.0.1:13305/api/v1/health" >/dev/null; then
    echo "OK: Lemonade Server is responding"
    exit 0
  fi
  sleep 1
done

echo "Lemonade Server did not become ready on http://127.0.0.1:13305"
exit 1
```
<!-- @test:end -->

## 2. Verify the Local Model

Confirm Lemonade can serve the selected model:

```bash
curl -s "${LEMONADE_BASE_URL}/models" | python3 -m json.tool
```

Then send a small chat request:

```bash
curl -sS "${LEMONADE_BASE_URL}/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"${LEMONADE_MODEL}"'",
    "messages": [
      {"role": "user", "content": "Reply with exactly: OK"}
    ],
    "temperature": 0,
    "max_tokens": 64
  }' | python3 -m json.tool
```

If this returns a `choices` array, Lemonade is ready for Agent Canvas.

## 3. Optional: Use SGLang for Tool Calls

SGLang is a useful fallback when the automation depends heavily on OpenAI
`tools`. Launch SGLang with an explicit grammar backend, Qwen reasoning parser,
Qwen tool-call parser, and strict tool decoding:

```bash
cd ~/work/sglang
source ~/.venvs/sglang-rocm/bin/activate

export SGLANG_TOOL_STRICT_LEVEL=2
export HIP_VISIBLE_DEVICES=0
export PYTHONPATH="$HOME/work/sglang/python:${PYTHONPATH:-}"

python -u -m sglang.launch_server \
  --model-path "Qwen/Qwen3-8B" \
  --host 127.0.0.1 \
  --port 13306 \
  --served-model-name "${SGLANG_MODEL}" \
  --dtype bfloat16 \
  --context-length 8192 \
  --max-total-tokens 8192 \
  --max-running-requests 1 \
  --mem-fraction-static 0.55 \
  --attention-backend torch_native \
  --grammar-backend xgrammar \
  --reasoning-parser qwen3 \
  --tool-call-parser qwen \
  --disable-cuda-graph \
  --disable-custom-all-reduce
```

Verify parsed OpenAI tool calls:

```bash
curl -fsS "${SGLANG_BASE_URL}/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen3-8B",
    "messages": [
      {
        "role": "user",
        "content": "What is the weather in Pittsburgh? Use the weather tool."
      }
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "get_weather",
          "description": "Get current weather for a city",
          "parameters": {
            "type": "object",
            "properties": {
              "city": { "type": "string" },
              "unit": {
                "type": "string",
                "enum": ["celsius", "fahrenheit"]
              }
            },
            "required": ["city"],
            "additionalProperties": false
          }
        }
      }
    ],
    "tool_choice": "required",
    "chat_template_kwargs": { "enable_thinking": false },
    "temperature": 0,
    "max_tokens": 96
  }' | python3 -m json.tool
```

The response should have `finish_reason: "tool_calls"` and a non-empty
`choices[0].message.tool_calls` array. The
`chat_template_kwargs.enable_thinking=false` setting is important for this
Qwen3-style checkpoint; without it, SGLang can return tagged tool-call text in
`reasoning_content` instead of parsed OpenAI `message.tool_calls`.

SGLang was not demonstrated with `qwen3.6-35b-a3b` on a 32 GB GPU. The 35B
Lemonade path uses `Qwen3.6-35B-A3B-GGUF`, while the verified SGLang fallback
was `Qwen/Qwen3-8B` served as `Qwen3-8B`. Attempts to run
`mattbucci/Qwen3.6-35B-A3B-AWQ` with AWQ quantization, a 4,096-token context,
and `--mem-fraction-static` between `0.40` and `0.50` failed during model
initialization with HIP out-of-memory errors on a 32 GB ROCm GPU.

## 4. Start Agent Canvas

From the Agent Canvas checkout:

```bash
cd ~/work/agent-canvas
npm ci
npm run dev
```

The default development launcher starts:

- Agent Server on port `18000`
- Automation backend on port `18001`
- Frontend on port `3001`
- Ingress proxy on port `8000`

Open Agent Canvas if you want to inspect the result visually:

```text
http://localhost:8000
```

The screenshots below show the equivalent frontend state, but the setup in this
playbook uses backend API calls.

![Agent Canvas onboarding screen with OpenHands selected](screenshots/01-onboarding-choose-agent.png)

![Agent Canvas onboarding backend connection success](screenshots/02-onboarding-backend-connected.png)

## 5. Configure Agent Server Through the API

First switch Agent Server to the OpenHands agent. This is a separate call so a
previous ACP configuration does not get deep-merged into the OpenHands settings
variant:

```bash
curl -sS -X PATCH "${AGENT_SERVER_API_URL}/settings" \
  -H "X-Session-API-Key: ${AGENT_CANVAS_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "agent_settings_diff": {
      "agent_kind": "openhands"
    }
  }' | python3 -m json.tool
```

Then configure Lemonade, the Qwen tokenizer, and the condenser:

```bash
curl -sS -X PATCH "${AGENT_SERVER_API_URL}/settings" \
  -H "X-Session-API-Key: ${AGENT_CANVAS_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "agent_settings_diff": {
      "llm": {
        "model": "openai/'"${LEMONADE_MODEL}"'",
        "base_url": "'"${LEMONADE_BASE_URL}"'",
        "api_key": "lemonade",
        "custom_tokenizer": "'"${QWEN_CUSTOM_TOKENIZER}"'",
        "litellm_extra_body": {
          "enable_thinking": true
        }
      },
      "condenser": {
        "enabled": true,
        "condenser_kind": "llm_summarizing",
        "max_tokens": '"${CONDENSER_MAX_TOKENS}"'
      }
    },
    "conversation_settings_diff": {
      "confirmation_mode": false,
      "max_iterations": 20
    },
    "active_profile": null
  }' | python3 -m json.tool
```

Verify the saved settings without exposing secrets:

```bash
curl -sS "${AGENT_SERVER_API_URL}/settings" \
  -H "X-Session-API-Key: ${AGENT_CANVAS_API_KEY}" \
  | python3 -m json.tool
```

The LLM profile should show:

| Field | Value |
| --- | --- |
| Model | `openai/Qwen3.6-35B-A3B-GGUF` |
| Base URL | `http://127.0.0.1:13305/api/v1` |
| Custom tokenizer | `Qwen/Qwen3.6-35B-A3B` |
| LiteLLM extra body | `{"enable_thinking":true}` |
| Condenser kind | `llm_summarizing` |
| Condenser max tokens | `56000` |

The `openai/` prefix tells LiteLLM to use OpenAI-compatible request formatting
against the Lemonade endpoint. The custom tokenizer is the original Hugging
Face tokenizer for the GGUF model; it lets OpenHands count the same
chat-template tokens that the local model server sees.

![Agent Canvas LLM profile configured for Lemonade](screenshots/03-llm-profile-configured.png)

![Agent Canvas home after onboarding](screenshots/05-agent-canvas-home.png)

## 6. Install GitHub and Slack MCP Servers

Configure MCP servers through `PATCH /api/settings`. The token values are sent
only to the local Agent Server and are persisted as encrypted settings.

```bash
export GITHUB_PERSONAL_ACCESS_TOKEN="YOUR_GITHUB_TOKEN"
export SLACK_BOT_TOKEN="xoxb-..."
export SLACK_TEAM_ID="T0123456789"
export SLACK_CHANNEL_IDS="${SLACK_DIGEST_CHANNEL}"

curl -sS -X PATCH "${AGENT_SERVER_API_URL}/settings" \
  -H "X-Session-API-Key: ${AGENT_CANVAS_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "agent_settings_diff": {
      "mcp_config": {
        "mcpServers": {
          "github": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-github"],
            "env": {
              "GITHUB_PERSONAL_ACCESS_TOKEN": "'"${GITHUB_PERSONAL_ACCESS_TOKEN}"'"
            }
          },
          "slack": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-slack"],
            "env": {
              "SLACK_BOT_TOKEN": "'"${SLACK_BOT_TOKEN}"'",
              "SLACK_TEAM_ID": "'"${SLACK_TEAM_ID}"'",
              "SLACK_CHANNEL_IDS": "'"${SLACK_CHANNEL_IDS}"'"
            }
          }
        }
      }
    }
  }' | python3 -m json.tool
```

Set `SLACK_CHANNEL_IDS` to the digest channel ID so the agent does not need to
page through every Slack channel.

You can test one MCP server before running the automation:

```bash
curl -sS -X POST "${AGENT_SERVER_API_URL}/mcp/test" \
  -H "X-Session-API-Key: ${AGENT_CANVAS_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "github",
    "timeout": 60,
    "server": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "'"${GITHUB_PERSONAL_ACCESS_TOKEN}"'"
      }
    }
  }' | python3 -m json.tool
```

![Agent Canvas MCP page with GitHub and Slack servers installed](screenshots/04-mcp-servers-installed.png)

## 7. Create the Digest Automation

Use the automation prompt preset endpoint directly:

```bash
CREATE_RESPONSE="$(
  curl -sS -X POST "${AUTOMATION_API_URL}/v1/preset/prompt" \
    -H "Authorization: Bearer ${AGENT_CANVAS_API_KEY}" \
    -H "Content-Type: application/json" \
    -d '{
      "name": "GitHub Development Digest to Slack",
      "prompt": "Use the GitHub MCP server for exactly one repository: '"${GITHUB_REPO_FILTER}"'. Inspect recent development activity since the previous weekday, including merged pull requests, newly opened or reopened pull requests, notable commits pushed to main or release branches, new issues, important issue updates, releases, risks, blockers, and review requests. Keep GitHub lookups small: inspect the latest 3 to 5 commits, pull requests, issues, and releases. Use the Slack MCP server to post directly to channel ID '"${SLACK_DIGEST_CHANNEL}"'. Keep the Slack message concise: title with date range, 3 to 7 bullets, links back to GitHub, and a Needs attention section only if needed. End with: This digest was generated by an AI agent (OpenHands) on behalf of the user. Do not include secrets, raw tokens, private environment variables, or unrelated Slack messages.",
      "trigger": {
        "type": "cron",
        "schedule": "0 9 * * 1-5",
        "timezone": "'"${DIGEST_TIMEZONE}"'"
      },
      "timeout": 900
    }'
)"

echo "${CREATE_RESPONSE}" | python3 -m json.tool
export AUTOMATION_ID="$(
  python3 -c 'import json,sys; print(json.load(sys.stdin)["id"])' \
    <<< "${CREATE_RESPONSE}"
)"
```

The response includes the new automation ID, the cron trigger, and the
generated prompt-preset entrypoint.

![Agent Canvas automation detail after creation](screenshots/06-automation-created.png)

## 8. Test the Automation

Run the automation once:

```bash
curl -sS -X POST "${AUTOMATION_API_URL}/v1/${AUTOMATION_ID}/dispatch" \
  -H "Authorization: Bearer ${AGENT_CANVAS_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{}' | python3 -m json.tool
```

Check the run status:

```bash
curl -sS "${AUTOMATION_API_URL}/v1/${AUTOMATION_ID}/runs?limit=5" \
  -H "Authorization: Bearer ${AGENT_CANVAS_API_KEY}" | python3 -m json.tool
```

You should see the latest run transition to `COMPLETED`, and the target Slack
channel should contain the digest.

![Agent Canvas automation run completed successfully](screenshots/08-automation-run-completed.png)

![Slack channel showing the generated OpenHands digest](screenshots/09-slackbot-message.png)

## Troubleshooting

- **Lemonade is down after a reboot or power outage:** restart it with the
  `setsid -f build/lemond ...` command in step 1, then re-run the health check.
- **Agent Server rejects the LLM settings after setting `custom_tokenizer`:**
  install `transformers` in the Agent Server Python environment, restart Agent
  Server if needed, and retry the settings patch. OpenHands requires
  Transformers to load the tokenizer chat template when `custom_tokenizer` is
  set.
- **Agent Canvas cannot reach Lemonade:** verify
  `curl -fsS "${LEMONADE_BASE_URL}/health"` and confirm the base URL in the
  Agent Server settings matches the running local endpoint or HTTPS tunnel.
- **The automation API returns 401:** load
  `AGENT_CANVAS_API_KEY` from `~/.openhands/agent-canvas/api-key.txt`. Use it
  as `Authorization: Bearer ...` for the automation backend.
- **GitHub MCP cannot see private repositories:** confirm the GitHub token has
  read access to the target repository and that the MCP test endpoint advertises
  GitHub tools.
- **Slack can read channels but cannot post:** invite the Slack app to the
  target channel and confirm the bot has `chat:write`.
- **The automation lists too many Slack channels:** use a Slack channel ID and
  set `SLACK_CHANNEL_IDS` on the Slack MCP server.
- **The automation exceeds context:** confirm Lemonade was started with
  `ctx_size=65536`, confirm the OpenHands LLM has `custom_tokenizer` set, and
  confirm the condenser has `max_tokens` set below the Lemonade context window.
  Also use an explicit repository and cap GitHub result sets to 3 to 5 items.
- **SGLang returns tool text instead of OpenAI tool calls:** include
  `"chat_template_kwargs":{"enable_thinking":false}` in LiteLLM extra body when
  using Qwen3 with SGLang tools.

## Next Steps

- Add a weekly release-only digest.
- Add a GitHub event-triggered automation for faster PR or push alerts.
- Route the same digest into Notion, Linear, or another MCP-backed tool.

## Resources

- [AMD AI Playbooks](https://developer.amd.com/playbooks/)
- [Lemonade Server documentation](https://lemonade-server.ai/docs)
- [OpenHands extensions repository](https://github.com/OpenHands/extensions)
- [Model Context Protocol servers](https://github.com/modelcontextprotocol/servers)
- [Slack MCP package](https://www.npmjs.com/package/@modelcontextprotocol/server-slack)
