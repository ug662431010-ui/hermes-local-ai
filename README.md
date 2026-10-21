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

**Method.** All three models were selected from the dropdown of the web chat UI (http://localhost:8080) and served by the Ollama instance running inside Docker (CPU only, no GPU). Each model was asked the same kinds of questions:

1. "Introduce yourself in English"
2. "Introduce yourself in Thai"
3. A simple year subtraction (for example "What is 2026-1998")

The whole test was run twice: once **with internet access** and once **with the internet disconnected**, to confirm that the stack is fully local.

### 7.1 Results with internet access

| Model | Works | Speed | Thai | English | Arithmetic |
|---|---|---|---|---|---|
| hermes3:3b | Yes | Slightly slow, but acceptable | Distorted: odd wording and stray foreign-script characters | Good | Mostly correct (see 7.3) |
| qwen2.5:3b | Yes | Fast | Good | Good | Correct in most runs (see 7.3) |
| hermes-th | Yes | A little slower than hermes3:3b, but not by much | Says it is specialized in Thai, but odd characters still appear in the output | Good | Gave the most detailed explanations of the three (see 7.3) |

**Summary:** all three models work. `hermes3:3b` is held back mainly by its Thai output; `qwen2.5:3b` gave the best overall mix of speed and language quality.

### 7.2 Results without internet access (offline)

| Model | Works | Speed | Thai | English | Arithmetic |
|---|---|---|---|---|---|
| hermes3:3b | Yes | Slightly slow, but acceptable | Distorted (same as online) | Good | Mostly correct (see 7.3) |
| qwen2.5:3b | Yes | Fast | Odd output (stray Chinese characters mixed in) | Good | Correct in most runs (see 7.3) |
| hermes-th | Yes | Fast | Usable, with fewer odd outputs than the other two | Good | Answered subtraction questions (see 7.3) |

**Summary:** the stack works with no internet connection, which satisfies the "totally local" requirement. Offline, `hermes-th` produced the cleanest Thai of the three.

### 7.3 Arithmetic errors observed

Small models are not reliable for arithmetic. The following wrong answers were recorded in the test screenshots:

| Model | Question | Model's answer | Correct answer |
|---|---|---|---|
| hermes3:3b | 2026 - 1995 | 131 | 31 |
| hermes-th | 2026 - 1985 | 141 | 41 |
| qwen2.5:3b | 2026 - 1988 | 37 (first attempt, answered in Chinese); 38 when asked again in English | 38 |

Other runs were correct (for example 2026 - 2005 = 21, 2026 - 1998 = 28, 2026 - 1983 = 43). Because of these errors, none of these 3B models should be trusted for calculations without verification.

### 7.4 Hermes Agent compatibility (terminal)

Hermes Agent requires a context window of at least 64,000 tokens, so the Ollama context was raised to 65,536 in `docker-compose.yml`.

- `hermes3:3b`: runs in the terminal agent (including with the internet disconnected). It answers simple questions, but some replies contain leaked tool-call text, and Hermes Agent itself warns that Hermes 3 models are not designed for agent tool use.
- `qwen2.5:3b`: **cannot be used with the agent.** Ollama loads it with only 32,768 tokens of context, below the agent's minimum. It still works normally in the web chat UI.
- `hermes-th`: tested in the web chat UI only.

## 8. Conclusions

- Ollama is used to **run and customize** models (system prompt and parameters via a Modelfile); it does not retrain them (no fine-tuning).
- The whole system is described as code in `docker-compose.yml`, so it can be reproduced on another machine with one command, and it keeps working without an internet connection once the models are downloaded.
- All three models run on CPU inside Docker. `qwen2.5:3b` is the fastest; `hermes-th` was the best at Thai when offline; `hermes3:3b` is the only one that is also usable with the terminal agent.
- Limitations: 3B models are less capable than larger ones, Thai output is often distorted, arithmetic is unreliable (section 7.3), and responses are slower without a GPU.

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
