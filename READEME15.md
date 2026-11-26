# Day 15: Connecting your app to a real AI model

This guide explains the new Day 15 part of `note15` in beginner-friendly language. The file also repeats the Day 14 review; this guide focuses on the new feature: asking a real OpenAI model for a response.

## What changes today?

Until now, the app made up a sample response. After this lesson, a user can send a prompt to the app, and the app forwards it to an OpenAI model. The real response and token usage are then saved in PostgreSQL.

The full journey looks like this:

1. A user sends a prompt and a login token to FastAPI.
2. FastAPI checks the prompt and verifies the user's token.
3. The LLM service sends the prompt to OpenAI.
4. OpenAI returns generated text and token counts.
5. The app saves the request and usage under that user in PostgreSQL.
6. FastAPI sends the result back to the user.

## What is an LLM API?

An **LLM** (Large Language Model) is an AI system that can generate text. An **API** is a way for one program to ask another program to do something.

In this lesson, your app sends the user's prompt to OpenAI through its API. The prompt is the input; the generated answer is the output. The app does not run the AI model on your own computer.

## Keep the API key secret

To use the API, you need an API key from the OpenAI developer platform. API access may require billing or available credits; a ChatGPT subscription and API billing are separate.

Store the key in your local `.env` file, not in Python source code:

```text
OPENAI_API_KEY=your_actual_api_key_here
OPENAI_MODEL=your_available_model
OPENAI_TIMEOUT=30
```

The model name is configurable. Use one that your API project can access; the example in the lesson may not be available to every account.

Treat the API key like a password:

- Never put it in code, a public README, a response, or a log.
- Keep `.env` out of GitHub using `.gitignore`.
- If a key is accidentally exposed, revoke it and create a replacement.

## The OpenAI client and service

The lesson installs the OpenAI Python SDK and creates a reusable client in `config/llm_config.py`. That configuration reads the key, model name, and timeout from the environment.

The client uses a timeout so the app does not wait forever, and it can retry certain temporary failures. The separate `services/llm_service.py` handles the model call. Keeping this code separate from the API route makes the app easier to change or extend to other providers later.

The service uses OpenAI's Responses API:

- It sends the chosen model and the prompt.
- It reads the generated text from the response.
- It reads the input, output, and total token counts reported by the API.

These are real usage counts from the provider, unlike the rough word-based estimates in earlier lessons. A **token** is a small piece of text; it may be a whole word, part of a word, or punctuation.

## Why track token usage?

Models process text as tokens. The number of input and output tokens helps you understand how much the app is using the API and can help estimate costs. Different models can have different prices, so the lesson records usage but does not calculate a final cost.

## Request and response shapes

The new request schema accepts:

- `prompt`: required text, between 5 and 20,000 characters;
- `max_output_tokens`: optional output limit, defaulting to 500 and limited to 1 through 4,000.

The client does not choose an arbitrary model in this lesson. The app selects the model from configuration, which helps the platform control which model is used.

The response includes a request ID, model name, status, generated text, and input/output/total token counts.

## The updated `POST /requests` route

The lesson replaces the old simulated `POST /requests` endpoint with a real one. Do not add a second route with the same HTTP method and path.

This route requires a valid Day 14 login token. It:

1. Sends the prompt to the LLM service.
2. Saves the result, usage, and authenticated user's ID in PostgreSQL.
3. Returns the generated answer and usage details.

Because it is protected, log in first and send the token as a bearer token. Saving the user ID connects the request history to the person who submitted it.

## Try it in Swagger

Before testing, make sure:

- PostgreSQL is running and the app's database is configured.
- The required Python packages are installed in your virtual environment.
- `.env` contains a valid API key and a model available to your API account.
- `.env` is ignored by Git.

Start the app from the project folder:

```powershell
fastapi dev main.py
```

Then open `http://127.0.0.1:8000/docs`:

1. Log in using the Day 14 `/auth/token` endpoint.
2. Click **Authorize** and provide the bearer token.
3. Send a request to `POST /requests`, for example:

   ```json
   {
     "prompt": "Explain RAG architecture in simple terms.",
     "max_output_tokens": 500
   }
   ```

4. Check that the response contains generated text and token counts.
5. In pgAdmin, check the newest `ai_requests` record to confirm it includes the model, provider, user ID, and usage.

The response text, request ID, and token counts will vary.

## Errors and what to check

- **401 from OpenAI**: the API key may be missing, invalid, or revoked.
- **429**: the API project may have a rate limit or billing/quota issue.
- **404**: the configured model may not be available to your API project.
- **Connection error**: check your network, firewall, and timeout.
- **Missing API key**: check the `.env` filename, location, and variable spelling.
- **500 from your app**: check the server terminal for the underlying database or application error.

The app should give users a safe, simple error message. Keep secret keys and sensitive provider details out of responses and logs.

## Key ideas to remember

- A prompt is what you send; the generated response is what the model returns.
- The API key proves your app is allowed to call the provider, so protect it.
- The model is selected by app configuration in this lesson.
- The provider reports token usage; the app saves those counts in PostgreSQL.
- A bearer token identifies the logged-in user, and the saved record should belong to that user.
- The AI call can fail because of credentials, quota, network issues, or model availability.

## Practice ideas

After the basic workflow works, try:

1. Changing the system instructions the model receives.
2. Adding a configuration endpoint that shows the selected provider and model, but never the API key.
3. Limiting prompts to 10,000 characters.
4. Building a usage report for the signed-in user.
5. Measuring how long the provider call takes.

The next lesson explores prompt design and model settings. First, make sure this full workflow works: authenticated request in, real model answer out, and usage safely saved to the database.
