# RAG Neo4j Students

A natural-language sales analytics API for an online academy. It uses an LLM through OpenRouter to translate questions into Neo4j Cypher, runs the query against a graph of courses, students, purchases, and course progress, and returns an analytical answer with suggested follow-up questions.

Despite the repository name, the current retrieval path is graph querying with generated Cypher; it does not configure a vector store or embeddings.

## Capabilities

- Ask sales and student-engagement questions in natural language.
- Plan complex questions as up to three sub-questions and synthesize their results.
- Generate Cypher using the live Neo4j schema and academy-specific business rules.
- Validate generated queries with `EXPLAIN` and attempt one automatic correction.
- Return a natural-language answer and follow-up questions.
- Seed Neo4j with a course catalog and randomized student, purchase, and progress data.

The included dataset is synthetic except for the course names and URLs. Since Faker generates new records at each seed, analytics can differ between runs.

## Requirements

- Node.js `>=24.10.0` (as declared in `package.json`). Node's built-in TypeScript support is used; no build step is configured.
- npm
- Docker Compose, or a separately managed Neo4j 5-compatible database
- An OpenRouter API key for API requests and end-to-end tests

The optional LangGraph CLI configuration in `langgraph.json` declares Node 20, which conflicts with the package engine requirement. The npm scripts run the app directly with Node, not through that CLI. Align `langgraph.json` with the installed runtime before using `npm run langgraph:serve`.

## Setup

1. Install dependencies:

   ```sh
   npm install
   ```

2. Create `.env` from the example and add an OpenRouter key:

   ```sh
   cp .env.example .env
   ```

   ```dotenv
   OPENROUTER_API_KEY=your_openrouter_api_key
   ```

   `LANGSMITH_API_KEY`, `LANGCHAIN_TRACING_V2`, and `LANGCHAIN_PROJECT` are included for optional LangSmith tracing. The app's Neo4j connection is currently configured in `src/config.ts`, not through environment variables.

3. Start Neo4j:

   ```sh
   npm run docker:infra:up
   ```

   The Compose service exposes Neo4j Browser at [http://localhost:7474](http://localhost:7474) and Bolt at `localhost:7687`. Its default credentials are `neo4j` / `password`, matching the application configuration. These defaults are for local development only; change them before exposing the database or using it outside a disposable environment.

4. Seed the database once:

   ```sh
   node --env-file .env data/seed.ts
   ```

   This creates course, student, purchase, and progress data. **Seeding first deletes every node and relationship in the configured Neo4j database**, so use a dedicated development database. The `npm run seed` script adds Node watch mode and remains running to rerun the seed when its files change; use the direct command above for a one-time seed.

5. Start the API:

   ```sh
   npm start
   ```

   The service listens on `http://localhost:4000` (bound to `0.0.0.0` inside its runtime).

## API

### `POST /sales`

Request JSON:

```json
{
  "question": "Show me the revenue distribution across all courses"
}
```

`question` is required and must contain at least three characters. A successful response has this shape:

```json
{
  "answer": "The generated analytical response...",
  "followUpQuestions": [
    "Which course has the highest completion rate?",
    "Which payment method is most common?"
  ],
  "query": "MATCH ... RETURN ...",
  "error": null
}
```

`query` and `error` may be absent depending on the graph path. Invalid request bodies are rejected by Fastify validation; processing failures return HTTP 500 with an `error` message.

Try it with curl:

```sh
curl -X POST http://localhost:4000/sales \
  -H 'Content-Type: application/json' \
  -d '{"question":"Which students have 100% progress in their courses?"}'
```

Other example questions include:

- `List all available courses`
- `Who bought Formação JavaScript Expert?`
- `What is the total revenue from credit card payments?`
- `Which students bought a course but never started it?`
- `Which courses are commonly bought together?`

## Graph Data Model

The seed process creates this graph:

```text
(Student)-[:PURCHASED {status, paymentMethod, paymentDate, amount}]->(Course)
(Student)-[:PROGRESS {progress}]->(Course)
```

`Student` nodes have `id`, `name`, `email`, and `phone`. `Course` nodes have `name` and `url`. The seed generates 20 students; each receives between one and ten randomized purchase records. Purchase status is `paid` or `refunded`, payment method is `pix` or `credit_card`, and amounts range from 0 to 2000. Progress from 0 to 100 is generated for paid purchases.

The query-generation context instructs the model to treat paid purchases as active revenue and exclude refunded purchases from revenue calculations. This is prompt guidance, not a database constraint.

## Project Layout

- `data/courses.json`: academy course catalog used for seeding.
- `data/seed.ts`, `data/seedHelpers.ts`: database initialization and randomized sample-data generation.
- `src/server.ts`: Fastify server and `POST /sales` route.
- `src/config.ts`: OpenRouter model options, Neo4j connection, and graph limits.
- `src/services/`: OpenRouter structured-output and Neo4j integrations.
- `src/graph/`: LangGraph state machine and its question-planning, Cypher, execution, correction, and response nodes.
- `src/prompts/v1/`: prompts and structured-output schemas for the graph workflow.
- `tests/sales.e2e.test.ts`: end-to-end API coverage for sales and engagement questions.
- `docker-compose.yaml`: local Neo4j Community service with APOC enabled and persistent files under `storage/`.

## Commands

| Command | Purpose |
| --- | --- |
| `npm start` | Start the API using `.env`. |
| `npm run dev` | Start the API in Node watch mode with the inspector enabled. |
| `node --env-file .env data/seed.ts` | Seed the configured Neo4j database once. |
| `npm run seed` | Run the seed entry point in watch mode. |
| `npm test` | Run Node's test runner over `tests/**/*.test.ts`. |
| `npm run test:e2e` | Run the E2E tests with the inspector enabled. |
| `npm run test:dev` | Run tests in watch mode with the inspector enabled. |
| `npm run test:e2e:dev` | Run E2E tests in watch mode with the inspector enabled. |
| `npm run langgraph:serve` | Start the LangGraph CLI dev server; see the Node version note above. |
| `npm run docker:infra:up` | Start the Compose infrastructure and wait for it to be ready. |
| `npm run docker:infra:down` | Stop Compose services, preserving persisted data. |
| `npm run docker:infra:logs` | Follow Compose service logs. |
| `npm run docker:infra:cleanup` | Stop services and delete database volumes and local `storage/` data. |

The E2E tests call OpenRouter and Neo4j and seed the database before running. They are integration tests, not isolated unit tests. Because seeding clears the database, do not point them at data you need to keep.

## Configuration and Operations

- The OpenRouter model is configured in `src/config.ts` as `arcee-ai/trinity-large-preview:free`; provider availability and model behavior depend on OpenRouter.
- Neo4j defaults are `neo4j://localhost:7687`, username `neo4j`, and password `password`, also in `src/config.ts` and `docker-compose.yaml`.
- Compose persists Neo4j data, logs, plugins, and import files beneath `storage/` (ignored by Git). The cleanup script removes this local state.
- The correction-attempt limit is one, and the planner is configured for at most three sub-questions.
- The server logs each question and processing summary to stdout. Fastify request logging is disabled.

## Troubleshooting

- **Neo4j connection refused:** start Compose with `npm run docker:infra:up` and confirm Bolt is available on port `7687`.
- **Authentication failure:** the application credentials must match `NEO4J_AUTH` in Compose. The current application credentials are fixed in `src/config.ts`.
- **LLM request failure:** check that `.env` exists, `OPENROUTER_API_KEY` is valid, and the configured model is available to the account.
- **Tests fail before making assertions:** confirm Neo4j is running and seeded, and that `.env` has a usable OpenRouter key. Tests need network access to OpenRouter.
- **No results:** run the seed command against the same Neo4j instance configured by the service.

## License

The package metadata declares the project license as MIT.