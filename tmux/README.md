docker attach

1. Start a tmux session:
   ```bash
   tmux new -s docker-logs
   ```
2. Run:
   ```bash
   docker logs -f <container_name_or_id>
   ```
3. Detach but keep it running: press `Ctrl + B` then `D`.  
4. Reattach later:
   ```bash
   tmux attach -t docker-logs
   ```
