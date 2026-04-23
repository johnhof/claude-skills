---
name: tunnel-db-prod
description: Open an Aptible tunnel to the Noho production app database (port 25432) in a background tmux session and print the connection string.
---

⚠️ This connects to the PRODUCTION app database. Confirm with the user before running any writes or destructive queries.

Run the following bash command to open the tunnel:

```bash
tmux kill-session -t db-prod-tunnel 2>/dev/null; tmux new-session -d -s db-prod-tunnel "aptible db:tunnel noho --environment noho-production --type postgresql --port 25432" && sleep 12 && tmux capture-pane -t db-prod-tunnel -p
```

Then print the connection details in this format:

```
Tunnel open on port 25432 (PRODUCTION)

psql "host=localhost.aptible.in port=25432 dbname=<dbname> user=aptible password=<password>"

Close with: tmux kill-session -t db-prod-tunnel
```

Extract the dbname and password from the connection string in the tmux output and include them in the printed psql command.
