# Day 13: Managing and understanding saved AI requests

This guide explains `note13` in beginner-friendly language. It builds on Day 12, when the app first saved request records in PostgreSQL.

## What you will learn

By the end of this lesson, your app can:

- Create, read, update, and delete saved request records.
- Find requests by their model or status.
- Return request history in smaller pages.
- Connect users and requests in the database.
- Calculate totals, such as how many requests were made and how many tokens they used.

## CRUD: the four basic database actions

CRUD is a short name for four things an app commonly does with stored data:

| Letter | Action | Example in this project |
| --- | --- | --- |
| C | Create | Save a new AI request or register a user |
| R | Read | Show request history |
| U | Update | Change a request's status |
| D | Delete | Remove a request |

SQLAlchemy turns these jobs into Python operations. For example, an app can add a Python object to a database session, commit it, and SQLAlchemy handles the database command.

## Connecting users to their requests

Imagine one user submits several requests. A common database design has:

- a `users` table with one row per user;
- an `ai_requests` table with one row per request;
- a `user_id` column on each request that points to a user.

That `user_id` is a **foreign key**. It connects a request row to the user's `id`. One user can have many requests, which is called a **one-to-many relationship**.

The lesson adds SQLAlchemy relationships on both models so Python code can navigate between a user and their requests.

There is an important limitation in this lesson: the API can register and look up users, but `POST /requests` does not yet attach a request to a user. So requests created through that endpoint have no linked user (`user_id` is `NULL`). Authentication and linking requests to the signed-in user come later.

## Be careful when changing an existing database

On Day 12, the `ai_requests` table was created without a `user_id` column. Adding `user_id` to a Python class does not automatically add that column to a database table that already exists.

The lesson presents two options:

1. If this is a throwaway learning database and you do not need its data, back up anything important and recreate it with the new table definitions.
2. If you need to keep existing records, add the change with a database migration or the provided SQL.

For real projects, migrations (commonly managed with Alembic) are the safer way to change database structure. Do not drop a database containing data you need.

## CRUD operations in the service

The request service contains functions for:

- saving a request;
- getting one request or a list of requests;
- filtering by model or status;
- changing a request's status;
- deleting a request;
- creating a user and finding a user by ID.

Updating a record means loading it, changing a field, and committing the change. Deleting a record means asking the database session to remove it and committing.

The API returns `404 Not Found` when the request or user being looked up does not exist. Registering an email address that is already in use returns `409 Conflict`.

## Pagination: show a small page at a time

If a database has thousands of records, returning them all at once can be slow and hard to use. **Pagination** splits the results into smaller groups.

The request history endpoint uses two numbers:

- `limit`: the maximum number of records to return;
- `offset`: how many records to skip first.

For example:

| Query | Records shown |
| --- | --- |
| `limit=10&offset=0` | First 10 records |
| `limit=10&offset=10` | Next 10 records |
| `limit=10&offset=20` | Following 10 records |

The lesson sets safe bounds for the page size so a caller cannot request an unlimited number of records.

## Filtering requests

The API can narrow down the saved history:

- `GET /requests/model/{model_name}` returns requests for one model.
- `GET /requests/status/{status}` returns requests with one status.

Only these statuses are allowed by the lesson: `pending`, `processing`, `success`, `failed`, and `cancelled`.

## Usage analytics

The lesson adds two reports:

- `GET /usage` returns totals for requests and input, output, and combined tokens.
- `GET /usage/models` groups the totals by AI model.

SQLAlchemy's aggregate functions do the math in the database:

- `COUNT()` counts rows.
- `SUM()` adds numeric values, such as token totals.
- `GROUP BY` makes separate totals for each model.
- `COALESCE()` changes a missing total (`NULL`) to zero.

These token numbers are still simulated estimates from the earlier lessons. A real integration should use the token usage reported by the AI provider.

## The routes added or expanded in the lesson

| Route | What it does |
| --- | --- |
| `POST /users` | Register a user |
| `GET /users/{user_id}` | Find a user |
| `POST /requests` | Process and save a simulated AI request |
| `GET /requests/history` | List saved requests, with `limit` and `offset` |
| `GET /requests/model/{model_name}` | Filter requests by model |
| `GET /requests/status/{status}` | Filter requests by status |
| `PUT /requests/{request_id}` | Update a request's status |
| `DELETE /requests/{request_id}` | Delete a request |
| `GET /usage` | Show overall request and token totals |
| `GET /usage/models` | Show request and token totals for each model |

Routes such as `/requests/history` and `/requests/model/...` are placed before `/requests/{request_id}`. This helps FastAPI match the more specific paths instead of treating words like `history` as request IDs.

The app also keeps the earlier model-listing, prompt-validation, and health routes.

## Try it in Swagger

Start the app from the project folder:

```powershell
fastapi dev main.py
```

Open `http://127.0.0.1:8000/docs` to try the endpoints.

A useful practice sequence is:

1. Register a user with `POST /users`.
2. Submit an AI request with `POST /requests`.
3. Check `GET /requests/history`, then try changing `limit` and `offset`.
4. Filter by model and status.
5. Update a request's status with `PUT /requests/{request_id}`.
6. Delete a request and confirm that looking it up returns 404.
7. Check `GET /usage` and `GET /usage/models`.

You can inspect the database directly in pgAdmin, too. The lesson includes SQL examples for joining users to requests and grouping requests by model. At this stage, the user/request join will show requests without a linked user, because the API does not associate them yet.

## Production ideas to keep in mind

This is a learning project, not a finished production system. Before real users rely on it, consider:

- **Authentication**: verify who is using the app.
- **Authorization**: make sure users can only view or change records they are allowed to access.
- **Soft deletion**: mark a record as deleted instead of permanently removing it, if you need an audit trail.
- **Migrations**: manage database structure changes safely.
- **Transactions and errors**: handle database failures consistently.
- **Real usage data**: use token counts reported by the actual provider.

## Key ideas to remember

- CRUD means Create, Read, Update, and Delete.
- A foreign key connects a record to another table.
- A one-to-many relationship means one user can have many requests.
- Pagination uses a limit and an offset to return smaller result sets.
- Aggregates such as `COUNT()` and `SUM()` help create usage reports.
- User registration is not the same as authentication, and this lesson does not yet link requests to users.

## Practice ideas

Try adding:

1. A route that lists requests for a particular user.
2. A report that counts requests by status.
3. A report that groups requests by provider.
4. A complete CRUD test: create, read, update, delete, and verify the final database state.

Day 14 reviews the backend work from Week 2 and introduces authentication topics. The larger project will connect to real AI services later.
