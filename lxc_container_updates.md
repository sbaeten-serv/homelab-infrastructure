Navigate into your service directory (e.g., `immich`, `jellyfin`, `nextcloud`):

```bash
sudo apt update && sudo apt upgrade -y
docker compose pull
docker compose up -d
docker image prune -f # safe command to remove unused dangling images
```

---

```bash
docker system prune -a --volumes -f # Warning: More invasive! Prunes all inactive volumes; only run when containers are active and running!
```

### Updating Odysseus

```bash
cd odysseus/odysseus
git pull
cd ..
docker compose down
docker compose up -d --build
```

### Useful Ollama Commands

```bash
# Access the interactive shell inside the Ollama container:
docker exec -it ollama-server 

# Example: Pulling a new model:
docker exec -it ollama-server ollama pull hermes3:8b
```