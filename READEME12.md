# Day 12: Saving AI requests in a database

This guide explains `note12` in beginner-friendly language. The big change today is that the AI Gateway stops forgetting request history when it restarts.

## The problem this lesson solves

Before this lesson, the app could receive and simulate an AI request, but it did not keep a permanent record of it. If the app stopped or restarted, its request history disappeared.

This lesson connects the app to **PostgreSQL**, a database that stores information on disk. After that, saved requests are still there when the app starts again.

The app's AI response is still simulated. This lesson saves the request and its result; it does not connect to a real AI provider.

## How the pieces work together

Think of the flow like a team passing a form along:

1. A user sends JSON to the FastAPI app.
2. FastAPI checks the request using the Pydantic schemas from Day 11.
3. The service prepares a simulated result.
4. SQLAlchemy turns Python instructions into database commands.
5. PostgreSQL saves a row containing the request and result details.
6. FastAPI sends a response back to the user.

```text
User -> FastAPI -> SQLAlchemy -> PostgreSQL
                    |
              Python objects
```

### The three database terms

- **PostgreSQL** is the database. It stores the information.
- **SQLAlchemy** is the Python toolkit and ORM. It helps the app work with database tables using Python objects.
- **Psycopg** is the driver that lets Python connect to PostgreSQL.

An **ORM** (Object Relational Mapper) maps Python classes and objects to database tables and rows. You can ask SQLAlchemy to save a Python object, and it creates the appropriate SQL operation behind the scenes.

## A quick look at SQL

SQL is the language commonly used to work with relational databases. The lesson introduces these operations:

- `CREATE TABLE` makes a new table.
- `INSERT` adds a row.
- `SELECT` reads rows.
- `UPDATE` changes existing rows.
- `DELETE` removes rows.

The lesson uses SQLAlchemy for the app's database work instead of writing each SQL command by hand.

## The database table

The lesson creates an `ai_requests` table. Each row represents one saved request. It includes information such as:

| Column | What it stores |
| --- | --- |
| `id` | A database-generated number for the row |
| `request_id` | The unique ID returned by the API |
| `model` | Which AI model was requested |
| `provider` | The model's provider, if known |
| `prompt` | The text the user sent |
| `status` | The result status |
| `input_tokens` | An estimate of input tokens |
| `output_tokens` | An estimate of output tokens |
| `total_tokens` | The combined token estimate |
| `created_at` | When the record was created |

The Python class `AIRequestRecord` describes this table. It is an ORM model, not the same thing as the AI model names (such as `gpt-4o`) listed in the app's configuration.

A **primary key** identifies a row. A **unique** `request_id` makes sure two rows cannot accidentally have the same API request ID.

## Database connection and keeping passwords safe

The app reads its connection information from a `.env` file using `python-dotenv`. The lesson's connection string has this general shape:

```text
DATABASE_URL=postgresql+psycopg://postgres:YOUR_PASSWORD@localhost:5432/ai_platform_db
```

Replace `YOUR_PASSWORD` with your own local database password. Do not copy a real password into this guide, source code, or a file you plan to publish.

The lesson adds `.env` to `.gitignore`. That tells Git not to include the file in commits. Keep this protection in place: a database password is a secret. If a password is ever accidentally published, change it rather than relying on deleting the file afterward.

The database setup also creates:

- an **engine**, which manages the connection to the database;
- a **session**, which is a short-lived workspace for reading and writing records;
- `get_db()`, which gives an API endpoint a session and closes it when the endpoint is finished.

## Saving and reading records

The service in `services/request_service.py` has three important jobs:

- `save_ai_request(...)` adds a record, commits it to make it permanent, and refreshes it to read back database-generated values.
- `get_all_requests(...)` returns recent records, with a limit so the app does not fetch an unlimited number at once.
- `get_request_by_id(...)` looks for one record using its request ID.

`commit()` saves the current transaction. If a database operation fails, `rollback()` cancels the unfinished transaction so the session can recover. The API catches database errors when saving and returns an error instead of pretending the save worked.

## The new API routes

The lesson keeps the existing model and prompt routes and adds two ways to look up saved requests:

| Route | What it does |
| --- | --- |
| `POST /requests` | Validates and processes a request, saves its record, and returns the result |
| `GET /requests/history` | Lists saved requests; `limit` can be between 1 and 200 |
| `GET /requests/{request_id}` | Finds one saved request by its request ID |
| `POST /requests/validate` | Checks a prompt, as in Day 11 |
| `GET /models` | Lists the available AI models |
| `GET /models/{model_id}` | Looks up one available AI model |

The history route must be defined before the route with `{request_id}`. Otherwise, FastAPI could mistake the word `history` for a request ID.

The history response includes the saved prompt, status, token estimates, and creation time. Pydantic's `from_attributes=True` setting lets the response schema read values directly from a SQLAlchemy object.

## Creating the table when the app starts

When FastAPI starts, the lesson's lifespan code runs `Base.metadata.create_all(...)`. SQLAlchemy uses the imported ORM model to create the table if it is missing.

This is convenient for a learning project, but it does **not** update an existing table when the model changes. Later, database migrations (for example, with Alembic) are used to manage those changes safely.

## Set it up and try it

The lesson assumes PostgreSQL is installed on Windows and that a database named `ai_platform_db` has been created. From the project folder:

1. Make sure the PostgreSQL server is running.
2. Create the `ai_platform_db` database if it does not exist.
3. Activate your Python virtual environment.
4. Install the packages:

   ```powershell
   python -m pip install sqlalchemy "psycopg[binary]" python-dotenv
   ```

5. Put your local connection string in `.env`, and confirm `.env` is listed in `.gitignore`.
6. Start the API:

   ```powershell
   fastapi dev main.py
   ```

7. Open `http://127.0.0.1:8000/docs` to try the endpoints in Swagger.

Submit a valid request with `POST /requests`, then call `GET /requests/history`. Copy its `request_id` and try `GET /requests/{request_id}`. You can also use pgAdmin's Query Tool to run `SELECT * FROM ai_requests;`.

To prove the data is persistent, stop and restart FastAPI, then check the history again. Your saved rows should still be there because PostgreSQL stores them independently of the running app.

## Common connection problems

- **Connection refused**: PostgreSQL may not be running.
- **Password authentication failed**: Check the password in `.env`.
- **Database does not exist**: Make sure `ai_platform_db` was created and the name matches.
- **`No module named sqlalchemy` or `psycopg`**: Install the missing package in the active virtual environment.
- **`DATABASE_URL` is missing**: Check that `.env` is in the project root and contains the setting.
- **Table does not exist**: Make sure the ORM model is imported before `create_all()` runs.

## What to remember

- PostgreSQL keeps records after the app restarts.
- SQLAlchemy lets Python code work with database rows as objects.
- A database session is where a group of reads and writes takes place.
- `commit()` saves changes; `rollback()` cancels unfinished changes after an error.
- Keep passwords in environment configuration, and never publish `.env`.
- This lesson simulates AI output; it is not yet calling a real AI model.

## Practice ideas

The lesson suggests building routes to:

1. Filter saved records by model.
2. Filter saved records by status.
3. Calculate totals such as the number of requests and tokens used.
4. Submit several requests, restart FastAPI, and confirm the records remain.

The next lesson builds on this foundation with database updates, relationships, filtering, pagination, and usage statistics.
