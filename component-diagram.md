# Component diagram

Below is a clean, implementation‑ready PlantUML component diagram that captures the full architecture you described:

- Frontend (Job Fit + Interview Tools)
- Backend API Gateway
- Google Gemini 3 as the LLM
- Cloudflare D1 Database for profile context, embeddings, contacts, sessions, and conversation history
- RAG pipeline enforcing strict profile‑only context
- Contact gating + org‑email filtering
- No external search or tools allowed for the LLM

Everything is expressed as a component diagram, not a sequence diagram, so you can drop it directly into a .puml file.

What this diagram expresses

1. Strict profile‑only LLM behavior
The dashed red constraint arrow shows that Gemini 3 receives a system prompt enforcing:
 • No external search
 • No browsing
 • No tools
 • No hallucinated experience
 • Only use the profile snippets retrieved from D1
 • Decline unrelated questions
This is the core safety boundary.

2. Cloudflare D1 as the single source of truth
D1 stores:
Table | Purpose

---------------
ProfileDocuments | Your education, experience, projects
ProfileEmbeddings | Vector embeddings for RAG
Contacts | User identity + email domain classification
Sessions | Session tokens for access control
Conversations | Multi-turn interview sessions
Messages | Chat history

1. RAG Engine
The RAG engine retrieves only the relevant profile snippets using:
 • Embedding similarity search
 • Document metadata filters
 • Hard limits on number of retrieved chunks
This ensures the LLM sees only what you want it to see.

2. Contact gating + org email filtering
The Auth Service and Email Domain Filter enforce:
 • Collect name, email, org
 • Reject free email domains (gmail, yahoo, etc.)
 • Issue session tokens only for approved users

3. Two tools, one architecture
Both the Job Fit Tool and Interview Tool reuse:
 • Session validation
 • RAG retrieval
 • Prompt builder
 • Gemini 3 API
