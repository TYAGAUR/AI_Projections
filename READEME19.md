# Day 19: Teach the App to Read Documents

This guide explains Day 19 of the 45-day AI Platform Engineer plan in simple words. It is for a 15-year-old learner, not a large team of experts.

## The big idea

You already built search for small pieces of information on Day 18. Today, you will let the app read a real document:

1. Upload a PDF, DOCX, or TXT file.
2. Read the text inside it.
3. Cut the text into small passages.
4. Turn each passage into a number-based meaning (`embedding`).
5. Save the passages and meanings in ChromaDB.
6. Ask a question in normal language.
7. Find the passages that best match the question.

This process is the first part of **RAG**:

- **Retrieval:** finding useful passages.
- **Augmented Generation:** later, giving those passages to an AI model so it can write a grounded answer.

Today you build the retrieval pipeline. Day 20 will add the final answer-writing step.

## What you will create

The app will gain two useful API actions:

- `POST /documents/upload` - upload and index a document.
- `POST /documents/search` - search inside documents belonging to the logged-in user.

The app will accept:

- PDF: `.pdf`
- Microsoft Word: `.docx`
- Plain text: `.txt`

The maximum file size is **10 MB**.

## Simple example

Imagine a document says:

```text
FastAPI is a modern Python framework.
It supports asynchronous programming.
Pydantic validates API request data.
FastAPI generates interactive API documentation.
```

A student asks:

```text
How does FastAPI validate incoming API requests?
```

The app does not read the whole document as a human would. It finds the passage containing words or meaning closest to "Pydantic validates API request data," then returns that passage.

## Folder structure

Keep the existing project files. Add these new files and folders:

```text
AIPlatformGateway/
|-- api/
|   |-- __init__.py
|   |-- auth.py
|   |-- documents.py
|   |-- endpoints.py
|   |-- knowledge.py
|
|-- config/
|   |-- __init__.py
|   |-- llm_config.py
|   `-- vector_config.py
|
|-- schemas/
|   |-- __init__.py
|   |-- document_schema.py
|   `-- knowledge_schema.py
|
|-- services/
|   |-- __init__.py
|   |-- document_processor.py
|   |-- embedding_service.py
|   `-- vector_store.py
|
|-- chroma_db/
|-- main.py
|-- .env
`-- requirements.txt
```

## Step 1: Install document tools

Open the VS Code terminal inside the project and run:

```bash
python -m pip install pypdf python-docx python-multipart
```

What these packages do:

- `pypdf`: reads text from PDF files.
- `python-docx`: reads text from Word (`DOCX`) files.
- `python-multipart`: lets FastAPI receive uploaded files.

Add these lines to `requirements.txt`:

```text
pypdf
python-docx
python-multipart
```

Do not remove packages you already use, such as `fastapi`, `openai`, `chromadb`, or `numpy`.

## Step 2: Extract and split document text

Create `services/document_processor.py`.

This file has two jobs:

1. `extract_document_text()` reads PDF, DOCX, or TXT text.
2. `chunk_text()` splits the text into smaller passages.

The extraction code should:

- read every PDF page and remember its page number;
- read DOCX paragraphs;
- decode TXT files as UTF-8 text;
- reject unsupported file types;
- raise an error if no readable text is found.

Encrypted PDFs and scanned image PDFs are not supported yet. A scanned PDF may need OCR, which means using a tool that recognizes text inside images.

### Chunking

**Chunking** means splitting a document into useful, searchable sections.

The starting settings are:

| Setting | Starting value |
|---|---:|
| Chunk size | 500 words |
| Chunk overlap | 75 words |
| Search results (`top_k`) | 3 |

Example with 10-word chunks and 3-word overlap:

```text
Chunk 1: FastAPI is a modern Python framework. It supports asynchronous...
Chunk 2: It supports asynchronous programming. Pydantic validates API...
Chunk 3: Pydantic validates API request data. FastAPI generates...
```

The repeated words are intentional. **Overlap** keeps information together when a sentence or idea is split across two chunks.

### A simple chunk function

```python
def chunk_text(text: str, chunk_size: int = 500, overlap: int = 75) -> list[str]:
    if chunk_size <= 0:
        raise ValueError("chunk_size must be positive.")

    if overlap < 0 or overlap >= chunk_size:
        raise ValueError("overlap must be non-negative and smaller than chunk_size.")

    words = text.split()
    if not words:
        return []

    chunks = []
    start = 0

    while start < len(words):
        end = min(start + chunk_size, len(words))
        chunks.append(" ".join(words[start:end]))

        if end == len(words):
            break

        start = end - overlap

    return chunks
```

## Step 3: Test chunking

Create `test_chunking.py` in the project root:

```python
from services.document_processor import chunk_text

sample_text = """
FastAPI is a modern Python framework.
It supports asynchronous programming.
Pydantic validates API request data.
FastAPI generates interactive API documentation.
"""

chunks = chunk_text(sample_text, chunk_size=10, overlap=3)

for index, chunk in enumerate(chunks, start=1):
    print(f"\nChunk {index}:")
    print(chunk)
```

Run:

```bash
python test_chunking.py
```

Look for repeated words at the boundaries. They show that overlap is working.

## Step 4: Create embeddings

Use the existing `generate_embeddings()` function from Day 18:

```python
vectors = generate_embeddings([
    "First document passage",
    "Second document passage",
    "Third document passage",
])

print(len(vectors))
print(len(vectors[0]))
```

An **embedding** is a list of numbers that represents the meaning of a passage. Similar meanings produce vectors that are close together.

With the default `text-embedding-3-small` model, each vector has **1,536 numbers**.

## Step 5: Save chunks in ChromaDB

Open `services/vector_store.py`.

Keep the existing Day 18 functions and add document functions. The main steps are:

1. Split the chunks into batches of 100.
2. Create embeddings for every batch.
3. Give every chunk a unique ID.
4. Save its text and metadata in ChromaDB.

Useful metadata includes:

- `document_id`: identifies the uploaded document.
- `filename`: remembers the original file name.
- `user_id`: keeps one user's documents separate from another user's documents.
- `chunk_index`: identifies the chunk's position.
- `page_number`: identifies the source PDF page, when available.

### Batch indexing

The batch loop is:

```python
for start in range(0, len(chunks), 100):
    batch = chunks[start:start + 100]
    texts = [item["text"] for item in batch]
    embeddings = generate_embeddings(texts)
```

Processing in batches is safer and more efficient than sending an entire large document to the embedding service at once.

### Search filtering

When a user asks a question, the app:

1. converts the question into an embedding;
2. asks ChromaDB for the closest chunks;
3. filters by `user_id`;
4. returns the text, source, page, and similarity details.

The `where={"user_id": str(user_id)}` filter is important for privacy.

## Step 6: Create request and response models

Create `schemas/document_schema.py` using Pydantic models.

It should define:

- `DocumentUploadResponse`
- `DocumentSearchRequest`
- `DocumentSearchResponse`
- `RetrievedChunk`

The search question can contain 3 to 5,000 characters. `top_k` can be from 1 to 10.

Example request:

```json
{
  "question": "How does FastAPI validate incoming API requests?",
  "top_k": 3
}
```

Example upload response:

```json
{
  "document_id": "generated-uuid",
  "filename": "fastapi_guide.pdf",
  "total_chunks": 8,
  "status": "indexed"
}
```

## Step 7: Create document endpoints

Create `api/documents.py`.

### Upload endpoint

`POST /documents/upload` should:

1. require a logged-in user;
2. check that the file is a PDF, DOCX, or TXT;
3. read at most 10 MB of file data;
4. extract text;
5. split text into chunks;
6. assign a document ID;
7. save all chunks in ChromaDB;
8. return the document ID and chunk count.

The endpoint should return clear HTTP errors:

| Situation | Status |
|---|---:|
| Unsupported file type | 400 |
| Empty file | 400 |
| File larger than 10 MB | 413 |
| No readable text | 400 |
| Processing or indexing failure | 502 |

### Search endpoint

`POST /documents/search` should:

1. validate the question and `top_k`;
2. convert the question to an embedding;
3. retrieve matching chunks;
4. return only chunks owned by the logged-in user.

## Step 8: Register the router

In `main.py`, import the document router:

```python
from api.documents import router as documents_router
```

Then register it:

```python
app.include_router(documents_router)
```

Keep all earlier router registrations, including authentication and knowledge search.

Restart the FastAPI server after saving every file.

## Step 9: Test in Swagger

Start the app:

```bash
fastapi dev main.py
```

Open the FastAPI Swagger page, usually at:

```text
http://127.0.0.1:8000/docs
```

### Test A: Sign in

1. Open `/auth/token`.
2. Enter an existing username and password.
3. Click **Authorize**.

### Test B: Upload

1. Open `POST /documents/upload`.
2. Click **Try it out**.
3. Choose a small, text-based PDF, DOCX, or TXT file.
4. Run the request.

The response should include an ID, filename, chunk count, and `status: "indexed"`.

### Test C: Search

Open `POST /documents/search` and try:

```json
{
  "question": "How does FastAPI validate incoming API requests?",
  "top_k": 3
}
```

Each result includes:

- `chunk_id`: unique chunk ID;
- `document_id`: source document ID;
- `filename`: source file name;
- `chunk_index`: chunk position;
- `page_number`: PDF page, if available;
- `content`: retrieved text;
- `distance`: how far the chunk is from the question;
- `similarity_score`: estimated similarity.

Values vary. They are measurements, not guarantees.

### Test D: Verify persistence

1. Upload a document.
2. Search successfully.
3. Stop the server.
4. Restart the server.
5. Search for the same information.

The document should still be searchable because ChromaDB is using `PersistentClient`.

### Test E: Verify user isolation

1. Create or use a second account.
2. Sign in as that user.
3. Search for the first user's document.

The second user should get no results. This is called **tenant isolation** or **multi-user privacy**.

## What RAG is doing

```text
Upload document
      |
      v
Read text -> Split into chunks -> Create embeddings -> Save in ChromaDB
                                                           |
                                                           v
User asks a question -> Create query embedding -> Find matching chunks
                                                           |
                                                           v
                                                Return useful passages
                                                           |
                                                           v
                                      Day 20: LLM writes a grounded answer
```

RAG is useful because the AI does not need to receive every document in the company. It receives only the passages that are probably relevant to the question.

## Common errors

| Error | Likely cause | What to check |
|---|---|---|
| `422 Unprocessable Entity` | Invalid upload or JSON | Check Swagger form fields and JSON types |
| `413 Request Entity Too Large` | File is over 10 MB | Upload a smaller file |
| `No readable text found` | Empty file or scanned PDF | Use a text PDF or add OCR |
| `401 Unauthorized` | Missing or expired login | Sign in again |
| `502 Document processing failed` | API, embedding, or database failure | Read server logs and check configuration |
| `ModuleNotFoundError` | Package is missing | Install the package named in the error |

## Study questions

### What is RAG?

RAG finds relevant information outside the LLM and gives that information to the model as context for a grounded answer.

### Why split documents?

Large documents can exceed model limits. Small chunks are easier to search and send to a model.

### What is chunk overlap?

It is text repeated in neighboring chunks. It helps an idea continue correctly across a chunk boundary.

### Ingestion or retrieval?

- **Ingestion:** extract, clean, split, embed, and store a document.
- **Retrieval:** embed a question and find the closest stored chunks.

### Why save metadata?

Metadata identifies the source document, page, owner, and chunk. It allows filtering and source attribution.

## Interview answer: 10,000 documents

A strong answer is:

1. Upload documents to object storage.
2. Create ingestion jobs in a database.
3. Put jobs in a queue such as Redis or RabbitMQ.
4. Run worker processes.
5. Extract text and use OCR for scanned documents.
6. Chunk documents and create embeddings in batches.
7. Store vectors and metadata in a vector database.
8. Track jobs, failures, retries, and completion status.
9. Add tenant-aware authentication and authorization.

Partial-failure safety matters: if batch 2 fails after batch 1 is saved, the system should not leave a document half-indexed and labeled as complete.

## Completion checklist

- [ ] Installed `pypdf`, `python-docx`, and `python-multipart`.
- [ ] Extracted text from a PDF.
- [ ] Extracted text from a DOCX file.
- [ ] Run the overlapping chunking exercise.
- [ ] Generated embeddings for several chunks.
- [ ] Indexed document chunks in ChromaDB.
- [ ] Uploaded a document through Swagger.
- [ ] Retrieved passages with a natural-language question.
- [ ] Restarted the app and confirmed persistence.
- [ ] Tested isolation between two users.

Do not continue to Day 20 until you can upload a document and retrieve useful passages through Swagger.

## What's next: Day 20

Day 20 builds a complete RAG application:

1. receive a user's question;
2. create a query embedding;
3. retrieve matching document chunks;
4. combine the chunks into context;
5. call the OpenAI Responses API;
6. write an answer supported by the uploaded documents;
7. include source references;
8. explain when no relevant information was found.
