 # claude-on-foundry-lab

## Setup the Azure AI Gateway lab

An existing lab walkthrough the deployment and configuration of the Azure AI Gateway
`https://github.com/Azure/aigateway/blob/main/labs/25-claude-code/README.md`


## Client configuration and deployment using MDM solution

### managed-settings.json

`managed-settings.json` is a better option for centrally managed client configuration. This file takes precedence over all other settings files, allowing users to customize their own `settings.json` file without compromising the general configuration.

On Windows, the file is located at `C:\ProgramFiles\ClaudeCode\managed-settings.json`

```json
{
    "apiKeyHelper": "python C:\\ProgramFiles\\ClaudeCode\\auth.py",
    "env": {
        "CLAUDE_GATEWAY_SCOPE": "https://azure-api.net/aigateway/Data.All",
        "CLAUDE_GATEWAY_TENANT_ID": "<YOUR_TENANT_ID>",
        "CLAUDE_GATEWAY_CLIENT_ID": "04b07795-8ddb-461a-bbee-02f9e1bf7b46", //entra azure cli app id
        "ANTHROPIC_BASE_URL": "https://<YOUR_AI_GATEWAY_FQDN>/default/models/anthropic",
        "ANTHROPIC_MODEL": "claude-opus-5-5"
    },
    "autoUpdatesChannel": "latest"
}
```

### Auth script `auth.py`

Install MSAL with `python -m pip install msal msal[broker]`, then save this as `C:\ProgramFiles\ClaudeCode\auth.py`. It runs directly on Windows, macOS, and Linux without Azure CLI. Use `python3` instead of `python` where needed.


```python
#auth.py
import argparse
import json
import os
import sys
import msal

parser = argparse.ArgumentParser()
parser.add_argument("--scope", default=os.environ.get("CLAUDE_GATEWAY_SCOPE"))
parser.add_argument("--as-headers", action="store_true")
args = parser.parse_args()
if not args.scope:
    parser.error("Set CLAUDE_GATEWAY_SCOPE or pass --scope.")

app = msal.PublicClientApplication(
    os.environ["CLAUDE_GATEWAY_CLIENT_ID"],
    authority=f"https://login.microsoftonline.com/{os.environ['CLAUDE_GATEWAY_TENANT_ID']}",
)
flow = app.initiate_device_flow(scopes=[args.scope])
if "user_code" not in flow:
    sys.exit(flow.get("error_description", "Could not start device sign-in."))
print(flow["message"], file=sys.stderr, flush=True)
result = app.acquire_token_by_device_flow(flow)
if "access_token" not in result:
    sys.exit(result.get("error_description", "Sign-in failed."))
token = result["access_token"]
print(json.dumps({"Authorization": f"Bearer {token}"}) if args.as_headers else token)
```

For a standalone terminal run, set `CLAUDE_GATEWAY_CLIENT_ID`, `CLAUDE_GATEWAY_TENANT_ID`, and `CLAUDE_GATEWAY_SCOPE` in your shell first, then run `python ~/.claude/auth.py`. The settings `env` block supplies these when Claude Code invokes the helper.

This minimal lab example signs in on each invocation; it does not persist tokens or provide unattended refresh. Complete the displayed device-code instructions promptly when used as a helper. Stdout contains only the token (or JSON headers with `--as-headers`); instructions and errors go to stderr. Do not log or commit tokens.

The settings examples use Windows commands. On macOS/Linux, use `python3 \"$HOME/.claude/auth.py\"` for `apiKeyHelper`, and append `--as-headers --scope https://monitor.azure.com/.default` for `otelHeadersHelper`.

### OTLP collector for client telemetry

`managed-settings.json` is a better option for centrally governing client configuration. This file takes precedence over all other settings files, allowing users to customize their `settings.json` file.

```json
//Gateway configuration removed for readability
{
  //apiKeyHelper....
  "otelHeadersHelper": "python C:\\ProgramFiles\\ClaudeCode\\auth.py --as-headers --scope https://monitor.azure.com/.default",
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_LOGS_EXPORTER": "otlp",
    "OTEL_EXPORTER_OTLP_PROTOCOL": "http/protobuf",
    "OTEL_EXPORTER_OTLP_TRACES_ENDPOINT": "<OTEL_EXPORTER_OTLP_TRACES_ENDPOINT>",
    "OTEL_EXPORTER_OTLP_LOGS_ENDPOINT": "<OTEL_EXPORTER_OTLP_LOGS_ENDPOINT>",
    "OTEL_EXPORTER_OTLP_METRICS_ENDPOINT": "<OTEL_EXPORTER_OTLP_METRICS_ENDPOINT>",
    "OTEL_LOG_USER_PROMPTS": "1",
    "OTEL_LOG_TOOL_DETAILS": "1",
    "OTEL_METRICS_INCLUDE_VERSION": "true",

    //additional gateway environment variables
    //...
  },
  "autoUpdatesChannel": "latest"
}
```