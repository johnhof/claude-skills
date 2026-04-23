---
name: tunnel-dw
description: Open an Aptible tunnel to the Noho data warehouse (port 35432) in a background tmux session and print the connection string.
---

Run the following bash command to open the tunnel:

```bash
tmux new-session -d -s dw-tunnel "aptible db:tunnel data-warehouse --environment noho-production --type postgresql --port 35432" 2>&1 && sleep 12 && tmux capture-pane -t dw-tunnel -p
```

Then print the connection details in this format:

```
Tunnel open on port 35432

psql "host=localhost.aptible.in port=35432 dbname=db user=aptible password=<password>"

Close with: tmux kill-session -t dw-tunnel
```

If the tunnel fails because the session already exists, kill the old one first:

```bash
tmux kill-session -t dw-tunnel 2>/dev/null; tmux new-session -d -s dw-tunnel "aptible db:tunnel data-warehouse --environment noho-production --type postgresql --port 35432" && sleep 12 && tmux capture-pane -t dw-tunnel -p
```

Extract the password from the connection string in the tmux output and include it in the printed psql command.
