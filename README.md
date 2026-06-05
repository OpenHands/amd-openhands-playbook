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

In this playbook, you will run a local LLM with Lemonade Server or SGLang,
connect Agent Canvas to that local OpenAI-compatible endpoint, and create an
automation that summarizes recent GitHub development activity into a Slack
digest.

The workflow is intentionally local-first:

1. Lemonade Server or SGLang hosts the model on your AMD system.
2. Agent Canvas uses that model through the same `/api/v1` OpenAI-compatible
   interface that cloud LLM clients use.
3. OpenHands Automations runs a scheduled task that reads GitHub activity and
   posts a short summary to Slack through MCP servers.

This gives you a private daily or weekly engineering update without sending the
model prompt or repository summary to a hosted LLM provider.

<!-- @device:stx,krk -->
> [!NOTE]
> A coding-agent workflow benefits from a larger model and a larger context
> window. Use at least 32 GB of system memory, and prefer 64 GB or more for
> larger Qwen, GPT-OSS, or similar GGUF models.
<!-- @device:end -->

## What You'll Learn

- How to start and verify Lemonade Server as a local LLM endpoint.
- How to start SGLang with grammar-constrained OpenAI tool calling.
- How to configure Agent Canvas through its onboarding flow.
- How to install GitHub and Slack MCP servers.
- How to create a scheduled automation with the prompt preset API.
- How to test the automation and tune the Slack digest format.

## Architecture

```text
GitHub repos  --->  GitHub MCP  ----+
                                    |
                                    v
                              OpenHands automation
                                    |
                                    v
Local LLM server <--- Agent Server / Agent Canvas ---> Slack MCP ---> Slack channel
```

The automation itself is created from a natural-language prompt. The prompt
defines the GitHub scope, reporting cadence, Slack destination, and output
format. The prompt preset endpoint handles the automation boilerplate for you.

## Prerequisites

<!-- @os:linux -->
<!-- @require:lemonade,nodejs -->
<!-- @os:end -->

<!-- @os:windows -->
<!-- @require:lemonade,nodejs -->
<!-- @os:end -->

You also need:

- A GitHub personal access token with read access to the repositories you want
  summarized. If the automation will comment or create issues, add only the
  specific write permissions it needs.
- A Slack app bot token that starts with `xoxb-`.
- A Slack workspace/team ID that starts with `T`.
- A Slack channel ID or channel name where the digest should be posted.
- Agent Canvas running with the automation stack enabled.

For the Slack MCP server, the common bot token scopes are:

- `channels:history`
- `channels:read`
- `chat:write`
- `reactions:write`
- `users:read`
- `users.profile:read`

If you want to summarize private Slack channels, add the corresponding private
channel scopes and invite the Slack app to those channels. For a GitHub-only
development digest, the automation does not need to read Slack history; Slack is
only the reporting destination.

## Variables Used in This Playbook

<!-- @device:halo,halo_box,stx,krk -->
<!-- @var:id=lemonade_model value="Qwen3.6-35B-A3B-GGUF" -->
<!-- @device:end -->

```bash
export LEMONADE_BASE_URL="http://127.0.0.1:13305/api/v1"
export LEMONADE_MODEL="Qwen3.6-35B-A3B-GGUF"
export SGLANG_BASE_URL="http://127.0.0.1:13306/v1"
export SGLANG_MODEL="adp-qwen35-4b-flashattn-ckpt1000"
export AGENT_CANVAS_URL="http://localhost:8000"
export AUTOMATION_API_URL="http://localhost:8000/api/automation"
export GITHUB_REPO_FILTER="your-org/*"
export SLACK_DIGEST_CHANNEL="#eng-digest"
export DIGEST_TIMEZONE="America/New_York"
```

For a remote OpenHands or cloud automation environment, replace
`LEMONADE_BASE_URL` with a reachable HTTPS tunnel URL, for example:

```bash
export LEMONADE_BASE_URL="https://YOUR_NGROK_DOMAIN.ngrok-free.dev/api/v1"
```

The current local development setup this playbook was modeled from serves
Lemonade on `127.0.0.1:13305`, with the loaded model
`Qwen3.6-35B-A3B-GGUF`, a llama.cpp Vulkan backend, and a 65,536-token context.

If you use SGLang instead of Lemonade, this playbook was also verified with
SGLang on `127.0.0.1:13306`, serving
`adp-qwen35-4b-flashattn-ckpt1000` with a 32,768-token context.

## 1. Start Lemonade Server

If Lemonade is already installed, start a model from the Lemonade CLI:

```bash
lemonade config set llamacpp.backend=vulkan
lemonade config set ctx_size=65536
lemonade run "${LEMONADE_MODEL}"
```

If you are developing Lemonade from a local source checkout, you can run the
server binary directly:

```bash
cd ~/work/lemonade
build/lemond --host 127.0.0.1 --port 13305
```

Lemonade exposes an OpenAI-compatible API at:

```text
http://127.0.0.1:13305/api/v1
```

Optional: if Agent Canvas or the automation runner is not on the same machine,
publish the Lemonade endpoint through a secure tunnel:

```bash
ngrok http 13305 --url YOUR_NGROK_DOMAIN.ngrok-free.dev
```

Use the HTTPS tunnel URL as the Agent Canvas LLM base URL in later steps.

### Optional: Use SGLang for Grammar-Constrained Tool Calls

SGLang is a better fit when the automation depends heavily on OpenAI `tools`.
Launch SGLang with an explicit grammar backend, Qwen reasoning parser, Qwen
tool-call parser, and strict tool decoding:

```bash
cd ~/work/sglang
source ~/.venvs/sglang-rocm/bin/activate

export SGLANG_TOOL_STRICT_LEVEL=2
export HIP_VISIBLE_DEVICES=0
export PYTHONPATH="$HOME/work/sglang/python:${PYTHONPATH:-}"

python -u -m sglang.launch_server \
  --model-path "$HOME/work/models/adp-qwen35-4b-flashattn-ckpt1000" \
  --host 127.0.0.1 \
  --port 13306 \
  --served-model-name "${SGLANG_MODEL}" \
  --dtype bfloat16 \
  --context-length 32768 \
  --max-total-tokens 32768 \
  --max-running-requests 1 \
  --mem-fraction-static 0.75 \
  --attention-backend torch_native \
  --grammar-backend xgrammar \
  --reasoning-parser qwen3 \
  --tool-call-parser qwen \
  --disable-cuda-graph \
  --disable-custom-all-reduce
```

Verify the OpenAI-compatible endpoint:

```bash
curl -fsS "${SGLANG_BASE_URL}/models" | python3 -m json.tool
```

Then verify parsed OpenAI tool calls. The `chat_template_kwargs` field is
important for this Qwen3-style checkpoint: without `enable_thinking: false`,
SGLang can return a grammar-constrained tagged call in `reasoning_content`
instead of populating `message.tool_calls`.

```bash
curl -fsS "${SGLANG_BASE_URL}/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "adp-qwen35-4b-flashattn-ckpt1000",
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
`choices[0].message.tool_calls` array.

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

List Lemonade models and confirm the selected model is downloaded:

```bash
curl -s "${LEMONADE_BASE_URL}/models" | python3 -m json.tool
```

You should see an entry like:

```json
{
  "id": "Qwen3.6-35B-A3B-GGUF",
  "downloaded": true,
  "recipe": "llamacpp"
}
```

Send a quick chat completion request:

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

## 3. Start Agent Canvas with Automations

From the Agent Canvas checkout:

```bash
cd ~/work/agent-canvas
npm ci
npm run dev
```

For a static production-style frontend, use:

```bash
npm run dev:static
```

Both launchers start:

- Agent Server on port `18000`
- Automation backend on port `18001`
- Frontend on port `3001`
- Ingress proxy on port `8000`

Open Agent Canvas in your browser:

```text
http://localhost:8000
```

## 4. Complete the Onboarding Flow

Agent Canvas shows a four-step onboarding modal for first-time users.

### Choose Agent

Select **OpenHands**. This route uses the Agent Server LLM configuration you
will point at Lemonade. The other agent choices are ACP subprocess agents and
manage their own model settings.

### Check Backend

Confirm the local Agent Server is connected. If you used the default launcher,
the backend should be reachable through the ingress at:

```text
http://localhost:8000
```

Click **Next** only after the connection banner reports success.

### Set Up LLM

Open the advanced or all-settings view in the LLM step and use:

For Lemonade:

| Field | Value |
| --- | --- |
| Custom model | `openai/Qwen3.6-35B-A3B-GGUF` |
| Base URL | `http://127.0.0.1:13305/api/v1` |
| API key | `lemonade` |

For SGLang:

| Field | Value |
| --- | --- |
| Custom model | `openai/adp-qwen35-4b-flashattn-ckpt1000` |
| Base URL | `http://127.0.0.1:13306/v1` |
| API key | `sglang` |
| LiteLLM extra body | `{"chat_template_kwargs":{"enable_thinking":false}}` |

Why the `openai/` prefix? Agent Canvas talks to local OpenAI-compatible
servers through LiteLLM. The prefix tells LiteLLM to use OpenAI-compatible
request formatting, while the base URL redirects the request to the local LLM
server.

For the SGLang Qwen3-style checkpoint, keep `enable_thinking` disabled through
`chat_template_kwargs` when using OpenAI tools. In testing, leaving thinking
enabled produced tagged tool-call text in `reasoning_content`; disabling it
made SGLang return parsed OpenAI `message.tool_calls`.

If Agent Server cannot reach the local model server at `127.0.0.1:13305`
or `127.0.0.1:13306`, use your tunnel URL instead:

```text
https://YOUR_NGROK_DOMAIN.ngrok-free.dev/api/v1
```

Click **Next** to save the LLM profile.

### Say Hello

Send the default hello message or continue directly to the recommended
automation cards. The final onboarding step also contains recommended
automations; for this playbook, you will create a custom GitHub-to-Slack digest
in the next sections.

## 5. Install GitHub and Slack MCP Servers

Agent Canvas uses MCP servers to let the automation read GitHub and post to
Slack. Open the MCP directory:

```text
http://localhost:8000/mcp
```

Install **GitHub** from the marketplace:

| Field | Value |
| --- | --- |
| Command | `npx -y @modelcontextprotocol/server-github` |
| Personal access token | Your GitHub token |

Install **Slack** from the marketplace:

| Field | Value |
| --- | --- |
| Command | `npx -y @modelcontextprotocol/server-slack` |
| Bot token | Your `xoxb-...` Slack bot token |
| Team / Workspace ID | Your `T...` Slack workspace ID |

After installing the Slack server, invite the Slack app to the channel where
you want the digest posted.

## 6. Create the GitHub-to-Slack Digest Automation

Open a new Agent Canvas conversation and paste the prompt below. Replace the
placeholder values before sending.

When creating the automation from an Agent Canvas conversation, the agent sees a
`<RUNTIME_SERVICES>` block that lists the automation backend URL. Trust that
block instead of guessing ports; the Agent Server and the Automation backend are
different services.

```text
Create a scheduled OpenHands automation named "GitHub Development Digest to Slack".

Use the local OpenHands Automations prompt preset endpoint. Read the automation
backend URL from the <RUNTIME_SERVICES> block, and use the /api/automation/v1/preset/prompt
path on that backend. The automation should:

1. Run every weekday at 9:00 AM in America/New_York.
2. Use the GitHub MCP server to inspect recent development activity from repositories matching "your-org/*".
3. Summarize activity from the previous weekday, including:
   - merged pull requests
   - newly opened or reopened pull requests
   - notable commits pushed to main or release branches
   - new issues and high-priority issue updates
   - releases or tags
   - risks, blockers, and review requests that need attention
4. Use the Slack MCP server to post the digest to #eng-digest.
5. Keep the Slack post concise:
   - title with date range
   - 3 to 7 bullets of meaningful changes
   - "Needs attention" section only when there are blockers
   - links back to GitHub PRs, issues, commits, and releases
6. Do not include secrets, raw tokens, private environment variables, or unrelated Slack messages.
7. If GitHub returns too much activity, prioritize merged PRs, releases, and items labeled bug, security, release, or blocker.

Use this trigger:
{
  "type": "cron",
  "schedule": "0 9 * * 1-5",
  "timezone": "America/New_York"
}

Use a timeout of 900 seconds.
```

The agent should create the automation through the prompt preset, then return
the new automation ID and a short description of the configured trigger.

### API Equivalent

If you prefer to create the automation directly, use the local automation API.
The default local automation key is generated by the Agent Canvas launcher and
stored at `~/.openhands/agent-canvas/automation-api-key.txt`.

```bash
export OPENHANDS_AUTOMATION_API_KEY="$(cat ~/.openhands/agent-canvas/automation-api-key.txt)"

curl -sS -X POST "${AUTOMATION_API_URL}/v1/preset/prompt" \
  -H "Authorization: Bearer ${OPENHANDS_AUTOMATION_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "GitHub Development Digest to Slack",
    "prompt": "Use the GitHub MCP server to inspect recent development activity from repositories matching '"${GITHUB_REPO_FILTER}"' since the previous weekday. Summarize merged pull requests, new or reopened pull requests, notable commits pushed to main or release branches, new issues, important issue updates, releases, risks, blockers, and review requests. Use the Slack MCP server to post a concise digest to '"${SLACK_DIGEST_CHANNEL}"'. Include a title with the date range, 3 to 7 meaningful bullets, a Needs attention section only when needed, and links back to GitHub. Do not include secrets, raw tokens, private environment variables, or unrelated Slack messages.",
    "trigger": {
      "type": "cron",
      "schedule": "0 9 * * 1-5",
      "timezone": "'"${DIGEST_TIMEZONE}"'"
    },
    "timeout": 900
  }' | python3 -m json.tool
```

Save the returned `id`:

```bash
export AUTOMATION_ID="PASTE_RETURNED_ID_HERE"
```

## 7. Test the Automation

Run the automation once before waiting for the schedule:

```bash
curl -sS -X POST "${AUTOMATION_API_URL}/v1/${AUTOMATION_ID}/dispatch" \
  -H "Authorization: Bearer ${OPENHANDS_AUTOMATION_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{}' | python3 -m json.tool
```

Then inspect recent runs:

```bash
curl -sS "${AUTOMATION_API_URL}/v1/${AUTOMATION_ID}/runs?limit=5" \
  -H "Authorization: Bearer ${OPENHANDS_AUTOMATION_API_KEY}" | python3 -m json.tool
```

You should see a run transition from queued or running to completed. Check the
target Slack channel for a digest message.

## 8. Tune the Digest

After the first run, adjust the automation prompt if the digest is too long,
too short, or missing important context. Good prompt refinements include:

- Narrow the repository filter from `your-org/*` to a small allowlist.
- Add label priorities, such as `security`, `release`, `blocker`, or `customer`.
- Exclude noisy bot activity such as dependency update branches.
- Ask for a separate section for "merged", "in review", and "needs attention".
- Require all GitHub links to point to canonical PR, issue, commit, or release
  pages.

You can edit, disable, delete, and inspect automation runs from:

```text
http://localhost:8000/automations
```

## Troubleshooting

### Agent Canvas Cannot Reach the Local LLM Server

Verify the model server:

```bash
curl -fsS "${LEMONADE_BASE_URL}/health"
curl -fsS "${LEMONADE_BASE_URL}/models"
```

For SGLang, use:

```bash
curl -fsS "http://127.0.0.1:13306/health"
curl -fsS "${SGLANG_BASE_URL}/models"
```

If Agent Server runs on a different machine or in a remote sandbox, replace
`127.0.0.1` with a reachable host or HTTPS tunnel.

### The Model Selector Does Not Show the Local Model

Use the advanced LLM settings form and enter:

For Lemonade:

```text
Model: openai/Qwen3.6-35B-A3B-GGUF
Base URL: http://127.0.0.1:13305/api/v1
API key: lemonade
```

For SGLang:

```text
Model: openai/adp-qwen35-4b-flashattn-ckpt1000
Base URL: http://127.0.0.1:13306/v1
API key: sglang
LiteLLM extra body: {"chat_template_kwargs":{"enable_thinking":false}}
```

The API key is a placeholder for local Lemonade or SGLang, but many
OpenAI-compatible clients require the field to be non-empty.

### GitHub MCP Cannot See Private Repositories

Create a fine-grained GitHub token scoped to the repositories you need. Grant
read-only contents, issues, pull requests, metadata, and discussions access
unless the automation must write comments or update issues.

### Slack MCP Can Read Channels but Cannot Post

Confirm the Slack bot has `chat:write`, is installed to the workspace, and is
invited to the target channel. If the channel is private, the app must be added
to that private channel and have the corresponding private-channel scopes.

### Automation Creation Fails with 401

For local Agent Canvas, reload the API key from:

```bash
export OPENHANDS_AUTOMATION_API_KEY="$(cat ~/.openhands/agent-canvas/automation-api-key.txt)"
```

For OpenHands Cloud, use the hosted API key instead:

```bash
export OPENHANDS_API_KEY="YOUR_OPENHANDS_API_KEY"
```

Then call the cloud automation endpoint with:

```bash
-H "Authorization: Bearer ${OPENHANDS_API_KEY}"
```

## Next Steps

- Add a second weekly digest that includes only releases and release blockers.
- Add a Slack thread reply with raw links for developers who want details.
- Add a GitHub event-triggered automation for `pull_request.opened` or `push`
  events when you need faster alerts than a daily digest.
- Route the same digest into Notion, Linear, or another MCP-backed tool.

## Resources

- [AMD AI Playbooks](https://developer.amd.com/playbooks/)
- [AMD Playbooks GitHub repository](https://github.com/amd/playbooks)
- [Lemonade Server documentation](https://lemonade-server.ai/docs)
- [OpenHands extensions repository](https://github.com/OpenHands/extensions)
- [Model Context Protocol servers](https://github.com/modelcontextprotocol/servers)
- [Slack MCP package](https://www.npmjs.com/package/@modelcontextprotocol/server-slack)
