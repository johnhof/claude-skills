---
name: tunnel-db-staging
description: Open an Aptible tunnel to the Noho staging app database (port 15432) in a background tmux session and print the connection string.
---

Run the following bash command to open the tunnel:

```bash
tmux kill-session -t db-staging-tunnel 2>/dev/null; tmux new-session -d -s db-staging-tunnel "aptible db:tunnel noho --environment noho-staging --type postgresql --port 15432" && sleep 12 && tmux capture-pane -t db-staging-tunnel -p
```

Then print the connection details in this format:

```
Tunnel open on port 15432 (staging)

psql "host=localhost.aptible.in port=15432 dbname=<dbname> user=aptible password=<password>"

Close with: tmux kill-session -t db-staging-tunnel
```

Extract the dbname and password from the connection string in the tmux output and include them in the printed psql command.
