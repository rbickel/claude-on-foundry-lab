# claude-on-foundry-lab


## Client configuration

### managed-settings.json

`managed-settings.json` is a better option for centrally governing client configuration. This file takes precedence over all other settings files, allowing users to customize their `settings.json` file.

```json
{
  "apiKeyHelper": "python \"%USERPROFILE%\\.claude\\auth.py\"",
  "env": {
    "CLAUDE_GATEWAY_SCOPE": "https://azure-api.net/aigateway/Data.All",
    "CLAUDE_GATEWAY_TENANT_ID": "<YOUR_TENANT_ID>",
    "CLAUDE_GATEWAY_CLIENT_ID": "<YOUR_PUBLIC_CLIENT_APP_ID>",
    "ANTHROPIC_BASE_URL": "https://<YOUR_GATEWAY_PREFIX>.ai.gateway.azure.com/default/models/anthropic",
    "ANTHROPIC_MODEL": "claude-opus-5-5"
  },
  "autoUpdatesChannel": "latest"
}
```

### Auth script `auth.py`

Install MSAL with `python -m pip install msal`, then save this as `~/.claude/auth.py`. It runs directly on Windows, macOS, and Linux without Azure CLI. Use `python3` instead of `python` where needed.

Use your existing Entra app's client ID below. Enable **Allow public client flows**, grant delegated permissions/consent for the requested API, and ensure your tenant permits device-code sign-in. The user must also have access to the target resource.

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
//Gateway configuration remove for readability
{
  "otelHeadersHelper": "python \"%USERPROFILE%\\.claude\\auth.py\" --as-headers --scope https://monitor.azure.com/.default",
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_LOGS_EXPORTER": "otlp",
    "OTEL_EXPORTER_OTLP_PROTOCOL": "http/protobuf",
    "OTEL_EXPORTER_OTLP_ENDPOINT": "https://<APPLICATION_INSIGHT_OTPL_ENDPOINT>",
    "OTEL_LOG_USER_PROMPTS": "1",
    "OTEL_LOG_TOOL_DETAILS": "1",
    "OTEL_METRICS_INCLUDE_VERSION": "true",

    //additional gateway environment variables
    //...
  },
  "autoUpdatesChannel": "latest"
}
```