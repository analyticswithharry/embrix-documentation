# Embrix Documentation

Embrix is a local-first AI toolkit for Python and Node.js applications.

It provides a practical set of building blocks for modern AI workflows:

- vector storage and semantic search
- retrieval-augmented generation (RAG)
- tool-using agents
- stateful workflow orchestration
- database vectorization and querying
- bring-your-own model provider support

This repository is intended as the documentation and demo companion for the published Embrix packages.

## Packages

### Python package

Package name:

`embrix`

Install from PyPI:

```bash
pip install embrix
```

Optional extras:

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

Install only the drivers or SDKs you need:

```bash
npm install better-sqlite3
npm install openai
npm install pg
npm install mysql2
npm install mongodb
```

## Repository contents

This documentation repository includes:

- `embrix_python.ipynb` — Python SDK demo notebook
- `embrix_node.ipynb` — Node.js SDK demo notebook
- SQLite demo database files used by the notebooks

The notebooks demonstrate:

- environment setup
- vector search
- workflow execution
- agent usage
- RAG pipelines
- database vectorization and querying

## Python package overview

The Python package includes:

- `VectorStore` for vector persistence and similarity search
- `StateGraph` for multi-step workflows
- `Agent` for tool-using agent behavior
- `RAGPipeline` for document ingestion and grounded answers
- `DBVectorizer` and `DBQueryEngine` for database-aware retrieval and querying

### Python quick start

#### Vector search

```python
from embrix import VectorStore

store = VectorStore()

store.upsert("docs", [
    {
        "_id": "1",
        "values": [0.1, 0.2, 0.3],
        "text": "Embrix helps build local-first AI apps.",
        "metadata": {"topic": "overview"},
    }
])

results = store.query("docs", [0.1, 0.2, 0.3], top_k=1)
print(results)
```

#### Stateful workflow

```python
from embrix import START, END, StateGraph

def retrieve(state):
    return {"context": f"facts for: {state['question']}"}

def answer(state):
    return {"answer": f"Response using {state['context']}"}

graph = StateGraph()
graph.add_node("retrieve", retrieve)
graph.add_node("answer", answer)
graph.add_edge(START, "retrieve")
graph.add_edge("retrieve", "answer")
graph.add_edge("answer", END)

result = graph.compile().invoke({"question": "What is Embrix?"})
print(result["answer"])
```

#### Agent

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

pipeline.ingest(
    ["Embrix supports vector search, workflows, agents, and database querying."],
    namespace="docs",
)

print(pipeline.query("What does Embrix support?", namespace="docs"))
```

#### Database querying

```python
from embrix import DBVectorizer, DBQueryEngine, VectorStore
from embrix.db import connect as db_connect

connector = db_connect("sqlite", db_path="products.db")
vectorizer = DBVectorizer(
    connector,
    embedder,
    store=VectorStore(backend="memory", dimension=embedder.dimension),
)

vectorizer.vectorize_table(
    "products",
    namespace="products",
    text_columns=["name", "description"],
)

engine = DBQueryEngine(vectorizer=vectorizer, llm=chat_model, connector=connector)
print(engine.search("budget running shoes", namespace="products"))
print(engine.sql_query("Show products under 100 dollars", table="products"))
```

## Node.js package overview

The Node.js package includes:

- `VectorStore` for vector persistence and search
- `StateGraph` for state-based workflows
- `Agent` for tool-driven agent execution
- `RAGPipeline` for retrieval and grounded generation
- `DBVectorizer` and `DBQueryEngine` for database search and text-to-SQL

### Node.js quick start

#### Vector search

```ts
import { VectorStore } from "embrix-node";

const store = new VectorStore({ dbPath: "embrix.db", dimension: 3 });

store.upsert("docs", [
  {
    id: "intro",
    values: [0.1, 0.2, 0.3],
    text: "Embrix helps build local-first AI apps.",
    metadata: { topic: "overview" },
  },
]);

const results = store.query("docs", [0.1, 0.2, 0.3], 1);
console.log(results);
```

#### Stateful workflow

```ts
import { START, END, StateGraph } from "embrix-node";

type WorkflowState = {
  question: string;
  context?: string;
  answer?: string;
};

const graph = new StateGraph<WorkflowState>();

graph.addNode("retrieve", (state) => ({
  ...state,
  context: `facts for ${state.question}`,
}));

graph.addNode("answer", (state) => ({
  ...state,
  answer: `Response using ${state.context}`,
}));

graph.addEdge(START, "retrieve");
graph.addEdge("retrieve", "answer");
graph.addEdge("answer", END);

const result = await graph.compile().invoke({ question: "What is Embrix?" });
console.log(result.answer);
```

#### Agent

```ts
import { Agent, OpenAICompatibleChatModel, Tool } from "embrix-node";

const calculator = new Tool(
  "calculator",
  "Evaluate arithmetic expressions",
  async (input) => String(Function(`return (${input})`)()),
);

const agent = new Agent({
  tools: [calculator],
  llm: new OpenAICompatibleChatModel({
    model: "gpt-4o-mini",
    apiKey: process.env.OPENAI_API_KEY,
  }),
});

console.log(await agent.run("Use the calculator tool to compute 12 * 9."));
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
  ["Embrix supports vector search, workflows, agents, and database querying."],
  "docs",
);

console.log(await rag.query("What does Embrix support?", "docs"));
```

#### Database querying

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
console.log(await engine.sqlQuery("Show products under 100 dollars", "products"));
```

## Provider model

Embrix does not bundle its own hosted model service.

Instead, you connect it to your own provider, gateway, API key, or local runtime.

Built-in provider adapters include:

- `OpenAICompatibleChatModel`
- `AnthropicChatModel`
- `GeminiChatModel`
- `OllamaChatModel`
- `CustomChatModel`

This makes Embrix suitable for local development, private infrastructure, hosted APIs, and offline-friendly workflows.

## Architecture

```text
User Question / Application Input
                |
                v
        Embrix Building Blocks

   +-----------------------------+
   | VectorStore                 |
   | Store vectors and search    |
   +-----------------------------+
                |
                v
   +-----------------------------+
   | RAGPipeline                 |
   | Retrieve context and answer |
   +-----------------------------+
                |
                v
   +-----------------------------+
   | Agent                       |
   | Use tools and reason        |
   +-----------------------------+
                |
                v
   +-----------------------------+
   | StateGraph                  |
   | Run multi-step workflows    |
   +-----------------------------+
                |
                v
   +-----------------------------+
   | DBVectorizer / DBQueryEngine|
   | Query structured data       |
   +-----------------------------+
                |
                v
          Final Output
```

## Typical use cases

- internal knowledge tools
- semantic search applications
- RAG-backed assistants
- tool-using AI agents
- workflow-based automation
- database exploration in natural language
- local-first AI prototypes

## Repository structure

```text
embrix-documentation/
├── README.md
├── LICENSE
├── embrix_python.ipynb
├── embrix_node.ipynb
├── embrix_python_demo.sqlite
├── embrix_node_products.sqlite
├── embrix_node_db_vectors.sqlite
├── embrix_node_rag.db
└── embrix_node_vector_demo.sqlite
```

## License

MIT
