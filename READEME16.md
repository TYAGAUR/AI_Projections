# Day 16: Giving your AI app reusable instructions

This guide explains `note16` in beginner-friendly language. The lesson builds on Day 15, when the app first connected to a real OpenAI model.

## What this lesson adds

The app will become easier to control and update. It adds:

- Reusable instruction templates for different kinds of tasks.
- Model aliases that let the app choose a model configuration.
- Optional generation settings, such as temperature.
- A new endpoint for advanced requests.
- Structured JSON output for data-focused tasks.
- Prompt versions so you can tell which instructions were used.

## Prompt engineering: give the AI a clearer job

A prompt is the text and instructions sent to an AI model. A vague prompt like “Explain AI” could lead to many different kinds of answers.

A clearer prompt explains:

1. **Role**: what kind of helper the AI should act like.
2. **Context**: what background information it needs.
3. **Task**: what it should do.
4. **Output format**: how the answer should be organized.

For example, asking for a beginner-friendly explanation with a practical example and a short word limit gives the AI a clearer target.

## Different kinds of instructions

The lesson separates instructions into three groups:

- **System instructions** set broad behavior and platform rules.
- **Developer instructions** describe how this particular app should behave.
- **User input** is the question or task a person submits.

In the lesson's OpenAI Responses API code, application-level instructions are sent separately from the user's prompt. Keep internal instructions private; do not show them as if they were the user's content.

## Reusable prompt templates

Instead of copying different instructions into every API route, the lesson stores named templates in `prompts/templates.py`. Examples include:

- `general_assistant` for clear, useful answers;
- `technical_assistant` for architecture and engineering explanations;
- `code_assistant` for maintainable code and explanations.

A prompt service looks up a template by name and returns its instructions and version. If the name does not exist, the service reports an error instead of silently choosing a different template.

Templates make it easier to update and test how the AI behaves in one place.

## Model aliases and routing

The model router maps simple names such as `fast` and `quality` to model settings. An endpoint can use an alias without exposing every model ID to the person making the request.

This is called **model routing**. In a larger platform, the app might send simple tasks to a faster or cheaper model and complex tasks to a more capable one.

In this lesson, both aliases may point to the same actual model. The aliases are a starting point for configuration, not a guarantee that they use different models. Choose model IDs that are available to your API account.

## Model parameters

Parameters change how a model responds:

- `max_output_tokens` limits the size of the generated answer.
- `temperature`, when supported, affects how varied responses may be.
- The model choice can affect quality, speed, and cost.

Lower temperature values are often used for more consistent answers; higher values can allow more variety. These effects and ranges depend on the model.

Not every model supports every setting. The lesson tracks whether a model configuration supports temperature and only sends it when allowed. Keep that setting disabled unless you have confirmed the chosen model supports it. For initial tests, use `temperature: null`.

## The advanced request endpoint

The lesson adds `POST /requests/advanced`. It accepts a prompt and optional settings such as:

- `template_name`;
- `model_alias`;
- `max_output_tokens`;
- `temperature`.

The endpoint requires a logged-in user. It passes the selected template, model alias, and supported parameters to the LLM service, then returns details such as the response, model alias, template name, prompt version, and token usage.

Test it with the `general_assistant`, `technical_assistant`, and `code_assistant` templates. The instructions should change the style of the answers. If the aliases point to the same model, the differences come from the templates rather than different models.

## Structured JSON output

Sometimes an app needs data in a predictable shape instead of a paragraph. For example, a resume-analysis feature might need a candidate name, a list of skills, and a number of years of experience.

The lesson uses a Pydantic model to describe that shape, then asks the model for output that follows a JSON Schema. The app parses the JSON and validates it against the Pydantic model.

This helps software handle the result consistently, but it does **not** prove the answer is factually correct. A correctly shaped response can still contain mistakes. The app should use clear rules, handle refusals or incomplete responses, and check important facts before relying on them.

## Prompt versioning

Each template has a version, such as `1.0`. If you change its instructions, update the version. The app can then record which template and version produced a response.

This is useful when comparing old and new prompts: if an answer changes, you can see which instruction version was used. Keep prompt changes in source control and test them before release.

## Try the lesson

Start FastAPI from the project folder:

```powershell
fastapi dev main.py
```

Open `http://127.0.0.1:8000/docs` and authenticate using the Day 14 login flow. Then try `POST /requests/advanced` with a valid prompt, template name, and model alias.

Check that:

1. The selected template changes the answer's style.
2. The response includes the selected alias and prompt version.
3. An unknown template or model alias returns an error.
4. Unsupported temperature settings are not sent to the model.
5. Token usage and the request are saved as expected.

## Key ideas to remember

- Clear instructions help guide an AI response.
- Templates keep application instructions reusable.
- Model aliases let the app choose model configurations without exposing arbitrary model IDs.
- Only use parameters supported by the selected model.
- Structured output helps software read AI results, but does not guarantee that the results are true.
- Versioning prompts makes changes easier to track and compare.

## Practice ideas

The lesson suggests adding:

1. Templates for SQL help and resume help.
2. A `balanced` model alias.
3. An endpoint that lists template names and versions.
4. A structured endpoint for a business task.
5. Tests for invalid template names and model aliases.

Day 17 moves on to streaming responses, asynchronous work, and background tasks.
