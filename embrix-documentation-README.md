# Embrix for Python and Node.js

Embrix is a local-first AI application toolkit for Python and Node.js.

It brings together the pieces you usually need for practical AI workflows across both packages:

- vector storage and semantic search
- stateful workflow orchestration
- tool-using agents
- retrieval-augmented generation (RAG)
- database vectorization and natural-language querying
- an optional REST API for local services

By default, Embrix works locally with SQLite and a bring-your-own-model approach.

## Packages

### Python package

Package name:

`embrix`

Install from PyPI:

```bash
pip install embrix
```

Useful extras:

```bash
pip install "embrix[openai]"
pip install "embrix[sentence]"
pip install "embrix[all]"
```

### Node.js package

Package name:

`embrix-node`

Install from npm:

```bash
npm install embrix-node
```

Install the drivers or provider SDKs you actually use:

```bash
npm install better-sqlite3
npm install openai
npm install pg
npm install mysql2
npm install mongodb
```

## What the packages include

### Python package

| Component                        | Purpose                                                                     |
| -------------------------------- | --------------------------------------------------------------------------- |
| `VectorStore`                    | Store vectors and run similarity search                                     |
| `StateGraph`                     | Build multi-step workflows with state transitions                           |
| `Agent`                          | Run tool-using agents with your chosen model                                |
| `RAGPipeline`                    | Ingest documents, retrieve context, and answer questions                    |
| `DBVectorizer` + `DBQueryEngine` | Turn database rows into searchable context and query them in plain language |

### Node.js package

| Component                        | Purpose                                                      |
| -------------------------------- | ------------------------------------------------------------ |
| `VectorStore`                    | Local vector persistence and similarity search               |
| `StateGraph`                     | State-based workflow execution                               |
| `Agent`                          | Tool-using agents with iterative reasoning                   |
| `RAGPipeline`                    | Retrieval plus grounded response generation                  |
| `DBVectorizer` + `DBQueryEngine` | Database row vectorization, semantic search, and text-to-SQL |

## Quick start

### Python

#### Vector search

```python
from embrix import VectorStore

store = VectorStore()
store.upsert(
    "docs",
    [
        {
            "_id": "intro",
            "values": [0.1, 0.2, 0.3],
            "text": "Embrix runs local AI workflows with simple building blocks.",
            "metadata": {"topic": "overview"},
        }
    ],
)

results = store.query("docs", [0.1, 0.2, 0.3], top_k=1)
print(results)
store.close()
```

#### Stateful workflow

```python
from embrix import END, START, StateGraph

def retrieve(state):
    return {"context": f"facts for: {state['question']}"}

def respond(state):
    return {"answer": f"Answer from {state['context']}"}

graph = StateGraph()
graph.add_node("retrieve", retrieve)
graph.add_node("respond", respond)
graph.add_edge(START, "retrieve")
graph.add_edge("retrieve", "respond")
graph.add_edge("respond", END)

result = graph.compile().invoke({"question": "What is retrieval?"})
print(result["answer"])
```

#### Agent with your own provider

```python
from embrix import Agent, OpenAICompatibleChatModel
from embrix.agents import Tool

def calculator(expression: str) -> str:
    return str(eval(expression, {"__builtins__": {}}, {}))

agent = Agent(
    tools=[Tool("calculator", "Evaluate arithmetic expressions", calculator)],
    llm=OpenAICompatibleChatModel(
        model="gpt-4o-mini",
        api_key="your-api-key",
    ),
)

print(agent.run("Use the calculator tool to compute 12 * 9."))
```

#### RAG pipeline

```python
from embrix import RAGPipeline
from embrix.embeddings import SentenceEmbedder
from embrix.llms import OllamaChatModel

pipeline = RAGPipeline(
    embedder=SentenceEmbedder(),
    llm=OllamaChatModel(model="llama3.1"),
)

pipeline.ingest([
    "Embrix supports retrieval, workflows, agents, and database tools."
], namespace="docs")

print(pipeline.query("What does Embrix support?", namespace="docs"))
```

#### Database search and question answering

```python
from embrix import DBQueryEngine, DBVectorizer, VectorStore
from embrix.db import connect as db_connect

connector = db_connect("sqlite", db_path="products.db")
vectorizer = DBVectorizer(connector, embedder, store=VectorStore(backend="memory", dimension=embedder.dimension))
vectorizer.vectorize_table("products", namespace="products", text_columns=["name", "description"])

engine = DBQueryEngine(vectorizer=vectorizer, llm=chat_model, connector=connector)
hits = engine.search("budget running shoes", namespace="products")
rows = engine.sql_query("Show products under 100 dollars", table="products")
```

### Node.js

#### Vector storage

```ts
import { VectorStore } from "embrix-node";

const store = new VectorStore({ dbPath: "embrix.db", dimension: 3 });

store.upsert("docs", [
  {
    id: "intro",
    values: [0.1, 0.2, 0.3],
    text: "Embrix helps you build local-first AI features.",
    metadata: { topic: "overview" },
  },
]);

const results = store.query("docs", [0.1, 0.2, 0.3], 1);
console.log(results);
```

#### Stateful workflow

```ts
import { END, START, StateGraph } from "embrix-node";

type WorkflowState = { question: string; context?: string; answer?: string };

const graph = new StateGraph<WorkflowState>();
graph.addNode("retrieve", (state) => ({
  ...state,
  context: `facts for ${state.question}`,
}));
graph.addNode("respond", (state) => ({
  ...state,
  answer: `Answer from ${state.context}`,
}));
graph.addEdge(START, "retrieve");
graph.addEdge("retrieve", "respond");
graph.addEdge("respond", END);

const result = await graph.compile().invoke({ question: "What is retrieval?" });
console.log(result.answer);
```

#### Agent with your own provider

```ts
import { Agent, OpenAICompatibleChatModel, Tool } from "embrix-node";

const calculator = new Tool(
  "calculator",
  "Evaluate arithmetic expressions",
  async (input) => {
    return String(Function(`return (${input})`)());
  },
);

const agent = new Agent({
  tools: [calculator],
  llm: new OpenAICompatibleChatModel({
    model: "gpt-4o-mini",
    apiKey: process.env.OPENAI_API_KEY,
  }),
});

console.log(
  await agent.run("Use the calculator tool to compute (12 * 9) + 4."),
);
```

#### RAG pipeline

```ts
import {
  OpenAIEmbedder,
  RAGPipeline,
  VectorStore,
  createChatModel,
} from "embrix-node";

const store = new VectorStore({ dbPath: "rag.db", dimension: 1536 });
const embedder = new OpenAIEmbedder({ apiKey: process.env.OPENAI_API_KEY! });
const llm = createChatModel("ollama", { model: "llama3.1" });

const rag = new RAGPipeline(embedder, llm, store);
await rag.ingest(
  ["Embrix supports search, workflows, agents, and database tooling."],
  "docs",
);

console.log(await rag.query("What does Embrix support?", "docs"));
```

#### Database search and querying

```ts
import {
  DBQueryEngine,
  DBVectorizer,
  OpenAIEmbedder,
  VectorStore,
  dbConnect,
  createChatModel,
} from "embrix-node";

const connector = dbConnect("sqlite", { dbPath: "products.db" });
const embedder = new OpenAIEmbedder({ apiKey: process.env.OPENAI_API_KEY! });
const store = new VectorStore({
  dbPath: "products-vectors.db",
  dimension: 1536,
});

const vectorizer = new DBVectorizer(connector, embedder, { store });
await vectorizer.vectorizeTable("products", "products", {
  textColumns: ["name", "description"],
});

const engine = new DBQueryEngine(vectorizer, undefined, {
  connector,
  llm: createChatModel("openai-compatible", {
    model: "gpt-4o-mini",
    apiKey: process.env.OPENAI_API_KEY,
  }),
});

console.log(await engine.search("budget running shoes", "products"));
console.log(
  await engine.sqlQuery("Show products under 100 dollars", "products"),
);
```

## Model provider approach

Embrix does not bundle a hosted model service.

You plug in your own provider, key, gateway, or local runtime.

Built-in chat adapters include:

- `OpenAICompatibleChatModel`
- `AnthropicChatModel`
- `GeminiChatModel`
- `OllamaChatModel`
- `CustomChatModel`

That means the same app structure can work with hosted APIs, internal gateways, or local model runtimes.

## Azure database support

Embrix can work with Azure-hosted databases as source systems for vectorization and querying.

Current connector patterns fit well with:

- Azure Database for PostgreSQL
- Azure Database for MySQL
- Azure Cosmos DB for MongoDB
- Azure SQL Database through SQLAlchemy

Today, `VectorStore` is local-first and uses SQLite or memory by default.

That means a practical deployment pattern is:

- keep operational data in an Azure database
- use `DBVectorizer` to read and embed rows
- store vectors in Embrix locally or through a custom adapter

If you want Azure to act as the vector database itself, the strongest fit for the current architecture is Azure Database for PostgreSQL with `pgvector`.

Because Embrix already separates database connectors from vector storage and supports custom adapters, adding a PostgreSQL vector backend is a natural extension.

## REST API

Start the API server locally:

```bash
uvicorn embrix.api.server:app --reload --port 8080
```

Main endpoints:

- `POST /vectors/upsert`
- `POST /vectors/query`
- `POST /vectors/fetch`
- `POST /vectors/delete`
- `GET /vectors/list`
- `GET /describe_index_stats`
- `GET /namespaces`
- `GET /health`

## Package layout

### Python

```text
embrix/
├── agents/
├── api/
├── db/
├── embeddings/
├── graph/
├── llms/
├── rag/
└── vector/
```

### Node.js

```text
embrix-node/
├── src/agents/
├── src/db/
├── src/embeddings/
├── src/graph/
├── src/llms/
├── src/rag/
└── src/vector/
```

## Common use cases

- local AI prototypes
- notebook experiments
- backend services that need retrieval or agent steps
- document Q&A
- database exploration with semantic search
- Azure-hosted application data with Embrix retrieval workflows
- lightweight internal AI tools

## Development

Build the Node package locally:

```bash
cd embrix-node
npm install
npm run build
```

## License

MIT
