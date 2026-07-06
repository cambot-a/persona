# Persona

Persona is an n8n workflow for a personalized French newsletter assistant. It collects subscriber preferences through chat, stores them in an n8n Data Table, researches relevant news, writes an HTML email, sends it through email tooling, and provides an unsubscribe webhook.

The workflow is exported in [`Persona.json`](./Persona.json).

## Screenshots

<img width="708" height="564" alt="image" src="https://github.com/user-attachments/assets/c7c3d8d0-c095-4be1-8a85-3ab45f7ca0bd" />
<img width="1312" height="594" alt="image" src="https://github.com/user-attachments/assets/78577246-a7bd-4109-8411-631632cbb954" />

| Area | Placeholder | Suggested final image |
| --- | --- | --- |
| Full workflow canvas | [`workflow-overview.md`](./assets/screenshots/workflow-overview.md) | `workflow-overview.png` |
| Chat subscription flow | [`chat-subscription-flow.md`](./assets/screenshots/chat-subscription-flow.md) | `chat-subscription-flow.png` |
| Generated newsletter email | [`newsletter-email.md`](./assets/screenshots/newsletter-email.md) | `newsletter-email.png` |
| Unsubscribe page | [`unsubscribe-page.md`](./assets/screenshots/unsubscribe-page.md) | `unsubscribe-page.png` |

When the real screenshots are ready, place them in `assets/screenshots/` and update this section to reference the image files directly.

## What It Does

- Starts a conversational signup flow from an n8n chat trigger.
- Collects a user's name, email address, interests, and preferred send time.
- Stores subscriber data in an n8n Data Table named `User Infos`.
- Lets users update their profile information through chat.
- Lets users unsubscribe by saying `STOP` or by opening an unsubscribe link.
- Searches the web for news matching each user's interests.
- Generates a styled HTML newsletter in French.
- Sends confirmation and newsletter emails.
- Runs a scheduled job that checks which subscribers should receive a newsletter at the current time.

## Workflow Architecture

The workflow has four main paths.

### 1. Chat Signup And Account Management

The chat path starts at `When chat message received` and passes the message to `ChatBot`.

`ChatBot` is a LangChain agent backed by an Ollama chat model and short-term memory. It is instructed to speak French and collect:

- `userName`
- `userEmail`
- `userInterests`
- `sendHour`
- `sendMinute`

The agent has access to these tools:

- `Get rows` to look up existing subscribers.
- `Insert row` to create a new subscriber.
- `Update rows` to modify subscriber information.
- `Delete rows` to unsubscribe a user.
- `Send subscribe notification` to send HTML confirmation emails through Gmail.

### 2. Scheduled Newsletter Delivery

`Schedule Trigger` runs on a minute interval. It calls `Get row(s)` to retrieve subscribers whose stored send hour and minute match the current execution data.

Matching rows are passed into a research and writing chain:

1. `research Agent` searches for relevant recent news using `SerpApi1`.
2. `email writing agent` turns the research output into a complete HTML newsletter.
3. `HTML` cleans markdown fences from the generated HTML.
4. `Send an Email` sends the rendered newsletter through SMTP.

### 3. Direct Newsletter Agent

The workflow also contains a `Newsletter agent` path with:

- `Google search in SerpApi`
- `Ollama Chat Model`
- `Send news in Gmail`

This agent is configured to search for five articles based on the subscriber interests and send a French HTML newsletter through Gmail.

### 4. Unsubscribe Webhook

The public unsubscribe path starts at:

```text
GET /webhook/unsubscribe?email=<subscriber-email>
```

The webhook deletes the matching row from `User Infos`, then returns either:

- a success message when the subscriber was removed, or
- an error message when the email was not found.

## Data Model

The workflow expects an n8n Data Table named `User Infos`.

| Column | Type | Description |
| --- | --- | --- |
| `userName` | string | Subscriber display name. |
| `userEmail` | string | Subscriber email address. Used as the lookup key. |
| `userInterests` | string | Topics used to personalize newsletter research. |
| `sendHour` | number | Preferred delivery hour. |
| `sendMinute` | number | Preferred delivery minute. |
| `subscribed` | boolean | Optional subscription state column included in the schema. |
| `emailFrequency` | number | Optional update field currently referenced by the update tool. |

## Required Services

To run the workflow, configure these services in n8n:

- n8n with chat, webhook, schedule, Data Table, email, and LangChain nodes available.
- Ollama with the model used by the workflow, currently `llama3.2:latest`.
- SerpAPI credentials for web search.
- Gmail credentials for Gmail-based confirmation/newsletter tools.
- SMTP credentials for the `Send an Email` node.

## Credentials Referenced By The Export

The exported workflow includes credential references from the original n8n instance. These IDs will not work after import unless the target n8n instance has matching credentials.

Replace or reconnect these credentials after importing:

| Credential name in export | Used by |
| --- | --- |
| `personality_ai` | ChatBot Ollama model |
| `ollama-research-agent` | Newsletter and research Ollama models |
| `Ollama email writing agent` | Email writing Ollama model |
| `research_account` | SerpAPI tools |
| Gmail credential | Gmail send tools |
| `SMTP account` | SMTP newsletter send node |

## Setup

1. Import [`Persona.json`](./Persona.json) into n8n.
2. Reconnect all credentials in the imported workflow.
3. Create or map the `User Infos` Data Table with the columns listed above.
4. Make sure Ollama is running and that `llama3.2:latest` is installed.
5. Configure SerpAPI, Gmail, and SMTP credentials.
6. Review the email sender and recipient fields in `Send an Email`.
7. Update webhook URLs if the workflow is not running on `localhost:5678`.
8. Activate the workflow when configuration is complete.

## Usage

### Subscribe Through Chat

Open the n8n chat trigger and send a French signup request. Example:

```text
Bonjour, je veux recevoir une newsletter. Je m'appelle Camille, mon email est camille@example.com, je m'intéresse à l'IA et aux startups, et je veux la recevoir à 9h30.
```

The assistant should collect any missing information before inserting the row.

### Update Subscriber Information

Ask the assistant to update stored information. Example:

```text
Je suis Camille, je veux changer mes centres d'intérêt pour IA, productivité et santé.
```

The assistant is expected to look up the existing profile and update it.

### Unsubscribe Through Chat

Send:

```text
STOP
```

The assistant should identify the subscriber, delete the stored row, and send a confirmation email.

### Unsubscribe By Link

Open:

```text
http://localhost:5678/webhook/unsubscribe?email=camille@example.com
```

Replace the base URL when running n8n on another host.

## Important Configuration Notes

- The workflow is exported with `"active": false`; activate it manually in n8n.
- The unsubscribe links currently point to `http://localhost:5678`. Change this before using the workflow outside local development.
- The scheduled search prompt includes `actualités juin 2026`. Update this wording if the workflow should always search for current news.
- The SMTP node currently contains fixed sender and recipient addresses from the original export. Review these before sending real emails.
- The workflow contains both Gmail and SMTP sending paths. Decide whether both are needed in your deployment.
- The chat prompt requires the assistant to avoid exposing raw database rows or tool JSON.

## Repository Structure

```text
.
├── Persona.json
├── README.md
└── LICENSE
```

## Development Notes

This repository stores the n8n workflow export rather than application source code. Most changes should be made in n8n, exported again, and reviewed in `Persona.json`.

Before committing an updated workflow export:

- Remove test subscriber data from pinned or sample data.
- Check that no private credential values were exported.
- Verify webhook URLs match the intended environment.
- Confirm hard-coded email addresses are intentional.
- Take fresh screenshots and update the screenshot section.

## License

This project is licensed under the MIT License. See [`LICENSE`](./LICENSE).
