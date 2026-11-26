# Day 11: Making AI requests safer with Pydantic

This guide explains the ideas in `note11` in beginner-friendly language. The lesson builds on a small FastAPI project and teaches it to check incoming AI requests before the app tries to process them.

## The big idea

Imagine an app lets people ask an AI model questions. A user sends information such as:

```json
{
  "model": "gpt-4o",
  "prompt": "Explain AI engineering",
  "max_tokens": 500
}
```

The app should check that the model and prompt were provided, that the values have the right types, and that numbers are within sensible limits. Otherwise, incorrect data could cause confusing errors later.

**Pydantic** is a Python library that checks data against rules you define. FastAPI can use those rules automatically when it receives JSON.

Think of a Pydantic model like a form with rules: it tells the app which answers are required, which are optional, and what counts as an acceptable answer.

## Important building blocks

### `BaseModel`: describe the form

A class that inherits from Pydantic's `BaseModel` describes the fields an incoming request should have:

```python
from pydantic import BaseModel

class AIRequest(BaseModel):
    model: str
    prompt: str
    max_tokens: int
```

Here, `model` and `prompt` must be strings, and `max_tokens` must be an integer.

### `Field`: add rules and defaults

`Field` lets you add extra rules to a field:

```python
from pydantic import Field

max_tokens: int = Field(default=500, ge=1, le=4000)
```

- `default=500` means the app uses 500 if the user leaves this field out.
- `ge=1` means the value must be greater than or equal to 1.
- `le=4000` means the value must be less than or equal to 4000.

For a required field, `Field(...)` uses `...` to say that the user must provide a value.

### Optional fields

```python
provider: str | None = None
```

This field can contain a string or `None`. Since it has a default of `None`, the user can leave it out.

### Custom validators: add your own checks

Some rules need more than a type or number range. The lesson uses a custom validator to remove extra spaces from the beginning and end of a prompt, then reject it if nothing is left.

For example, `"  Explain AI  "` becomes `"Explain AI"`, while `"     "` is rejected. This lesson uses Pydantic v2's `@field_validator`.

### Nested models: organize complicated JSON

A request can contain an object inside another object. For example, generation settings can be grouped together:

```json
{
  "model": "gpt-4o",
  "prompt": "Explain AI engineering",
  "generation": {
    "max_tokens": 1000,
    "temperature": 0.5
  },
  "metadata": {
    "application": "AI Gateway"
  }
}
```

The `generation` object can have its own Pydantic model and rules. This is called a **nested model**. `default_factory=dict` gives each request its own empty dictionary when metadata is not supplied.

## What the request fields mean

The lesson's `AIRequest` model includes:

| Field | Meaning |
| --- | --- |
| `model` | The ID of the AI model the user wants |
| `prompt` | The text or question sent to the model |
| `max_tokens` | An upper limit on the generated response size |
| `temperature` | A setting that affects how varied the model's output can be |
| `provider` | The company or service that provides the model; optional |
| `stream` | Whether the response should be sent gradually; defaults to `false` |

The lesson limits `temperature` to the range 0 to 2. Lower values usually mean more predictable responses, while higher values can allow more variety. The model provider may have its own supported settings and limits.

## How a request travels through the project

The lesson uses this project layout:

```text
AIPlatformGateway/
├── main.py
├── api/
│   └── endpoints.py
├── services/
│   └── model_service.py
├── schemas/
│   └── request_schema.py
├── config/
│   └── models_config.py
└── requirements.txt
```

Each part has a job:

1. **`schemas/request_schema.py`** describes accepted request and response data.
2. **FastAPI and `api/endpoints.py`** receive the HTTP request and use the schema to validate its JSON.
3. **`services/model_service.py`** checks the selected model and prepares a simulated result.
4. **`config/models_config.py`** contains the available model information.
5. **`main.py`** creates the FastAPI app and connects the routes.

In short: JSON arrives, the schema checks it, the endpoint calls the service, and FastAPI sends a response.

## What the lesson adds

- `POST /requests` accepts a validated AI request.
- `POST /requests/validate` checks a prompt and returns its character count.
- `GET /models` lists available models.
- `GET /models/{model_id}` looks up one model.
- The service checks that an optional provider matches the selected model.
- `AIResponse` describes the shape of the successful response.
- `PromptValidationRequest` and `PromptValidationResponse` describe the prompt-checking endpoint's input and output.

The lesson's service **simulates** a request; it does not send the prompt to a real AI provider. Its token counts are rough examples based on word counts, not actual model tokenization.

## Understanding the error codes

- **422 Unprocessable Entity**: the submitted data failed validation. For example, a temperature outside the allowed range or a prompt made only of spaces.
- **400 Bad Request**: the data passed basic validation, but an application rule was broken. In the lesson, that happens when the chosen provider does not match the model.
- **404 Not Found**: the requested model ID does not exist.

These are different kinds of problems, so the API uses different status codes to explain them.

## Try the lesson's tests

Start the app from the project folder:

```bash
fastapi dev main.py
```

Then open FastAPI's Swagger page at `http://127.0.0.1:8000/docs`. Swagger is an interactive page where you can try API routes and inspect their responses.

Try these cases:

1. Send a valid request to `POST /requests`. It should return a request ID, model, status, token estimates, and provider.
2. Send `temperature: 5`. It should fail with 422 because the allowed range is 0 to 2.
3. Send model `gpt-4o` with provider `Google`. It should fail with 400 if that model is configured as an OpenAI model.
4. Send a prompt containing only spaces. It should fail with 422.
5. Send a real prompt to `POST /requests/validate`. It should report that the prompt is valid and give its character count.

The request ID and token estimates can differ each time.

## Key words to remember

- **Schema validation**: checking whether data follows the rules and shape the app expects.
- **Serialization**: turning Python data into a format such as JSON to send elsewhere.
- **Deserialization**: turning incoming JSON into structured Python data.
- **Response model**: a schema that describes the data an API sends back.
- **Type annotation**: a hint such as `prompt: str` that says what kind of value is expected.

## Practice ideas

Once the examples make sense, try the lesson's assignments:

1. Add `top_p` to `AIRequest`, with a default of 1.0 and an allowed range from 0 to 1.
2. Make sure a prompt containing only spaces is rejected.
3. Add `GET /models/provider/{provider_name}` to list models from one provider.

The main lesson is that checking data near the entrance to an API makes problems easier to catch and gives users clearer feedback. In the next lesson, the project begins saving requests in a database.
