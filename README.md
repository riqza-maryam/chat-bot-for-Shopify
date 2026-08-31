<img width="2172" height="724" alt="image" src="https://github.com/user-attachments/assets/3949539e-2cbd-4ffc-b0fb-0b21cfc36c49" />

# Chatbot for Shopify

The **chatbot for shopify** is a storefront support tool that ties a new-order event to the customer context used in later conversations. It listens for [Shopify webhooks](https://shopify.dev/docs/apps/build/webhooks/subscribe), prepares a personalized onboarding context, accepts the customer’s question, and sends that question with the configured brand knowledge to the language-model step. The result is deliberately limited to two terminal paths: return an answer in chat, or create a human handoff in Slack. It does not edit orders, issue refunds, change inventory, or perform other store actions.
<img width="2172" height="724" alt="image" src="https://github.com/user-attachments/assets/5d35d4af-c891-4d5b-b2ad-0ea675a9db90" />
<p align="center">
  <a href="https://t.me/"><img src="https://img.shields.io/badge/Telegram-Chat-blue?style=for-the-badge&logo=telegram" alt="Telegram"></a>
  <a href="https://wa.me/"><img src="https://img.shields.io/badge/WhatsApp-Chat-25D366?style=for-the-badge&logo=whatsapp&logoColor=white" alt="WhatsApp"></a>
  <a href="mailto:hello@cogworklabs.com"><img src="https://img.shields.io/badge/Email-hello@cogworklabs.com-D14836?style=for-the-badge&logo=gmail&logoColor=white" alt="Email"></a>
  <a href="https://cogworklabs.com"><img src="https://img.shields.io/badge/Website-Cogwork%20Labs-000000?style=for-the-badge&logo=google-chrome&logoColor=white" alt="Website"></a>
</p>

## What Runs When an Order Arrives

The workflow starts with the `orders/create` topic. Shopify documents this event as a webhook subscription that pushes an order payload to the configured endpoint rather than requiring continuous polling. The handler validates the request, extracts the customer and order fields needed by the support flow, and passes that context into the onboarding stage. The onboarding stage does not create a new commerce action; it prepares the customer-specific context that later makes a support question less generic.

When the customer asks something, the conversation handler combines three inputs: the current question, the available order context, and the approved brand-knowledge files. The model call uses the [OpenAI Responses API](https://platform.openai.com/docs/quickstart/make-your-first-api-request) with retrieval from the configured knowledge source. The router then checks the model result and produces one of two states: `ANSWER` or `HUMAN_HANDOFF`. A handoff is posted through [Slack incoming webhooks](https://api.slack.com/messaging/webhooks), where a person can pick up the unresolved case.

## Core Features

| Feature | Description |
| :--- | :--- |
| **Order-event context** | Generic support replies lose useful purchase context. The webhook handler captures the new-order event and makes the relevant order context available to the conversation flow. |
| **Brand-knowledge retrieval** | A model without a bounded source can invent policy or product details. The response step is given the configured knowledge files before it answers. |
| **Two-state routing** | Uncertain questions should not be forced into an automatic reply. The router returns either `ANSWER` or `HUMAN_HANDOFF` as an explicit outcome. |
| **Slack escalation** | Unresolved cases disappear when there is no destination for them. The handoff branch posts the customer question and available context to a configured Slack channel. |
| **Storefront conversation path** | Support should remain usable where the customer asks the question. The chat flow accepts the message, carries context through the model step, and returns the selected outcome. |
| **Environment-based secrets** | API keys and webhook secrets do not belong in source control. Runtime credentials are loaded from environment variables rather than committed configuration. |

The feature boundary matters more than a long capability list. This repository is built around answering from known material and escalating the rest. That makes the failure mode legible: a missing policy document is a knowledge problem, a rejected webhook is an integration problem, and an unanswered case is a routing problem. Those are much easier to debug than one opaque “AI support” feature that hides every step behind a single request.
<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/04b24ecb-598f-4cc3-bfb3-773c16b722fd" />

---

# Chatbot for Shopify

## Answer or Human Handoff

The routing decision is the safety valve in the workflow. A direct answer is returned only after the question has been processed with the available customer context and brand knowledge. The alternative is not a fallback sentence pretending to help; it is a different output path. `HUMAN_HANDOFF` packages the unresolved question with the context the workflow already has and posts it to the support channel. Slack recommends keeping incoming-webhook URLs secret, so the repository treats that URL as an environment variable rather than a checked-in value.

This split is also the most important behavior to test before connecting a live store. Use one question clearly covered by the knowledge files and one deliberately absent question. The covered question should finish on `ANSWER`; the unsupported question should finish on `HUMAN_HANDOFF` and create a Slack message. Broader service research is useful context for why this boundary matters — see Zendesk's *CX Trends 2026* and Salesforce's *State of Service 2025*. The repository itself, however, is validated by its branch behavior rather than industry averages.

<p align="center">
  <img src="https://pub-cb2f0eca5bf044e5a45ef184f2ccf85d.r2.dev/posts/automation/cdh-src-b2e3714169594aac.gif" alt="Demo Animation" width="100%">
</p>

---

## How to Run Support Using Chatbot for Shopify

1. **Download & Set Up the Project** — Download, set up, and install chatbot for shopify from this repository, then copy the example environment file and install the project dependencies.
2. **Open the Storefront Chat** — Start the local app, open the connected storefront preview, and use the chat interface that sends customer messages into the conversation handler.
3. **Configure Context and Handoff** — Set the shop credentials, model API key, Slack webhook URL, and brand-knowledge source defined in `.env`, then load the approved knowledge files.
4. **Send a Test Question** — Submit a customer question. The run returns an answer to chat or emits `HUMAN_HANDOFF` and posts the unresolved case to Slack.

```bash
cp .env.example .env
npm install
npm run dev
```

Before using a production shop, register the order webhook against the running app and test the endpoint with a development store. The current Shopify webhook subscription documentation shows `ORDERS_CREATE` as a supported GraphQL topic and lets the subscription point at an HTTPS endpoint. Keep the webhook secret and every third-party credential outside the repository.

---

## Tech Stack and Runtime Boundaries

| Component | Role in this project |
|---|---|
| Node.js runtime | Runs the application and keeps webhook, conversation, routing, and notification code in one typed codebase. |
| Shopify webhooks | Starts the order-context path when a new order event reaches the app endpoint. |
| OpenAI API | Processes the customer question with the supplied context and configured knowledge retrieval. |
| Slack incoming webhook | Receives unresolved questions from the human-handoff branch. |
| Environment variables | Hold store credentials, model credentials, webhook secrets, and runtime configuration outside committed code. |

The model connection is server-side. The official OpenAI data controls documentation is the right place to review current retention behavior for API endpoints before choosing what store or customer data may be sent. The project should be configured with the minimum fields required for the support use case. Order context belongs in the request only when it helps answer the question; unrelated customer data does not improve the response and should not be added merely because it is available.

---

## Project Directory

shopify-support-chatbot/
app/
  routes/
    webhooks.orders-create.ts
    api.chat.ts
  services/
    shopify.server.ts
    knowledge.server.ts
    assistant.server.ts
    router.server.ts
    slack.server.ts
  types/
    conversation.ts
    handoff.ts
knowledge/
  README.md
  policies.md
  products.md
tests/
  orders-create.test.ts
  routing.test.ts
  handoff.test.ts
.env.example
shopify.app.toml
package.json
tsconfig.json
README.md

The directory separates event intake, answer generation, routing, and notification so each failure can be reproduced independently. `webhooks.orders-create.ts` owns the incoming order event. `api.chat.ts` accepts the question. The service files isolate store access, knowledge loading, model calls, route selection, and Slack posting. Tests mirror the same boundaries: an order-event test checks ingestion, a routing test checks both terminal states, and a handoff test checks the outbound payload without requiring the full storefront path.

---

## Performance and Reliability Checks

A successful local start is not enough. The useful verification is behavioral.

1. Send a valid order webhook and confirm the handler accepts it.
2. Ask a question directly supported by `knowledge/policies.md` or `knowledge/products.md` and confirm the chat path returns `ANSWER`.
3. Ask a question intentionally missing from those files and confirm the route becomes `HUMAN_HANDOFF`.
4. Verify that the Slack payload contains the unresolved question and only the customer/order context intended for a human reviewer.

Then test the failure cases one layer at a time: invalid webhook authentication, a missing model key, an unavailable knowledge file, and a rejected Slack webhook. The expected result is an explicit error at the responsible boundary, not a silent success. Shopify's webhook model is event-driven, so local verification should also include duplicate delivery handling and idempotent processing where the handler records or recognizes an already-seen event. The goal is not to benchmark response speed with invented numbers; it is to prove that each input reaches the correct output and that unsupported questions never masquerade as grounded answers.

---

## Use Cases

- **Answer policy questions from approved material.** A shopper asks about a rule already present in the knowledge files, and the chat path returns the grounded answer instead of forcing a support agent to repeat it.
- **Carry purchase context into support.** A customer asks a question after ordering, and the workflow can include the relevant order context already captured by the new-order event.
- **Escalate unsupported questions with context.** A question falls outside the configured knowledge, so the bot stops at `HUMAN_HANDOFF` and sends the case to Slack rather than inventing an answer.
- **Test knowledge changes before release.** A practitioner updates a policy or product document, runs covered and uncovered questions locally, and checks that the branch outcomes still match the intended support boundary.


## Frequently Asked Questions

**What does the bot use as its source of truth?**  
It uses the brand-knowledge files configured for the project, together with the customer question and any relevant order context captured by the workflow. The model step is not treated as the source of truth by itself; the repository is structured so the approved knowledge remains the material the answer is expected to follow.

**What happens when the bot cannot answer confidently?**  
The conversation takes the HUMAN_HANDOFF branch instead of forcing an automatic reply. That branch sends the unresolved question and available context to the configured Slack destination, making uncertainty a visible routing outcome rather than a hidden model guess.

**Which credentials are required to run the project?**  
The runtime needs the store/app credentials used by the Shopify integration, the model API credential, the webhook verification secret, and the Slack incoming-webhook URL. They belong in environment configuration and should not be committed to the repository; Slack's webhook documentation specifically treats the webhook URL as a secret.
These cases use the same pipeline; only the question and knowledge coverage change. That is why the scope stays narrow: one path returns a supported response and one path sends the case for human review. For repository maintenance, the checks remain concrete — update the knowledge source, verify the route, and inspect the outbound handoff when coverage is incomplete.
