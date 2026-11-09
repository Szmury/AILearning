# AI Development Glossary

Concepts learned so far: 28

Each entry has review questions; the tutor asks them later to check retention.

**Topics:** [AI Application Development](#ai-application-development) · [Deployment and Infrastructure](#deployment-and-infrastructure) · [Machine Learning Fundamentals](#machine-learning-fundamentals) · [Python](#python) · [Security and Authentication](#security-and-authentication)

<!-- Entries are added by the glossary-keeper agent into the matching topic section, sorted alphabetically within it. New topics get their own `##` section, kept in alphabetical order, and are added to the Topics line above. -->

## AI Application Development

### Approximate nearest neighbour (ANN) search
**Learned:** 2026-10-19 · **Related:** Vector database (RAG), Embeddings and cosine similarity, Chunking (RAG) · **Reviewed:** 2026-10-19 (quiz passed)
**Review questions:** Why is exact search too slow for millions of vectors (it compares the query with every stored vector), and what does ANN trade away to fix that? Name the two main approaches (HNSW graph, IVF clusters). What exactly is traded away (lower recall, i.e. a small chance of missing the true best match)? Why is a missed neighbour usually acceptable in RAG?

Exact search compares the query with every stored vector, which is too slow at scale; ANN pre-organises the vectors (IVF clusters, or an HNSW layered graph) so each search skips most of them, accepting a small chance of missing the true best match. Quality is measured by recall (share of the true top-k returned, often 95-99%) and tuned with knobs like HNSW ef_search or IVF nprobe, where more effort means higher recall but slower queries. It is fine for RAG because several top-k chunks go to the LLM, so chunking and embedding quality matter more than one swapped neighbour. Chroma, Qdrant, pgvector and Pinecone use these methods; FAISS and Annoy expose them as libraries.

### async def vs. def (FastAPI)
**Learned:** 2026-10-20 · **Related:** GIL (Global Interpreter Lock), Threads vs. processes (Python), Generators (Python), FastAPI basics (endpoints, Pydantic validation, Depends) · **Reviewed:** 2026-10-20 (quiz passed)
**Review questions:** Classify these: an async LLM client, a local PyTorch model, a sync DB session. Which get `async def` and which get `def`? Why does a blocking call inside `async def` freeze every request, while the same call in a plain `def` does not? Watch out: a synchronous DB session means plain `def` even for a small query; the test is "can I await it?", not "is it small?".

Decide by whether the work is waiting or computing, and whether the slow call can be `await`ed. `async def` runs on one event loop and suits non-blocking I/O (async LLM client, `httpx.AsyncClient`, async DB driver); plain `def` runs in a thread pool and suits blocking code (local PyTorch model, `requests`, sync DB session). A blocking call inside `async def` (e.g. `time.sleep`, `model.predict`) freezes the loop for everyone, because coroutines only yield control at `await`, whereas threads are preempted by the OS. For long jobs (e.g. 50 s), use a job queue (Celery, RQ, arq): POST returns a job id, a worker runs it, and GET polls the status.

### Background work in FastAPI (BackgroundTasks vs. job queues)
**Learned:** 2026-10-26 · **Related:** async def vs. def (FastAPI), Streaming responses (FastAPI, SSE), FastAPI basics (endpoints, Pydantic validation, Depends), GIL (Global Interpreter Lock), Threads vs. processes (Python) · **Reviewed:** not yet
**Review questions:** Why is "the user waits too long" NOT an argument against BackgroundTasks, and what does a queue give the user instead? How can a few long BackgroundTasks hurt unrelated endpoints? Who writes the job status and who reads it? Watch out: BackgroundTasks also replies immediately; the difference is what exists after the reply (job id, status, retries).

Long jobs (e.g. ingesting a 2-minute PDF: chunk, embed, store) must not run inside the request: accept the work, reply right away, and do it elsewhere. `BackgroundTasks` runs the function after the response is sent, in the same process, which is fine for losable work (email, audit line, cache clear), but a plain `def` task shares the ~40-thread pool so a few long ones stall normal endpoints, a blocking `async def` task freezes the event loop, and tasks live only in memory, so a crash or deploy loses them silently with no job id, status or retry. A job queue (Celery = heavyweight and sync-first, RQ = simple Redis-only, arq = async-native Redis) makes the API only enqueue the job and return 202 with a job id; a separate worker takes it from a broker and writes stages (queued, chunking, embedding, storing, done/failed) to shared storage, and the client polls `GET /jobs/{id}` or gets SSE pushes, so jobs survive restarts and failed ones can be retried. Rule of thumb: if losing the job would matter or it takes more than a few seconds, use a queue; otherwise use `BackgroundTasks`.

### Chunking (RAG)
**Learned:** 2026-10-19 · **Related:** Vector database (RAG), Embeddings and cosine similarity, Batch, Generators (Python) · **Reviewed:** 2026-10-19 (quiz passed)
**Review questions:** Why are chunks that are too small or too big both a problem, and how do they fail differently (small loses context, big loses focus because the vector becomes an average)? What is the rule of thumb for a starting chunk size and overlap? What does overlap fix (ideas cut at a boundary), and what does it not fix (a chunk whose subject is far away; use a heading or title instead)?

Documents are split into chunks before embedding, because embedding models have input limits and one vector for a whole document blurs its meaning into an average. Too big gives a vague vector and wastes prompt space; too small loses context (e.g. "It expires after 30 days." without its subject). Start with recursive splitting at about 300-800 tokens with 10-15% overlap, then inspect what is actually retrieved; adding a title or section heading to each chunk fixes lost-subject chunks. A chunker can be written as a generator.

### Embeddings and cosine similarity
**Learned:** 2026-10-19 · **Related:** Vector database (RAG), Chunking (RAG) · **Reviewed:** 2026-10-19 (quiz passed)
**Review questions:** What happens to the dot product and to the cosine similarity if you double a vector's length (dot product doubles, cosine unchanged)? Why is cosine the safe default for text? On unnormalized vectors, how can raw dot product rank a worse match above a better-aligned one? Watch out: "length" means the vector's magnitude, not precision.

An embedding model turns text into a vector that captures meaning; similar meaning means a similar direction. Cosine similarity = (a·b)/(|a||b|) ignores vector length: 1 means same direction, 0 unrelated, -1 opposite. Most embedding models return length-1 vectors, where cosine, dot product and L2 give the same ranking, but with unnormalized vectors cosine is the safe choice.

### FastAPI architecture (router, service, repository) and lifespan
**Learned:** 2026-10-20 · **Related:** FastAPI basics (endpoints, Pydantic validation, Depends), Generators (Python), Threads vs. processes (Python), Tool use, MCP and the agent loop · **Reviewed:** 2026-10-20 (quiz passed)
**Review questions:** What does each layer own: router, service, repository? Why load an expensive model in `lifespan` before `yield` instead of per request, and what does 3 workers x 2 GB mean for memory? Watch out: know the no-service case (trivial CRUD can depend directly on the repo; test with `dependency_overrides` or an in-memory DB).

Routers stay thin (parse the request, call a service, map errors to HTTP); services hold business logic and orchestration (e.g. RAG: embed, search, build prompt, call the LLM, save); repositories do database access. A service should run without FastAPI objects, so it can also be called from a script. Expensive objects (ML model, engine, LLM client) load once in `lifespan` (code before `yield` runs at startup, code after it at shutdown) and are passed in via `Depends`, while cheap per-request objects (session, repo, service) are built per request. Each worker is a separate process with its own copy of the loaded objects.

### FastAPI basics (endpoints, Pydantic validation, Depends)
**Learned:** 2026-10-20 · **Related:** Decorators (Python), Generators (Python), Virtual environment (venv) and pip, Vector database (RAG), Tool use, MCP and the agent loop · **Reviewed:** 2026-10-20 (quiz passed)
**Review questions:** Why does invalid input return 422 while an exception in your code returns 500? What makes a field optional versus nullable (`price: float | None` with no default)? Watch out: on a 422 the function never runs; the `Depends` chain builds the session first, then the repo, and the code after `yield` runs after the endpoint returns.

FastAPI turns Python functions into HTTP endpoints via decorators (`@app.get`, `@app.post`); type hints and Pydantic `BaseModel` define the contract, giving automatic parsing, validation and `/docs`. Invalid input returns 422 (client error, function never runs), while an exception inside your code returns 500 (server error). `= default` makes a field optional, while `| None` only makes it nullable, so `price: float | None` with no default is still required. FastAPI has no built-in database: pair it with an ORM such as SQLModel (the engine is shared, the Session is per request), and inject dependencies like sessions with `Depends`; tests swap them using `app.dependency_overrides`.

### Streaming responses (FastAPI, SSE)
**Learned:** 2026-10-21 · **Related:** async def vs. def (FastAPI), Generators (Python), FastAPI basics (endpoints, Pydantic validation, Depends), FastAPI architecture (router, service, repository) and lifespan, Tool use, MCP and the agent loop · **Reviewed:** not yet
**Review questions:** Why can't FastAPI return a 500 when the LLM fails halfway through a stream, and what do you do instead? Is SSE a WebSocket, and when would you pick WebSocket? Why does the client need a `done` event, and why must a dropped connection be treated as an error? Why use `fetch` plus a reader and a buffer instead of `EventSource`? Watch out: the status line and headers are already sent once the first chunk goes out.

The endpoint returns a generator wrapped in `StreamingResponse(media_type="text/event-stream")`, so the LLM's tokens reach the user as they are produced instead of after the full answer; each `yield` is sent while the HTTP connection stays open. Use `async def` with `async for` over an async LLM client so the event loop stays free; a blocking source needs a plain `def` generator (run in the thread pool). SSE (Server-Sent Events) is plain HTTP with a chunked body, not a WebSocket: messages look like `event: x\ndata: {...}\n\n`, one way from server to client. After the first chunk the status is already 200, so on failure yield an `error` event inside the stream, and always send a `done` event at the end so the client can tell a complete answer from a dropped connection. `EventSource` only supports GET, so POST chat endpoints use `fetch` with `res.body.getReader()` and a buffer split on "\n\n", because network chunks do not respect message boundaries.

### Tool use, MCP and the agent loop
**Learned:** 2026-10-19 · **Related:** Vector database (RAG), Embeddings and cosine similarity, Chunking (RAG) · **Reviewed:** 2026-10-19 (quiz passed)
**Review questions:** Who executes a tool call, and what does the model produce instead of a final answer? How does the backend know the model wants a tool (stop_reason == "tool_use" plus a tool_use block with name, arguments and id)? Is MCP something the model uses, or a protocol between the client app and a separate tool server (the model only sees ordinary tool definitions)? Why does the loop run on the backend rather than in the browser? Watch out: MCP vs. tool.

The model never runs code or touches a database; it only sees a tool's name, description and parameter schema. When it wants a tool it replies with `stop_reason == "tool_use"` and a `tool_use` block (name, arguments, id). The agent loop is code in your backend: call the model, run the requested function on tool_use, send back a `tool_result` with the same id, and repeat until the model answers normally. MCP is a standard for packaging tools on a separate server so any compatible client (e.g. Cursor) can use them; the model still sees ordinary tool definitions.

### Vector database (RAG)
**Learned:** 2026-10-19 · **Related:** Embeddings and cosine similarity, Chunking (RAG) · **Reviewed:** 2026-10-19 (quiz passed)
**Review questions:** What are the two phases of RAG, and why does RAG not need retraining? Why does vector search match "get my money back" to a "Refund policy" document when keyword search fails? In indexing, what order do chunking, embedding and storing happen in, and why (not embed first, then chunk)? At query time, what happens to the question (embed it, nearest-neighbour search for top-k chunks, the model answers from them)? Watch out: the order of indexing steps.

RAG (Retrieval-Augmented Generation) is open-book: retrieve the few relevant chunks and paste only those into the prompt. Indexing (once) embeds chunks and stores vectors plus text in a vector database (Chroma, Pinecone, pgvector, Qdrant); querying embeds the question, fetches the top-k nearest chunks by similarity, and the LLM answers from them. Retrieval quality limits answer quality, and RAG reduces but does not eliminate hallucination, so tell the model to say "I don't know" when the context lacks the answer.

## Deployment and Infrastructure

### Docker (images, containers, Dockerfile)
**Learned:** 2026-10-26 · **Related:** Virtual environment (venv) and pip, Background work in FastAPI (BackgroundTasks vs. job queues), FastAPI architecture (router, service, repository) and lifespan · **Reviewed:** not yet
**Review questions:** Why is a container not a VM, and what do namespaces vs. cgroups each control? Why copy requirements.txt and run pip install before copying the code in a Dockerfile, and why must data that must survive go in a volume?

Docker packages the app with everything it needs (Python version, libraries, system packages, config) so it runs the same anywhere, while a venv only isolates Python packages. A Dockerfile is the recipe, the built image is the read-only package, and a container is a running instance (many containers can come from one image, like objects from a class). A container is not a VM but an ordinary process on the host kernel: namespaces limit what it sees (filesystem, PIDs, network), and cgroups limit what it uses (CPU, RAM such as `--memory=2g`). Layers are cached, so copy requirements.txt and pip install before the code; use `-p` to map ports, `--host 0.0.0.0` so the app is reachable from outside the container, and `-e` for secrets rather than baking them into the image.

## Machine Learning Fundamentals

### Batch
**Learned:** 2026-10-05 · **Related:** Epoch
**Review questions:** Why use mini-batches instead of processing the whole dataset at once? What is the cost of more frequent updates?

A part of the data. We use batches because of memory restrictions, and we can move faster with smaller steps (more weight updates per epoch) than with one large step.

### Epoch
**Learned:** 2026-10-05 · **Related:** Batch, Training vs. validation
**Review questions:** What is the difference between a batch, a step, and an epoch?

The moment when the whole training data was seen—the model went over all batches once.

### Gradient
**Learned:** 2026-10-05 · **Related:** Loss, zero_grad and backward
**Review questions:** What does a gradient measure? In which direction does the optimizer move the weight?

Measures how a change of a weight affects the loss. The optimizer moves each weight in the opposite direction of its gradient (weight - lr * gradient).

### Loss
**Learned:** 2026-10-05 · **Related:** Gradient, Overfitting
**Review questions:** What do we want to happen to loss during training, and why is exactly 0 suspicious?

Tells how incorrect the model is. We want it as low as possible during training (not literally 0, which usually means memorization).

### Overfitting
**Learned:** 2026-10-05 · **Related:** Training vs. validation, Loss
**Review questions:** When training loss falls but validation loss rises, what is happening? Name two fixes.

The model memorized the training data: training loss keeps dropping while validation loss rises, so more training only makes it memorize. Fixes include new data, a smaller model, early stopping, or regularization.

### Training vs. validation
**Learned:** 2026-10-05 · **Related:** Overfitting, Epoch
**Review questions:** What is different about training and validation? Why do we need both?

Training updates the model's weights. Validation uses examples the model hasn't seen yet, to check how well it generalizes, with no weight updates.

### zero_grad and backward
**Learned:** 2026-10-05 · **Related:** Gradient
**Review questions:** Why call zero_grad before backward? What does backward calculate?

backward calculates new gradients; zero_grad resets the gradients first because backward adds onto existing ones, so old gradients would pile up across batches.

## Python

### Decorators (Python)
**Learned:** 2026-10-07 · **Related:** —
**Review questions:** What does `@name` above a function expand to? Why does `@app.get("/predict")` have parentheses while `@timer` does not, and how many nested functions does each need?

A decorator is a function that takes a function and returns a new wrapped version; `@name` is shorthand for `func = name(func)` and works because functions are values in Python. A plain decorator uses two layers (takes the function, returns the wrapper), while a decorator factory like `@repeat(3)` uses three nested layers (the outer function holds the setting, the middle takes the function to decorate, and the inner wrapper runs on each call); the setting is remembered via closure. FastAPI's `@app.get("/predict")` is a decorator factory that registers a function as the handler for an HTTP route.

### Generators (Python)
**Learned:** 2026-10-09 · **Related:** Batch, Decorators (Python)
**Review questions:** What is the difference between `return [..]` and `yield` in a function? When would you prefer a generator over a list, and what can't you do with one? How does a data loader relate to generators and batches?

A generator is a function that uses `yield` instead of `return`: it hands back one value, then pauses and remembers where it stopped, continuing when the next value is requested (like a C# method with `yield return`, or a GetNext() without allocating the whole collection). Unlike a list, the values are produced lazily one at a time, so memory use stays small; the trade-offs are that a generator can only be looped over once and cannot be indexed. Prefer generators for big data or item-by-item processing, e.g. a 50 GB log file (`for line in f` already reads lazily) or a training data loader that yields one batch at a time (PyTorch DataLoader), which connects to batches existing because of memory limits.

### GIL (Global Interpreter Lock)
**Learned:** 2026-10-09 · **Related:** Threads vs. processes (Python)
**Review questions:** What does the GIL allow and not allow when running several threads? Why can `counter += 1` still produce wrong results with the GIL, and how do you fix it? Why does the GIL hurt NumPy/PyTorch work so little?

The GIL is a lock inside CPython that lets only one thread run Python bytecode at a time, even on multi-core CPUs. It protects the interpreter's internals (e.g. reference counts) from memory corruption, but it does NOT prevent race conditions in user code: a thread switch can happen between bytecode instructions, so `counter += 1` (read, add, write; not atomic) can lose updates. Fix with threading.Lock (`with lock: counter += 1`). The GIL only restricts Python bytecode: NumPy/PyTorch heavy maths runs in C/C++/CUDA code that releases the GIL, so it barely hurts AI workloads.

### Threads vs. processes (Python)
**Learned:** 2026-10-09 · **Related:** GIL (Global Interpreter Lock), Batch
**Review questions:** For downloading 50 web pages vs. a heavy pure-Python calculation, which gets threads and which gets processes, and why? What is the cost of using processes?

Because of the GIL, threads only help for I/O-bound work (network calls, disk reads, API requests): a thread waiting on I/O releases the GIL so another thread can run, and the waits overlap. For CPU-heavy pure-Python work threads just take turns, so use multiprocessing: each process has its own interpreter and GIL and runs truly in parallel on separate cores, at the cost of heavier startup and copying data between processes. In AI projects threads help with data downloading, file reading and many concurrent LLM/model API calls; running several batches in threads on one GPU gains little since the GPU is the bottleneck; PyTorch DataLoader with num_workers uses worker processes for CPU-heavy preprocessing.

### Virtual environment (venv) and pip
**Learned:** 2026-10-07 · **Related:** —
**Review questions:** If you run pip install without activating the venv, where does the package go and what breaks? Why share requirements.txt instead of the .venv folder?

A venv is a private folder containing its own Python interpreter and package installations, isolating a project's dependencies so different projects can use conflicting versions without interfering (e.g., one using fastapi 0.100, another using 0.115). Activating the venv makes python and pip point to the venv's copies; without activation, pip installs into the global Python, causing 'module not found' errors and polluting other projects. requirements.txt (from pip freeze) records exact versions so others or Docker can recreate the environment; commit it but never the .venv folder itself.

## Security and Authentication

### API key authentication (FastAPI)
**Learned:** 2026-10-22 · **Related:** FastAPI basics (endpoints, Pydantic validation, Depends), Streaming responses (FastAPI, SSE), async def vs. def (FastAPI) · **Reviewed:** not yet
**Review questions:** Why does a missing or wrong key get 401 instead of 403, and why does the endpoint body (and the LLM call) never run? Why one key per client instead of a shared key, and why use `compare_digest` and store only hashes?

Authentication answers "who is calling?", authorization "what may they do?". Without a key, anyone who finds the URL can call your LLM endpoint and you pay for every call. The client sends a long random secret in a header (e.g. `X-API-Key`); an `APIKeyHeader` plus `Depends` check compares it with `hmac.compare_digest` and raises `HTTPException(401)` before the endpoint body runs. Use `dependencies=[Depends(check)]` on the decorator for guards that only block, and a `Depends` parameter when the function needs the caller's identity; keys belong in env vars or a secret store, only hashes in the DB, always over HTTPS, and one key per client lets you revoke or rate-limit a single client.

### JWT (JSON Web Token)
**Learned:** 2026-10-26 · **Related:** API key authentication (FastAPI), FastAPI basics (endpoints, Pydantic validation, Depends) · **Reviewed:** not yet
**Review questions:** How does the server know a JWT is genuine without storing it in a database (it recomputes the signature with its secret key and compares), and why can a user not change `role` from user to admin? Does each user get a different token and a different secret, and if Alice sends Bob's token, who does the server think she is? Why is a JWT hard to revoke, and name three ways to handle it (short expiry plus a revocable refresh token, jti denylist, token version or rotating the secret as the emergency option). Watch out: a JWT is signed, not encrypted, so anyone can read the payload and it must never hold secrets; it proves the token is genuine, not who is holding it.

A JWT is a signed token `header.payload.signature` (base64url) carrying claims such as `sub`, `role` and `exp`. The server signs it with a secret (HS256 is an HMAC of header and payload) and on each request recomputes the signature and compares, so no per-token database lookup is needed (stateless). Each user's token differs because the payload differs, but the server uses one shared secret; it is sent as `Authorization: Bearer <token>` and checked by a `Depends` guard that raises 401. The trade-off is that it is hard to revoke, since the token carries its own claims, so use short-lived access tokens with a revocable refresh token, a jti denylist, a token_version check, or secret rotation in an emergency; RS256 lets other services verify with a public key without being able to forge tokens.

### Refresh token rotation
**Learned:** 2026-10-26 · **Related:** Refresh tokens (JWT), JWT (JSON Web Token) · **Reviewed:** not yet
**Review questions:** What changes on every `/refresh` call, and why does that make a stolen refresh token detectable? How does the server react when an already-used token is presented again? Watch out: rotation does not stop a thief who uses the stolen token first; it limits the theft to one use and makes it visible.

Every `/refresh` call returns a new refresh token and marks the old one as used, so each refresh token works only once. Without rotation, a stolen refresh token stays valid for its whole lifetime, and the real user's own refreshes hide the theft. With rotation, presenting an already-used token means someone holds a copy, so the server revokes the whole token family (all tokens from that login) and both the thief and the real user must log in again, whichever side reuses the old token second. It adds a little server state (used/revoked flags) to the otherwise stateless refresh token design.

### Refresh tokens (JWT)
**Learned:** 2026-10-26 · **Related:** JWT (JSON Web Token), API key authentication (FastAPI) · **Reviewed:** not yet
**Review questions:** Why does sliding expiration (a new token with a later `exp` on every request) not solve revocation, and what does a refresh token do differently? Who holds the refresh token, who calls `/refresh` after a 401, and when is the user actually sent to the login page? Watch out: revoking the refresh token does not kill an already-issued access token immediately; it only stops new ones.

A refresh token is a second, long-lived credential (days to weeks) stored hashed in the server's DB and sent only to a `/refresh` endpoint, while the access token is a short-lived (5-15 min) JWT sent with every request. When the access token expires the server returns 401, and the client (usually a fetch wrapper or axios interceptor) calls `POST /refresh`; the server checks the stored hash (exists, not revoked, not expired), returns a new access token, and the client retries the request, so the user is sent to login only if `/refresh` itself fails. Revocation works by deleting the refresh token row, so a stolen access token dies within minutes and cannot be renewed, whereas sliding expiration fails because an attacker who keeps using the token keeps extending it. The trade-off is keeping a little server state (refresh tokens) so that everything else stays stateless.

### Token storage (httpOnly cookies, XSS and CSRF)
**Learned:** 2026-10-26 · **Related:** Refresh tokens (JWT), Refresh token rotation, JWT (JSON Web Token), API key authentication (FastAPI) · **Reviewed:** not yet
**Review questions:** Where can the browser keep tokens (localStorage, memory, httpOnly cookie) and what is each one's main risk? What is CSRF, and which cookie flag stops it (SameSite: Strict, Lax or None)? Watch out: httpOnly stops token theft, not XSS itself, since injected JS can still make requests that carry the cookie; and a cookie without Max-Age is a session cookie that is gone after the browser closes, so `/refresh` returns 401 the next day.

Browser options are localStorage (any injected script can read and steal the token), memory (lost on reload, fine for a short-lived access token) and an httpOnly cookie (page JavaScript cannot read it at all, while the browser attaches it automatically). A typical setup keeps the access token in memory and the refresh token in `Set-Cookie: refresh=...; HttpOnly; Secure; SameSite=Strict; Path=/refresh; Max-Age=2592000`, and on page load the app calls `POST /refresh` with `credentials: "include"` to get a new access token silently, without a password prompt. SameSite makes the browser skip the cookie on cross-site requests, which is the CSRF defence, so a forged request arrives without it and gets 401. Staying logged in needs both the cookie to exist in the browser (Max-Age) and the server to accept the token (DB row not expired or revoked, JWT signature and `exp` valid).
