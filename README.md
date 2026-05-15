# Embrix for Python

Embrix is a local-first AI application toolkit for Python.

It brings together the pieces you usually need for practical AI workflows in one package:

- vector storage and semantic search
- stateful workflow orchestration
- tool-using agents
- retrieval-augmented generation (RAG)
- database vectorization and natural-language querying
- an optional REST API for local services

By default, Embrix works locally with SQLite and a bring-your-own-model approach.

## Install from PyPI

Core package:

```bash
pip install embrix
```

Useful extras:

```bash
pip install "embrix[openai]"
pip install "embrix[sentence]"
pip install "embrix[all]"
```

## What the Python package includes

| Component                        | Purpose                                                                     |
| -------------------------------- | --------------------------------------------------------------------------- |
| `VectorStore`                    | Store vectors and run similarity search                                     |
| `StateGraph`                     | Build multi-step workflows with state transitions                           |
| `Agent`                          | Run tool-using agents with your chosen model                                |
| `RAGPipeline`                    | Ingest documents, retrieve context, and answer questions                    |
| `DBVectorizer` + `DBQueryEngine` | Turn database rows into searchable context and query them in plain language |

## Quick start

### Vector search

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

### Stateful workflow

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

### Agent with your own provider

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

### RAG pipeline

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

### Database search and question answering

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

## Common use cases

- local AI prototypes
- notebook experiments
- backend services that need retrieval or agent steps
- document Q&A
- database exploration with semantic search
- lightweight internal AI tools

## License

MIT
