# JobFit Architecture

A platform for job fit evaluation and virtual interviews, leveraging a closed, persistent LLM context based on a professional profile.

## Overview

This project defines the architecture for two primary tools:

1. **Job Fit Evaluator**: Analyze job descriptions against a professional profile to determine fit, identify gaps, and provide detailed explanations.
2. **Virtual Interview Tool**: Conduct virtual interviews where an AI agent answers questions based strictly on the provided professional context (resume, experience, education).

## Architecture

The system is designed with a focus on specific profile constraints and data privacy.

- **Frontend**: React/Next.js based web interface.
- **Backend**: Node.js or Python API handling auth and LLM interactions.
- **LLM Layer**: Constrained context using RAG (Retrieval-Augmented Generation) on a fixed profile dataset.
- **Data Layer**: Vector storage for profile embeddings and relational storage for session management.

See [Architecture Overview](architecture-overview.md) for detailed technical specifications.

### Diagrams

- [System Context](c4-sys-context.png)
- [Container Diagram](c4-container.png)
- [High Level Architecture](high-level-arch.png)

## License

Copyright 2026. All rights reserved.

Redistribution and use of these components are permitted ONLY if credit is given to the author.
Reuse without proper attribution is prohibited.
