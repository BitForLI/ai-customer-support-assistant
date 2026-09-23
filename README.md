# AI Customer Support Assistant

I built this project around one question: if an assistant can read products, orders, and company policies, how do I stop it from pretending it has authority it does not have?

The result is a local customer-support demo. It can answer routine questions, remember useful details across messages, and show where a policy answer came from. When a customer asks for a sensitive action such as cancelling an order, it asks for confirmation and creates a ticket for a person instead of changing the order itself.

## Three paths to try

| Question | Result |
| --- | --- |
| `七天内可以退货吗？` | Retrieves the return policy and includes the document title and ID. |
| `查询订单 9003` | Uses the sample order tool and reports the order status. |
| `取消订单 9001` | Requests confirmation and then creates a human-review ticket. The order is not cancelled. |

The included catalogue, orders, and policies are fictional. The intent rules and evaluation examples are mainly Chinese, so translated English prompts do not exercise exactly the same path.

## Where I drew the boundary

The assistant may search products, check stock, look up sample orders, retrieve policies, and open an escalation ticket. It may not directly cancel an order, issue a refund, or modify customer data.

That boundary is enforced in the workflow rather than left to the wording of a prompt. Sensitive requests pause for explicit confirmation before ticket creation. Tool failures are retried once, and optional OpenAI wording can fall back to a local draft. Selected prompt-injection phrases are rejected, but this is a narrow rule-based check—not a claim that the project is protected from every possible attack.

## How a message moves through the project

```text
customer message
  -> intent and entity extraction
  -> product, order, policy, or support tool
  -> confirmation and failure handling
  -> answer with tool history and any sources
```

[`src/agent_graph.py`](src/agent_graph.py) defines that LangGraph workflow. [`src/business_tools.py`](src/business_tools.py) reads the sample business data, [`src/rag_engine.py`](src/rag_engine.py) combines BM25 with character-bigram similarity for policy retrieval, and [`src/tickets.py`](src/tickets.py) stores escalation tickets in SQLite. The MCP server reuses the same underlying operations rather than implementing a second set of business rules.

## Run locally

Python 3.11 or newer is required.

```bash
python -m venv .venv
# Windows
.venv\Scripts\activate
pip install -r requirements.txt

python main.py --web
# Open http://127.0.0.1:8000
```

The default `LLM_PROVIDER=local` mode needs no API key. To use an OpenAI model for response wording, copy `.env.example` to `.env`, set `LLM_PROVIDER=openai` and provide `OPENAI_API_KEY`.

Command-line and MCP modes are also available:

```bash
python main.py
python -m src.mcp_server
```

## Tests and evaluation

```bash
python -m pytest -q
python -m evals.run_eval
```

Tests cover product and order routing, policy references, multi-turn customer details, confirmation, ticket creation, tool failure, and the HTTP API. The six-case offline evaluation checks expected intent, required answer text, tool use, and whether a source-required answer contains a citation.

The evaluation checks citation presence, not whether every generated sentence is fully supported by the cited document. It is a small regression suite for this demo, not a general measure of production assistant quality.
