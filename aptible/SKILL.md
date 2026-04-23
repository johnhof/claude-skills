---
name: aptible
description: Aptible operations — tunnel to databases and restart apps. Usage: /aptible <subcommand> [target]
---

Parse the user's arguments to determine which subcommand to run.

## Subcommands

### tunnel

Open a background tmux tunnel to a database. Kill any existing session with the same name first.

| Target | Command | Session name | Port |
|--------|---------|--------------|------|
| `dw` | `aptible db:tunnel data-warehouse --environment noho-production --type postgresql --port 35432` | `dw-tunnel` | 35432 |
| `staging` | `aptible db:tunnel noho --environment noho-staging --type postgresql --port 15432` | `db-staging-tunnel` | 15432 |
| `prod` | `aptible db:tunnel noho --environment noho-production --type postgresql --port 25432` | `db-prod-tunnel` | 25432 |

Pattern for all tunnels:
```bash
tmux kill-session -t <session> 2>/dev/null; tmux new-session -d -s <session> "<aptible command>" && sleep 12 && tmux capture-pane -t <session> -p
```

After running, extract the connection string from the tmux output and print:
```
Tunnel open — <target> on port <port>

psql "host=localhost.aptible.in port=<port> dbname=<dbname> user=aptible password=<password>"

Close with: tmux kill-session -t <session>
```

For `prod`, print a ⚠️ warning before running: "This is the PRODUCTION app database."

### restart

Restart an Aptible app. If no target is given, ask the user which app.

| Target | App | Environment |
|--------|-----|-------------|
| `api` or `backend` or `prod` | `patient-app-backend` | `noho-production` |
| `api-staging` or `backend-staging` or `staging` | `patient-app-backend-staging` | `noho-staging` |
| `dagster` | `dagster` | `noho-production` |
| `metabase` | `metabase` | `noho-production` |
| `infobot` | `infobot-staging` | `noho-staging` |

Command:
```bash
aptible restart --app <app> --environment <environment>
```

Print confirmation before running:
```
Restarting <app> in <environment>...
```

Then run the command and report success or failure.

## If arguments are missing or ambiguous

List available subcommands and targets and ask the user to clarify.
