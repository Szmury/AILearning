# Roadmap: Python, SQL, asyncio, FastAPI, Docker, CI/CD, RAG, Kubernetes, AWS, Triton

Order: Python → SQL → asyncio → FastAPI → Docker → CI/CD → RAG → Kubernetes → AWS → Triton. Each builds on the last.

✅ = understood and saved in [GLOSSARY.md](GLOSSARY.md).

## 1. Python
- ✅ **Core:** types, lists/dicts, loops, functions, f-strings
- **Modules and packages:** imports, ✅ virtual environments, ✅ `pip`
- **Files and errors:** reading/writing files, JSON, `try/except`
- **Later:** ✅ decorators, context managers (`with`), ✅ generators, pytest
- **ML libraries (optional):** scikit-learn for classic ML (train/test split, simple models, metrics like accuracy, precision, recall) next to PyTorch; TensorFlow/Keras only as awareness (know what it is, no deep dive)

## 2. SQL
- **Core:** `SELECT`, `WHERE`, `ORDER BY`, `LIMIT`
- **Changing data:** `INSERT`, `UPDATE`, `DELETE`, primary keys
- **Aggregation:** `COUNT`, `AVG`, `GROUP BY`
- **Joins:** foreign keys, `JOIN` ← *most important*
- **SQLAlchemy:** SQL from Python; engine and sessions, defining tables as Python classes (ORM models), querying with `select()`, relationships (builds on joins and foreign keys), Alembic for migrations. Ties into the ✅ FastAPI database access step (sessions, `Depends`)
- **Later:** indexes, transactions

## 3. asyncio
- **Concepts:** blocking vs non-blocking, why waiting on I/O wastes time ← *most important*
- **Syntax:** `async def`, `await`, coroutines
- **Running tasks together:** `asyncio.gather`, `create_task` ← *most important*
- **Later:** timeouts, async HTTP/database clients, running blocking code in a thread

## 4. FastAPI
- **HTTP basics:** methods, status codes, JSON
- ✅ **First app:** routes, path/query params, `/docs`
- ✅ **Pydantic models:** request validation ← *most important*
- ✅ **Async vs sync endpoints:** when to use `async def` (builds on asyncio)
- ✅ **Database access:** sessions, `Depends` ← *most important*
- **API security:** ✅ authentication (API keys, JWT via `Depends`, refresh tokens and rotation, token storage); still to do: rate limiting, request size limits, protecting against abuse and runaway LLM costs
- ✅ **Background work:** `BackgroundTasks` for small fire-and-forget jobs, job queues (Celery, RQ, arq) for long ones: POST returns a job id, a worker runs it, GET polls the status (builds on async vs sync endpoints)
- **Middleware and CORS:** CORS (needed once a browser frontend calls your API, e.g. the streaming endpoint), logging, timing, request IDs
- **Later:** error handling, testing

## 5. Docker
- **Concepts:** image vs container ← *most important*
- **Basics:** `docker run`, ports, logs
- **Dockerfile:** build an image of your app ← *most important*
- **Later:** volumes, Docker Compose (app + database), image security (pinned dependencies, scanning images, running as a non-root user)

## 6. CI/CD
- **Concepts:** what continuous integration and continuous delivery/deployment are, and why automate tests and builds ← *most important*
- **First pipeline:** GitHub Actions workflow (YAML) that runs on every push: install dependencies, run pytest, lint ← *most important*
- **Docker in CI:** build the image of your app and push it to a registry (Docker Hub, GitHub Container Registry, later AWS ECR) ← *most important*
- **Secrets:** keeping API keys and cloud credentials in the CI secret store, never in Git
- **Later:** caching dependencies, test environments, automatic deploy to AWS, pipelines for ML (evaluating the model or RAG quality before release), image scanning in the pipeline

## 7. RAG with a vector database
- **Prompt engineering:** how to write prompts that give reliable results ← *most important*
  - **Structure:** system prompt vs. user prompt, clear role and task, stating the output format, giving the model an "I don't know" way out
  - **Examples:** zero-shot vs. few-shot prompting, step-by-step reasoning
  - **Structured output:** asking for JSON and validating it with Pydantic
  - **RAG prompts:** separating retrieved context from instructions, citing sources
  - **Iterating:** testing prompts on a small set of example questions, versioning prompts like code
- ✅ **Concepts:** why RAG (LLMs don't know your data, hallucinate), the retrieve → augment → generate flow ← *most important*
- ✅ **Embeddings:** text as vectors, similarity search (cosine similarity), embedding models
- ✅ **Chunking:** splitting documents into pieces, chunk size and overlap
- **Vector database:** ✅ concept and ANN search learned; still to do: storing and searching embeddings hands-on. Start with pgvector (builds on SQL and Docker), then compare Chroma/Qdrant/Pinecone ← *most important*
- **Pipeline:** ingest documents → embed → store; query → embed → top-k search → prompt the LLM with the retrieved context ← *most important*
- **Reranking:** retrieve a wide top-k (e.g. 50) with fast vector search, then rescore with a cross-encoder or rerank API and keep the best few (e.g. 5) for the prompt; bi-encoder vs. cross-encoder, the speed vs. accuracy trade-off (builds on ANN search and embeddings)
- **Serving it:** a FastAPI endpoint for ingestion and questions (async calls to the embedding/LLM API)
- **Guardrails:** checks around the LLM so it stays safe and on-topic; input checks (prompt injection, off-topic, PII), output checks (grounded in the retrieved context, format validation with Pydantic, toxic or leaked content), and what to do when a check fails (refuse, retry, fall back) ← *most important*
- **Guardrail frameworks:** Guardrails AI (validators on LLM output, structured output), NeMo Guardrails (NVIDIA; rules for topics and dialogue flow), LLM Guard (input/output scanners for injection, PII, toxicity), Llama Guard (a safety classifier model), Microsoft Presidio (detecting and masking PII); start with Pydantic plus one of these, then compare. Managed options come later with AWS: Amazon Bedrock Guardrails
- **RAG security:** ← *most important*
  - **Data poisoning:** malicious or false documents slipped into the corpus; ingestion checks, trusting the source of each document, storing metadata (who added what, when) so bad documents can be found and removed
  - **Indirect prompt injection:** instructions hidden inside retrieved chunks (a web page or PDF saying "ignore previous instructions"); treat retrieved text as data, not commands, separate it from instructions in the prompt, and give tools least privilege
  - **Access control on retrieval:** metadata filters so users only retrieve documents they are allowed to see
- **Later:** metadata filters, hybrid search (keyword + vector), evaluating retrieval quality (measure the gain from reranking), citing sources

## 8. Kubernetes
- **Concepts:** why orchestration, cluster, node, pod ← *most important*
- **Workloads:** Deployment (replicas, rolling updates), Service (stable address to reach pods) ← *most important*
- **Config:** ConfigMap, Secret, environment variables
- **Basics:** `kubectl` (apply, get, logs, describe), YAML manifests, local cluster (minikube/kind)
- **Later:** Ingress, resource requests/limits, autoscaling (HPA), Helm, GPU scheduling for AI workloads, security basics (RBAC, network policies, keeping Secrets out of images and Git)

## 9. AWS
- **Basics:** regions, IAM (users, roles, permissions) ← *most important*
- **Storage:** S3 for files and datasets
- **Compute:** EC2 (a server), then ECS (run containers) ← *most important*
- **Database:** RDS (managed PostgreSQL)
- **Later:** cost control (budgets/alerts), CloudWatch logs, SageMaker/Bedrock for AI

- **Later:** EKS (managed Kubernetes) to run what you learned in the Kubernetes step

## 10. Triton (NVIDIA Triton Inference Server)
- **Concepts:** why a dedicated model server instead of loading the model inside your FastAPI app ← *most important*
- **Model repository:** folder layout, `config.pbtxt`, supported formats (PyTorch/TorchScript, ONNX, TensorRT)
- **Running it:** Triton Docker image, HTTP/gRPC endpoints, health checks
- **Client:** sending inference requests from Python (`tritonclient`) or from FastAPI ← *most important*
- **Performance:** dynamic batching (builds on batches), multiple model instances on a GPU ← *most important*
- **Later:** model ensembles, metrics/monitoring, deploying Triton on Kubernetes/EKS, model supply-chain safety (only load trusted model files; untrusted pickle/TorchScript can run code)

## To read
- [ ] **Gbrain project**: read it and understand how it works (added 2026-10-13)

## Capstone
FastAPI app that serves a model and stores predictions in a database, run with Docker Compose, then deployed to AWS. Add a RAG endpoint that answers questions over your own documents using a vector database (pgvector in the same Compose stack). Stretch: move model inference into Triton and run the whole stack on Kubernetes.
