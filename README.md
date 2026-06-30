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

Developers spend a lot of time on small recurring loops: reviewing labeled
pull requests, answering GitHub comments, triaging new issues, turning Slack
threads into standup notes or incident follow-ups, and tracking release or
research signals. Each loop is familiar, but it still requires judgment:
gather the right context, decide what matters, and post a clear update where
the team already works.

[OpenHands automations](https://docs.openhands.dev/openhands/usage/automations/overview)
turn those loops into scheduled or event-triggered agent conversations: runs
where an AI software agent can read context, call tools, and produce an update.
The shared automation templates in the OpenHands extensions catalog follow
this pattern for GitHub pull request review, repository monitoring, Linear
issue triage, incident retrospectives, Slack standup digests, and research
briefs: an automation wakes up, uses configured integrations such as GitHub or
Slack to fetch context, reasons over that context with a large language model
(LLM), and writes back a result.

[Agent Canvas](https://github.com/OpenHands/agent-canvas) is the local control
plane for building and testing those automations. In this playbook it runs an
OpenHands Agent Server, the backend process that executes agent conversations,
and connects the agent to external services such as GitHub and Slack.

To keep the workflow on your AMD system, the agent talks to a local model
served by Lemonade Server. Lemonade exposes that model through an
OpenAI-compatible API, so Agent Canvas can configure it like a remote
OpenAI-style endpoint while the model, prompt, and workflow context stay local.

In this playbook, you will build one concrete automation: a scheduled
GitHub-to-Slack development digest. It uses GitHub to inspect recent repository
activity, Slack to post the digest, Agent Canvas API calls to configure and
test the automation, and Lemonade to run the LLM locally.

![Architecture diagram showing GitHub MCP, OpenHands automation, Lemonade Server, and Slack MCP](screenshots/00-architecture-overview.png)

## What You'll Learn

- How to start Lemonade Server and verify a local model answers chat requests
- How to launch Agent Canvas and point its Agent Server at a local LLM
- How to install GitHub and Slack Model Context Protocol (MCP) servers through
  the Agent Server API
- How to create and dispatch a scheduled OpenHands automation that posts a
  development digest to Slack
- How to troubleshoot the most common local-model and automation failures

## Core Concepts

| Concept | What it is | Where it fits in this playbook |
| --- | --- | --- |
| Lemonade Server | A local LLM serving platform built for AMD hardware that exposes an OpenAI-compatible API. Your data never leaves your machine. | Runs the model that powers the agent. |
| OpenHands Agent Server | The backend process that executes OpenHands agent conversations. | Hosts the agent, its LLM profile, and its MCP servers. |
| Agent Canvas | The local control plane for OpenHands that runs Agent Server and a UI for inspecting agent runs. | Launches the backends and provides the API you call. |
| MCP server | A Model Context Protocol server that gives an agent tools for an external service such as GitHub or Slack. | Lets the agent read GitHub and write to Slack. |
| OpenHands automation | A scheduled or event-triggered agent conversation that fetches context, reasons over it, and writes a result somewhere. | The GitHub-to-Slack digest you build here. |

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

## 3. Start Agent Canvas

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

## 4. Configure Agent Server Through the API

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

## 5. Install GitHub and Slack MCP Servers

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

## 6. Create the Digest Automation

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

## 7. Test the Automation

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
