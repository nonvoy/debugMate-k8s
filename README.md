> [!WARNING]
> **Work in Progress**
>
> DebugMate is currently under active development. Features, APIs, data models, and architecture may change as the project evolves. The current implementation represents an early MVP focused on event ingestion, event processing, incident detection, and AI-assisted incident analysis.

## About DebugMate

DebugMate is an AI-powered observability assistant designed to help engineers understand production issues faster.

Modern distributed systems generate a large volume of logs, infrastructure events, and deployment notifications. DebugMate collects these events, normalizes and groups similar occurrences, detects potential incidents, and generates concise summaries to help engineers identify the most likely root causes.

The project is currently being developed as a collection of microservices:

- **debugMate-api** — Event ingestion API built with FastAPI.
- **debugMate-worker** — Background processing service responsible for event normalization, fingerprint generation, grouping, incident detection, and AI-powered analysis.