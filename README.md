# Hermes Local Agentic AI (Docker + Ollama)

A fully local AI agent stack (no external APIs). It combines language models installed inside Docker, the Hermes Agent, and a web chat interface. Everything is defined as Infrastructure as Code (`docker-compose.yml`) and starts with a single command.

Thai version: [README.md](README.md)

## 1. Problem and Approach

- **Problem:** We want an AI agent that runs on our own machine, keeps data local, and can be reproduced easily on another computer.
- **Approach:** Ollama serves the models inside Docker, Hermes Agent (Nous Research) acts as the agent, nginx serves the web UI, and Docker Compose provides the IaC layer.

## 2. Architecture

```
 User (Browser)            User (VS Code / Terminal)
       |                              |
       v                              v
  [web: nginx :8080]         [hermes-agent :8642]
       | /api/*                       | OPENAI_BASE_URL
       v                              v
          [ollama :11434 (host port: 11435)]
           ├─ hermes3:3b
           ├─ qwen2.5:3b
           └─ hermes-th (built from the Modelfile)
                ^
   [ollama-init] pulls the models automatically on startup
```

| Service | Role | Host port |
|---|---|---|
| ollama | Runs the language models inside Docker | 11435 |
| ollama-init | Pulls the models, builds `hermes-th` from the Modelfile, then exits | - |
| hermes-agent | Hermes Agent, connected to Ollama over the internal Docker network | 8642 |
| web | Web chat UI + proxy for `/api` to Ollama | 8080 |

## 3. Workflow / Input / Output

1. **Input:** The user types a message in the web UI (or sends a command from a terminal / VS Code).
2. The web UI sends `POST /api/chat`; nginx forwards it to Ollama.
3. Ollama runs the selected model and streams the response back.
4. **Output:** The reply appears incrementally in the web UI.

## 4. How to Run (Fully Local)

Requires Docker Desktop to be running. An internet connection is needed only the first time, to download the models; after that the stack runs offline.

```bash
docker compose up -d
docker compose logs -f ollama-init     # watch model download progress (wait for "models ready")
```

Open http://localhost:8080 and pick a model from the dropdown in the top-right corner.

Stop the stack: `docker compose down` (model data is kept in the `ollama_data` volume, so no re-download is needed).

## 5. Connecting from Outside (VS Code / Terminal / Web)

**Terminal / VS Code terminal**
```bash
# Chat with the Hermes Agent running in Docker
docker exec -it hermes-agent hermes chat

# Call the Ollama instance inside Docker directly via its API
curl http://localhost:11435/api/tags
curl http://localhost:11435/api/chat -d '{"model":"hermes-th","messages":[{"role":"user","content":"Hello"}],"stream":false}'
```

**Web:** http://localhost:8080 (chat UI) and http://localhost:8642 (Hermes API server)

## 6. Model Installation: Locally and Inside Docker

- **Local (host):** `ollama pull hermes3:3b` (port 11434)
- **Inside Docker:** `ollama-init` automatically pulls `hermes3:3b` and `qwen2.5:3b` (port 11435)
- Verify: `ollama list` (host) and `docker exec ollama ollama list` (Docker)

## 7. Multiple Model Test

Ask every model the same question and record the results:

```bash
docker exec ollama ollama run hermes3:3b "Explain Docker Compose briefly"
docker exec ollama ollama run qwen2.5:3b "Explain Docker Compose briefly"
docker exec ollama ollama run hermes-th "Explain Docker Compose briefly"
```

| Model | Thai quality | Speed | Notes |
|---|---|---|---|
| hermes3:3b | (fill in test result) | | |
| qwen2.5:3b | (fill in test result) | | |
| hermes-th | (fill in test result) | | |

## 8. Conclusions

- Ollama is used to **run and customize** models (system prompt and parameters via a Modelfile); it does not retrain them (no fine-tuning).
- The whole system is described as code in `docker-compose.yml`, so it can be reproduced on another machine with one command.
- Limitations: 3B models are less capable than larger ones, and responses are slower without a GPU.

## File Structure

```
hermes-web/
├── docker-compose.yml   # IaC: defines every service
├── Modelfile            # customizes the hermes-th model
├── nginx.conf           # proxies the web UI to Ollama
├── index.html           # web chat UI
├── hermes-data/
│   └── config.yaml      # points Hermes Agent at the Ollama instance in Docker
├── README.md            # Thai documentation
└── README.en.md         # English documentation
```
