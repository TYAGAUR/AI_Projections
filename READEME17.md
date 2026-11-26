# Day 17: Handling slow AI requests

This guide explains `note17` in beginner-friendly language. The lesson teaches three ways to keep an AI app useful when model calls take time:

1. **Async requests** let the server handle other work while waiting for the AI provider.
2. **Streaming** sends answer pieces as they arrive.
3. **Background tasks** let the user start a long job and check on it later.

## Async does not make the AI think faster

A normal request can wait for the full AI response before finishing. While it waits, a blocking operation can prevent the server from handling other work.

With asynchronous code, the server can pause one request while it waits for a network response, then work on another request. In Python:

- `async def` defines a coroutine that can pause while waiting.
- `await` pauses until an async operation is ready, without blocking the event loop.
- `asyncio.gather()` can wait for several async operations together.

This improves how well the server handles overlapping network waits. It does **not** make the AI model generate its answer faster.

The lesson adds an `AsyncOpenAI` client and an async service that awaits the network call. It also shows an async route at `POST /requests/async`.

One limitation: the example still uses a synchronous SQLAlchemy database session. Those database operations can block the event loop. The lesson keeps that setup to teach the concept; a more complete async application would use async database operations or move blocking work to a thread pool.

## Streaming: show the answer as it is written

Without streaming, the user waits for the complete answer. With streaming, the app sends small pieces of text as soon as they arrive, like text appearing gradually in a chat.

The lesson uses **Server-Sent Events (SSE)**. The `POST /requests/stream` route returns a `text/event-stream` response with events such as:

- `token`: a piece of generated text;
- `completed`: the generation has ended and usage totals are available;
- `error`: generation failed.

Streaming is useful when people benefit from seeing an answer right away. This lesson's streaming example does not save the completed response to PostgreSQL.

## Background tasks: start now, finish later

Some work, such as analyzing a large document, may take a long time. Keeping the user's web request open for the whole job is not ideal.

Instead, the app can:

1. Accept the request and create a task ID.
2. Return `202 Accepted` and the task ID quickly.
3. Process the job after the response.
4. Let the user check its status later.

The lesson adds:

- `POST /requests/background` to submit the task;
- `GET /requests/background/{task_id}` to check its status.

Possible states include `queued`, `processing`, `completed`, and `failed`. The example also checks that the task belongs to the logged-in user.

### Important: the background example is for learning

It stores task information in a Python dictionary in the server's memory. That means task information disappears if the app restarts, and multiple server processes do not share the same task list.

FastAPI's `BackgroundTasks` is useful for small, in-process work, but it is not a durable job system for large workloads. A production design usually stores task status in a database and sends jobs to a queue such as Redis or RabbitMQ for dedicated workers to process.

## Which approach should you use?

| Situation | A good fit |
| --- | --- |
| A regular request that waits on an AI network call | Async request |
| A chat answer that should appear gradually | Streaming |
| A long document analysis that should finish later | Background job |
| Many jobs across multiple servers | Durable queue and dedicated workers |

These approaches solve different problems. Streaming changes how output is delivered; background processing changes when the client waits for the result.

## Try the lesson

Start the app from the project folder:

```powershell
fastapi dev main.py
```

Authenticate with the Day 14 login flow in Swagger before trying protected routes.

1. **Async:** send a prompt to `POST /requests/async`. It should return a complete AI response and token usage.
2. **Streaming:** send a prompt to `POST /requests/stream`. The response should arrive as several SSE events. The lesson includes a small Python client to print the events as they arrive.
3. **Background:** send a prompt to `POST /requests/background`. Save the returned task ID, then check `GET /requests/background/{task_id}` until the task finishes.

Background tasks may finish quickly, so the first status you see can vary.

## Other useful terms

- **Concurrency**: multiple tasks make progress during overlapping time periods.
- **Parallelism**: tasks literally run at the same time, often on separate processor cores.
- **Timeout**: the maximum time an operation is allowed to wait.
- **Retry**: trying an operation again after a temporary failure.
- **Backpressure**: limiting incoming work so it does not overwhelm the system.
- **Idempotency**: making retries safe so the same action does not accidentally happen twice.

Async work is not a complete scaling solution by itself. A real system also needs to consider database limits, provider rate limits, timeouts, retries, and CPU-heavy work.

## Key ideas to remember

- Use async for network waits so the server can work on other requests in the meantime.
- Use streaming when users want to see output as it is generated.
- Use background jobs when work takes too long for one open request.
- Store important job state somewhere durable; in-memory state disappears on restart.
- Use a proper queue and worker processes for large, distributed workloads.

The next lesson introduces embeddings and vector databases, which are building blocks for semantic search and Retrieval Augmented Generation (RAG).
