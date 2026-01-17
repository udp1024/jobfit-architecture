# Architecture Overview

Here’s a compact but complete way to think about this—two tools, one profile, tightly constrained LLM.

## Quick comparison of low/no-cost LLM options

Provider | Model examples | Cost tier focus | Pros | Cons
--- | --- | --- | --- | ---
Google Gemini / Gemma | Gemini 2.0 Flash, Gemma 3 | Generous free tier | High quality, long context | Data use caveats, setup A B
Groq | Llama 3.x, Mixtral | Free, fast inference | Very fast, good open models | Rate limits, external service A C
OpenRouter | Many open models (Llama, Gemma) | Free + cheap | Model choice, unified API | Quotas, external dependency B C
Cloudflare Workers AI | Llama, Mistral | Free/low-cost | Edge deployment, low latency | Tied to Cloudflare stack A C
Ollama (self-hosted) | Llama, Mistral, Gemma, etc. | Local, “no API cost” | Full control, no data leaving | Needs your own server/CPU/GPU A C

> **Note:** For your use case—strictly profile-based answers, no external tools/search, low cost—I’d lean toward either:
>
> * Self-hosted Ollama with an 8–14B instruct model (Llama 3, Gemma, Mistral) if you’re okay running a small server; or
> * Groq or OpenRouter with a small Llama/Gemma instruct model and strict system prompts if you want a managed API.

## High-level architecture

### 1. Overall components

#### Frontend (Web UI)

* Tech: React/Next.js or similar.

##### Pages

* Job fit tool: Textarea for job description, “Evaluate fit” button, chat-like response area.
* Virtual interview tool: Input for interview question, chat-like response area.
* Contact gate: Simple form (name, email, org, role) shown before tools; only proceed if email passes org check.

#### Backend API

* Tech: Node.js (Express/Fastify) or Python (FastAPI).

##### Endpoints

* POST /auth/contact – validate email, store contact, issue session token.
* POST /job-fit – accept job description + session token, call LLM with your profile context.
* POST /interview – accept question + session token, call LLM with your profile context.
* GET /health – basic health check.

#### LLM layer

* Option A (managed): SDK/client for Groq/OpenRouter/Gemini/Gemma.
* Option B (self-hosted): Local HTTP endpoint (e.g., Ollama) running a chosen model.

#### Data layer

* Profile store: Your education, experience, projects as structured documents.
* Embedding store: Vector DB (e.g., PostgreSQL + pgvector, SQLite + embedding table) for RAG-style retrieval.
* Contact store: Relational DB (PostgreSQL/SQLite) for contact info and session tokens.
* Conversation store: Per-session chat history (last N turns) for continuity.

### Representing and persisting your profile context

#### Profile data model

Create a canonical profile document set, version-controlled (e.g., in Git):
 • profile/education.json
 • profile/experience.json
 • profile/projects.json
 • profile/summary.json (short narrative summary)
 • profile/constraints.json (things you don’t want the LLM to claim, e.g., “I have never worked with X”)
Each document has:
 • id
 • type (education, experience, project, summary, constraint)
 • title
 • body (rich text)
 • tags (skills, domains, tech stack, seniority, etc.)
On deployment/startup:

 1. Load all profile docs from disk.
 2. Generate embeddings for each body (using the same or a smaller embedding model).
 3. Store in vector DB with metadata (type, tags, etc.).
This gives you a fixed, explicit knowledge base that the LLM can be constrained to.

#### Restricting the LLM to your profile only

##### Prompting and context strategy

You want no external search, no tools, no browsing. That’s a combination of:
 • Model choice: Use a plain chat/completions endpoint with no tools/browsing enabled.
 • System prompt: Very explicit constraints.
 • Context construction: Only pass your profile snippets + user input + minimal chat history.

##### System prompt example (core idea)

For each request:

 1. Retrieve relevant profile snippets:
  ○ For job fit: use the job description as the query to the vector DB.
  ○ For interview: use the interview question as the query.
 2. Select top K snippets (e.g., 5–10) with highest similarity.
 3. Build the prompt:
  ○ System message: the constraint prompt above.
  ○ Context message: “Here are profile snippets about Salman:” + concatenated snippets.
  ○ User message: job description or interview question.
  ○ Optional: last 2–4 turns of conversation for continuity (but still within token budget).
Because the only knowledge the model sees is your profile snippets, it is effectively sandboxed to that domain—even if the model itself has broader training.

#### Tool 1: Job fit evaluator

##### Flow

 1. User passes contact gate (see next section).
 2. Frontend sends POST /job-fit with:
  ○ session_token
  ○ job_description
 3. Backend:
  ○ Validates session_token.
  ○ Runs embedding search on your profile using job_description.
  ○ Builds prompt:
   § System: constrained profile-only instructions.
   § Context: relevant education/experience/projects.
   § User: “Given the job description below, evaluate whether Salman is a good fit. Provide: (1) Yes/No/Partial fit, (2) Explanation referencing specific profile items, (3) Gaps or missing experience. Job description: …”
  ○ Calls LLM API.
  ○ Returns structured response (e.g., JSON with fit_score, fit_label, explanation, gaps).
 4. Frontend renders this in a chat bubble style.

#### Tool 2: Virtual interview Q&A

##### Flow

 1. User passes contact gate (same session).
 2. Frontend sends POST /interview with:
  ○ session_token
  ○ question
  ○ Optional conversation_id for multi-turn interviews.
 3. Backend:
  ○ Validates session_token.
  ○ Retrieves last N turns for conversation_id (if any).
  ○ Runs embedding search on your profile using question.
  ○ Builds prompt:
   § System: same constrained profile-only instructions.
   § Context: relevant profile snippets.
   § Conversation history: last N turns (assistant + user).
   § User: the new interview question.
  ○ Calls LLM API.
  ○ Stores the new turn in conversation history.
  ○ Returns answer.
 4. Frontend shows the answer in a chat interface.

#### Persisting context and sessions

##### Data persistence

 • Contacts table:
  ○ id, name, email, organization, role, created_at, is_org_email, is_blocked.
 • Sessions table:
  ○ id, contact_id, created_at, expires_at, status.
 • Conversations table:
  ○ id, session_id, tool_type (job_fit, interview), created_at.
 • Messages table:
  ○ id, conversation_id, sender (user, assistant), content, created_at.
 • Profile embeddings table:
  ○ id, doc_id, type, title, body, tags, embedding_vector.

##### This gives you

 • Per-user session context (for gating and rate limiting).
 • Per-conversation history (for multi-turn interviews).
 • Stable, versioned profile context (for retrieval).

##### Contact gating and organizational email filtering

##### Contact collection flow

 1. User lands on site → sees a short intro and a “Continue” button.
 2. Clicking “Continue” opens a contact form:
  ○ Fields: name, email, organization, role, purpose (optional).
 3. On submit, frontend calls POST /auth/contact.
 4. Backend validation
 • Basic validation:
  ○ Check email format with regex.
  ○ Normalize to lowercase.
 • Free-email blacklist:
  ○ Maintain a list of domains like gmail.com, yahoo.com, outlook.com, hotmail.com, icloud.com, etc.
  ○ If email domain is in this list → mark is_org_email = false.
 • Optional DNS/MX check:
  ○ For non-blacklisted domains, perform an MX lookup to ensure it’s a real domain.
 • Policy:
  ○ If is_org_email = false, you can:
   § Either block access entirely, or
   § Throttle (e.g., fewer questions, or only job-fit but not interview).
 • Session issuance:
  ○ If allowed, create a session row and return a session_token (JWT or random opaque token).
  ○ Frontend stores token in memory or secure cookie and attaches it to subsequent API calls.
This satisfies your “somewhat important” requirement to gather contact info and prefer organizational accounts.

##### Choosing a low/no-cost LLM API

Concrete options
 • Self-hosted Ollama (no API cost, infra only):
  ○ Run on a small VPS or home server.
  ○ Use a model like Llama 3 8B Instruct or Gemma 3 12B Instruct for good quality at modest hardware.
  ○ Expose a simple HTTP endpoint your backend calls.
  ○ Maximum control, no data leaves your environment.
 • Groq (managed, fast, free tier):
  ○ Use Llama 3.x or Mixtral models.
  ○ Very fast inference, good for chat-like UX.
  ○ Free tier with rate limits; good for low-traffic personal site.
 • OpenRouter (managed, many models):
  ○ Choose a small instruct model (Llama, Gemma, Mistral).
  ○ Single API, multiple providers behind the scenes.
  ○ Free/cheap tiers; good for experimentation.
 • Google Gemini / Gemma via Google AI Studio:
  ○ Generous free tier, strong models.
  ○ Be mindful of data usage policies and where data may be used for training.

##### Ranking Options

Given your emphasis on control and misuse prevention, I’d rank:

 1. Ollama (self-hosted)
 2. Groq with a small Llama model
 3. OpenRouter with a constrained model

##### Guardrails beyond prompts

Even with a constrained prompt and profile-only context, I’d add:
 • Input filters:
  ○ Reject or log clearly malicious or irrelevant prompts (e.g., “write malware”, “help me hack X”).
  ○ If the question is clearly not about your profile, respond with a fixed message:
“This tool only answers questions about Salman’s professional profile.”
 • Rate limiting:
  ○ Per-session and per-IP limits to reduce abuse.
 • Logging and review:
  ○ Store anonymized logs of prompts and responses for your own review to refine prompts and filters.
