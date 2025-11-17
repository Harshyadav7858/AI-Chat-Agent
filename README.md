# Personal AI Expert Lounge

A Spring Boot + Spring AI (Google GenAI) application that gives you multiple domain-specific AI experts behind a single, luxury chat UI. Responses stream live like ChatGPT and are tailored by expert persona (sports, medical, AI interview, education, and a dedicated Java expert).

---

## Table of Contents

- [Overview](#overview)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
  - [Backend](#backend)
  - [Frontend UI](#frontend-ui)
- [Expert Personas](#expert-personas)
- [Streaming Chat with WebFlux](#streaming-chat-with-webflux)
- [Bullet-Point Answer Formatting](#bullet-point-answer-formatting)
- [UX Features](#ux-features)
- [Configuration](#configuration)
- [How to Run](#how-to-run)
- [How to Use](#how-to-use)
- [Extending the App](#extending-the-app)

---

## Overview

This project is a **personal multi-expert AI assistant** built with **Spring Boot** and **Spring AI** using **Google GenAI** as the underlying model. It provides:

- Multiple domain experts (Sports, Medical, AI Interview, Education, Java).
- A modern, glassmorphism-based chat UI in the browser.
- **Streaming responses** over **Server-Sent Events (SSE)** so answers appear token-by-token like ChatGPT.
- Answers formatted as **bullet points** to improve readability and structure.

The focus is on **persona control** (different behavior per expert) and **high-end UX** rather than just a plain chatbot.

---

## Tech Stack

- **Java / Spring Boot**
- **Spring AI** with **Google GenAI Chat Model**
- **Spring WebFlux** (for SSE streaming)
- **Thymeleaf** (server-side HTML template)
- **Vanilla JavaScript** for chat logic and streaming
- **CSS (custom, glassmorphism style)** for the luxury UI

---

## Architecture

### Backend

Main backend components:

- `ChatService` / `ChatServiceImpl`
- `ExpertPromptProvider`
- `ChatController`
- `ChatStreamController`
- `UiController`

#### ChatService & ChatServiceImpl

`ChatService` is a simple service interface:

```java
public interface ChatService {
    String chat(String expert, String query);
}
```

`ChatServiceImpl` implements it using **Spring AI's ChatClient**:

- Injects `ChatClient.Builder` and `ExpertPromptProvider`.
- For every request:
  - Computes a **system prompt** based on the selected expert using `ExpertPromptProvider`.
  - Calls the model via `chatClient.prompt().user(query).system(systemPrompt).call().content()`.
  - Returns a single non-streaming response string.

This service is used by `ChatController` for the `/chat` endpoint (non-streaming, simple HTTP response) and gives a unified place for persona prompts.

#### ExpertPromptProvider

`ExpertPromptProvider` encapsulates all expert-specific system prompts. It maps an `expert` key (e.g. `sports`, `medical`, `java`) to a strong instruction string passed as **system message** into the model:

- Ensures **bullet-point formatting**.
- Enforces domain boundaries (e.g. medical answers must be informational with a safety tone).
- Defines tone and level of detail for each expert.

Example mapping (simplified):

- `sports` / `cricket`: world-class sports analyst, bullet-point insights.
- `medical`: medical information assistant, simple language, safety reminders, bullet points.
- `ai-interview`: AI/ML interview mentor, conceptual explanations & follow-up questions in bullet points.
- `education`: study & exam coach, plans and strategies in bullet points.
- `java`: **Java expert only** – answers in Java (code and APIs), explains concepts in bullet points with modern Java style.

This design keeps persona logic isolated and easy to extend.

#### ChatController

- `GET /chat`
- Query params:
  - `expert` (optional, default `general`)
  - `q` (required: user query)
- Delegates to `ChatService.chat(expert, q)`.
- Returns a full response string (non-streaming). Useful for testing and potentially non-stream UIs.

#### ChatStreamController (WebFlux SSE)

- `GET /chat-stream`
- Produces `text/event-stream` (SSE).
- Query params:
  - `expert` (optional, default `general`)
  - `q` (required: user query)
- Steps:
  1. Use `ExpertPromptProvider` to compute the system prompt for the given `expert`.
  2. Build `SystemMessage` + `UserMessage`.
  3. Call `googleGenAiChatModel.stream(systemMessage, userMessage)` to get a `Flux<String>`.
  4. Spring WebFlux exposes this as SSE, so the browser receives chunks progressively.

This endpoint powers the **ChatGPT-like streaming** in the frontend.

#### UiController

- `GET /`
- Adds the initial `expert` (default `general`) into the model.
- Returns the `index` Thymeleaf template which renders the chat UI.

### Frontend UI

The UI lives in:

- `src/main/resources/templates/index.html`
- `src/main/resources/static/css/luxury.css`
- `src/main/resources/static/js/chat-ui.js`

Key ideas:

- Left panel: expert selection and preset question buttons.
- Right panel: streaming chat window + input row.
- Styling: glassmorphism, gradients, rounded chips, hover animations.

#### index.html

Main elements:

- **Expert cards** (`.expert-card`): buttons with `data-expert` attribute:
  - Sports
  - Medical
  - AI Interview
  - Education
  - Java Expert (coding persona that always answers in Java)
- **Preset prompts** under each expert:
  - Example: quick prompts for sports (match summary, player analysis).
  - Example: quick prompts for medical (flu overview, lifestyle changes).
  - These appear as small pill buttons (`.preset-btn`).
- **Chat window** (`#chat-window`):
  - Contains a system hint message.
  - Contains a hidden `#typing-indicator` element for “Expert is thinking…” text.
- **Input row** (`#chat-form`):
  - Text input for user question.
  - Send button.
  - Regenerate button (`↻`) to repeat last question.

#### luxury.css

- Implements **dark, luxury glassmorphism style**:
  - Radial gradient background.
  - Glass panels for left and right.
  - Stylish cards for experts.
- Chat window styling:
  - Rounded corners, subtle border, gradient background.
  - `.chat-message.user` vs `.chat-message.assistant` classes for different alignment and background.
  - `.assistant-bullets` styles `<ul>` lists generated from bullet-point responses.
- Extra UX elements:
  - `.typing-indicator` with a pulsing dot animation.
  - `.preset-btn` pill buttons with hover effects.
  - `.regen-btn` with hover shadow.

#### chat-ui.js

This script powers the interactive behavior:

- Tracks state:
  - `currentExpert` (selected expert key).
  - `currentStream` (active SSE `EventSource`).
  - `lastQuestion` (for regenerate + Up arrow recall).
  - `lastRole` and `lastMessageEl` (for message grouping).
- Hooks into UI elements:
  - Expert cards (`.expert-card`) to change expert.
  - Preset buttons (`.preset-btn`) to auto-fill and send questions.
  - Typing indicator (`#typing-indicator`).
  - Regenerate button (`#regen-btn`).

Core behavior:

1. **Expert selection**

   - Clicking a card calls `setExpert(exp)`:
     - Updates `currentExpert`.
     - Updates selected expert text and chip text.
     - Adds expert-specific theme class to `body` (e.g. `expert-sports`, `expert-medical`, `expert-java`).
     - Triggers a chip animation (`chip-switching` CSS class) for a subtle visual transition.

2. **Sending a message**

   - On form submit or keyboard shortcut:
     - Append a user message bubble in the chat window.
     - Close any previous SSE stream if active.
     - Show typing indicator.
     - Create an empty assistant message bubble.
     - Open an `EventSource` to `/chat-stream?expert=...&q=...`.

3. **Streaming responses**

   - Every SSE `message` event appends text to a buffer.
   - `renderBullets(buffer)` converts the text into an HTML `<ul>` with `<li>`s.
   - The assistant message bubble’s `innerHTML` is updated live, so bullet points grow while streaming.
   - On error or completion, the typing indicator hides and stream is closed.

4. **Regenerate response**

   - `lastQuestion` stores the last submitted question.
   - Clicking `↻` puts `lastQuestion` back into the input and triggers form submit.

5. **Preset questions**

   - Each `.preset-btn` has `data-expert` and `data-prompt`.
   - Clicking a preset:
     - Calls `setExpert(expert)`.
     - Fills input with `data-prompt`.
     - Auto-submits the form.

6. **Message grouping**

   - `appendMessage(role, text)` and the streaming bubble use `lastRole` and `lastMessageEl`.
   - If the new message has the same role as the last one, the message gets `.grouped`, allowing the CSS to visually group consecutive assistant messages.

7. **Keyboard shortcuts**

   - `Ctrl+Enter`: sends the current message.
   - `Up arrow`: recalls the last question into the input (like some chat apps).
   - `/` (slash): focuses the input from anywhere in the page.

These small touches make the app feel more like a polished chat product.

---

## Expert Personas

Current expert keys and their roles:

- `sports` / `cricket` – **Cricket & Sports Guru**
  - Deep knowledge of players, matches, and tactical insights.
  - Always answers in bullet points.

- `medical` – **Health Information Guide**
  - Explains common conditions, symptoms, and precautions.
  - Always reminds user to consult a real doctor.
  - Bullet-point structure.

- `ai-interview` / `ai interview` – **AI/ML Interview Mentor**
  - Explains ML concepts, algorithms, and system design topics.
  - Can ask interview-style questions and evaluate answers.
  - Bullet-point structure.

- `education` – **Study & Exam Coach**
  - Helps plan studies, explain concepts, and suggest strategies.
  - Bullet-point guidance.

- `java` – **Java Expert**
  - Always answers using Java (code examples, APIs, best practices).
  - Encourages modern Java style (lambdas, streams, Optional).
  - Bullet-point explanations plus code snippets where needed.

Each expert is implemented via a specific system prompt inside `ExpertPromptProvider`.

---

## Streaming Chat with WebFlux

The app uses **Spring WebFlux** to expose an SSE endpoint `/chat-stream`:

- `ChatStreamController` returns `Flux<String>` from `GoogleGenAiChatModel.stream(...)`.
- `produces = MediaType.TEXT_EVENT_STREAM_VALUE` ensures the browser treats the response as SSE.
- The frontend uses `EventSource` to consume this stream and update the UI incrementally.

This pattern gives a **live-typing** feel similar to ChatGPT or other modern AI chat UIs.

---

## Bullet-Point Answer Formatting

To enforce structured output, all expert prompts explicitly instruct the model to:

- Use bullet points starting with `"- "`.
- Organize content into clear, scannable items.

On the frontend, the raw streaming text is:

1. Accumulated in a `buffer` string.
2. Split into lines.
3. Converted to `<li>` elements inside a `<ul class="assistant-bullets">`.

This ensures that even though the model is streaming plain text, the final rendering is a **proper HTML bullet list**, improving readability.

---

## UX Features

- **Expert cards** with hover effects and quick selection.
- **Preset prompts** per expert for fast use.
- **Streaming chat** with typing indicator.
- **Regenerate last answer**.
- **Message grouping** for cleaner dialog.
- **Keyboard shortcuts** for power users:
  - `Ctrl+Enter` = send.
  - `ArrowUp` in input = recall last question.
  - `/` = focus input.
- **Luxury glassmorphism design**:
  - Blurred panels.
  - Gradients and subtle glows.
  - Responsive layout.

---

## Configuration

You need a valid **Google GenAI / Gemini** API configuration compatible with Spring AI.

Typical setup:

- Add your Google GenAI API key as an environment variable or in `application.properties`, e.g.:

```properties
spring.ai.google.genai.api-key=YOUR_API_KEY_HERE
spring.ai.google.genai.chat.options.model=gemini-1.5-pro
```

Refer to the official Spring AI docs for the exact property names for the version you are using.

---

## How to Run

From the project root:

```bash
mvn spring-boot:run
```

By default, Spring Boot will start on port `8080` or as configured.

If you have changed server port (for example to `8086`), open:

```text
http://localhost:8086/
```

in your browser.

---

## How to Use

1. **Select an expert** from the left side: Sports, Medical, AI Interview, Education, or Java Expert.
2. Optionally, click a **preset prompt** to get a quick, ready-made question.
3. Type your own question in the input bar.
4. Press **Enter**, **Ctrl+Enter**, or click **Send**.
5. Watch the answer **stream in real-time** as bullet points.
6. Use the **Regenerate** button to ask the last question again (useful if you want a different variation).

For Java-specific help:

- Choose **Java Expert**.
- Ask programming questions, API usage, or request code examples.

---

## Extending the App

Some ideas for further extension:

- Add new expert personas (Finance coach, Resume reviewer, etc.).
- Implement a **settings panel** (light/dark, font size, per-expert temperature) stored in `localStorage`.
- Persist **multi-turn conversation history** per user.
- Add a **conversation list** on the left to switch between different chats.
- Integrate a **vector database** and build RAG-style experts with your own documents.

Because persona logic is centralized in `ExpertPromptProvider` and the UI is built around generic expert keys, adding new experts is straightforward:

1. Add a new case in `ExpertPromptProvider`.
2. Add an expert card in `index.html` with `data-expert="your-key"`.
3. Optionally, add presets for that expert.

---

Enjoy building and iterating on your **Personal AI Expert Lounge**! If you share this on GitHub, consider adding screenshots or GIFs of the streaming UI and expert selection panel to showcase the experience.
