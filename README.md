# Building a Simple Research Agent with Auwgent, pip, and Python

![Cover: Auwgent research agent](https://blog-images-cdn.henryantwi.me/auwgent-blog-images/screenshots/auwgent-thumbnail1.webp)

[Auwgent](https://github.com/snrraptopack/a-simple-research-agent) compiles agent definitions into typed Python bindings. The **[official documentation](https://auwgent.snrraptopack.workers.dev)** covers the framework. This walkthrough uses a virtual environment and **pip**, a **`.env`** file for API keys, optional **`auwgent generate`** when you change agent sources, and **`python index.py`** to run the CLI.

## What this project is

Agents that do real work usually need **tools**, **streaming or structured events**, and a **repeatable compile step** when the definition changes. With Auwgent you **define** the agent under `auwgent_src/` and **generate** `generated/main_types.py` (and related files) so you get a typed `auwgent(...)` factory in Python.

The example repo is a **CLI research agent** that can:

- Search the web with [Tavily](https://app.tavily.com/home) (`search_web`).
- Manage in-memory todos (`create_todo`, `read_todo`, `delete_todo`).
- Use **planning** and **dialogue** custom intents defined in `auwgent_src/`.

The chat model is served through **Groq**’s OpenAI-compatible API. Create a key in the [Groq console](https://console.groq.com/keys) and set `GROQ_API_KEY` in `.env`.

## Prerequisites

- Python **3.11+**
- **`venv`** and **`pip`**
- **`GROQ_API_KEY`** and **`TAVILY_API_KEY`**
- If you edit `.agent` files: **Node 18+** and the Auwgent CLI:

```bash
npm install -g @snrraptopack/auwgent-cli
```

## Project layout

- **`auwgent.yaml`** — compiler paths and Python target.
- **`auwgent_src/`** — `.agent` sources (main agent, prompt, custom intents).
- **`generated/`** — output from the compiler (`main_types.py`, `main.agent.json`, …).
- **`index.py`** — CLI loop and intent handlers.
- **`tools.py`** — Tavily search and todo tools, wired to the generated types.

Root **`requirements.txt`** lists Python dependencies.

```
a-simple-research-agent/
├── .gitignore
├── README.md
├── auwgent.yaml
├── auwgent_src/
│   ├── custom_intents.agent
│   ├── main.agent
│   └── prompt.agent
├── generated/
│   ├── main.agent.json
│   └── main_types.py
├── index.py
└── tools.py
```

### `auwgent.yaml`

```yaml
source: "auwgent_src"
output: "generated"
targets:
  - py
```

![Project layout in the editor](https://blog-images-cdn.henryantwi.me/auwgent-blog-images/screenshots/01-project-layout.png)

## Step 1 — Clone the repository

```bash
git clone https://github.com/snrraptopack/a-simple-research-agent.git
cd a-simple-research-agent
```

## Step 2 — Virtual environment and install

Dependencies: **`auwgent-sdk`**, **`python-dotenv`**, **`tavily-python`**.

```bash
python -m venv .venv
```

Activate the environment:

- **Windows (PowerShell):** `.\.venv\Scripts\Activate.ps1`
- **Windows (cmd):** `.\.venv\Scripts\activate.bat`
- **macOS / Linux:** `source .venv/bin/activate`

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

### `requirements.txt`

```text
auwgent-sdk
python-dotenv
tavily-python
```

## Step 3 — Environment variables

Create **`.env`** in the project root. Do not commit it. Keys: [Groq](https://console.groq.com/keys), [Tavily](https://app.tavily.com/home).

```env
GROQ_API_KEY=your_groq_key_here
TAVILY_API_KEY=your_tavily_key_here
```

## Entrypoint code

### `index.py`

```python
from generated.main_types import MainAgentContext, auwgent
import asyncio
from typing import Any

from dotenv import load_dotenv
import os

from tools import has_todos, todo_ids,todo_left,tools

load_dotenv()


def _response_chunk(value: Any) -> str:
    """Partial and full intents may use delta, text, or snapshot (see auwgent_sdk)."""
    if not isinstance(value, dict):
        return ""
    if value.get("delta"):
        return str(value["delta"])
    if value.get("text"):
        return str(value["text"])
    snap = value.get("snapshot")
    if isinstance(snap, str) and snap:
        return snap
    return ""


_streamed_this_turn = False


context:MainAgentContext = {
    'user_name': "Your name",
}

agent = auwgent({
    "apiKeys": {
        "my_groq_apiApiKey": os.getenv("GROQ_API_KEY","")
    },
    "context": context,
    "tools":tools
})

def handle_intent(name: str, value: Any, agent_name: str):
    global _streamed_this_turn
    if name == "response_text":
        chunk = _response_chunk(value)
        if chunk:
            print(chunk, end="", flush=True)
            _streamed_this_turn = True

async def handle_intent_full(name: str, value: Any, agent_name: str):
    global _streamed_this_turn
    if name == "response_text":
        if not _streamed_this_turn:
            chunk = _response_chunk(value)
            if chunk:
                print(chunk, end="", flush=True)
        _streamed_this_turn = False
        return
    print(f"{name}: {value}")

agent.on_intent_partial(handle_intent)
agent.on_intent(handle_intent_full)

async def main():
    global _streamed_this_turn

    print("Chat with the agent (type 'exit' or 'quit' to stop)\n")

    while True:
        try:
            user_input = input("\nYou:").strip()

            if user_input.lower() in ['exit', 'quit', 'q']:
                print("Goodbye!")
                break

            if not user_input:
                continue

            _streamed_this_turn = False
            print("\nAgent: ", end="", flush=True)
            result = await agent.run(user_input)
            print()

        except KeyboardInterrupt:
            print("\n\nGoodbye!")
            break
        except Exception as e:
            print(f"\nError: {e}")

if __name__ == "__main__":
    asyncio.run(main())
```

### `tools.py`

```python
from tavily import TavilyClient

from dotenv import load_dotenv
import os

from generated.main_types import MainAgentTools
from typing import Optional

load_dotenv()
tavily_client = TavilyClient(api_key=os.getenv("TAVILY_API_KEY"))

todo_state:dict[str, list[str]] = {}
has_todos = len(todo_state.keys()) > 0
todo_ids = ",".join(todo_state.keys())
todo_left = len(todo_state.keys())


async def search_web(query: str) -> str:
    response = tavily_client.search(query)
    return response["results"][0]["content"]

async def read_todo(id:str) -> list[str]:
    clean_id = id.strip()
    if clean_id in todo_state:
        return todo_state[clean_id]
    return ["No todos found for this ID"]

async def create_todo(id: str, tasks: list[str]) -> list[str]:
    todo_state[id.strip()] = tasks
    return tasks

async def delete_todo(id: str, target_task: Optional[str] = None, main: Optional[bool] = None) -> bool:
    clean_id = id.strip()
    if clean_id in todo_state and main is not None:
        todo_state.pop(clean_id)
        return True
    if clean_id in todo_state and target_task is not None:
        todo_state[clean_id] = [t for t in todo_state[clean_id] if t != target_task]
        return True
    return False


tools:MainAgentTools = {
    'create_todo': create_todo,
    'read_todo': read_todo,
    'delete_todo': delete_todo,
    'search_web': search_web,
}
```

## Step 4 — Regenerate after editing `.agent` files

```bash
auwgent generate
```

This updates `generated/main_types.py` and `main.agent.json`. You can skip this on a clean clone if you are not changing the agent definition.

![auwgent generate in the terminal](https://blog-images-cdn.henryantwi.me/auwgent-blog-images/screenshots/04-auwgent-generate.png)

## Step 5 — Run the agent

With the virtual environment active:

```bash
python index.py
```

Stop with `exit`, `quit`, `q`, or Ctrl+C.

![Starting the CLI](https://blog-images-cdn.henryantwi.me/auwgent-blog-images/screenshots/05-run-cli.png)

## How it fits together

![Data flow diagram](https://blog-images-cdn.henryantwi.me/auwgent-blog-images/screenshots/flow.png)

- **`index.py`** — input loop and printing agent output.
- **`tools.py`** — Tavily and todo implementations.
- **`generated/main_types.py`** — generated types and factory; regenerate after compiler changes instead of editing by hand.

## Agent definitions (`auwgent_src/`)

### `main.agent`

```text
import {Planing, Dialogue} from "./custom_intents"
import {MainPrompt} from "./prompt"

agent MainAgent{
    default config {
        model:custom("my-groq-api", "https://api.groq.com/openai/v1/chat/completions", "openai/gpt-oss-120b")
        prompt: MainPrompt(ctx.user_name)
    }

    context {
        user_name:string,
    }

    intent:Planing + Dialogue

    tool search_web(query:string):string
        @desc "use this tool to search the web for information"

    tool create_todo(id:string @desc "should be descriptive", tasks:string[]):string[]
        @desc "use this tool to create todos you will use to solve the problem"
        @example(id="understanding the reasoning of humans", tasks=["list of the steps you will take"])

    tool read_todo(id:string @desc "a valid todo id"):string[]
        @desc "use this tool to read a todo"
        @example(id="understanding the reasoning of humans")

    tool delete_todo(id:string @desc "a valid todo id", target_task?:string @desc "specific todo in the list", main?:boolean @desc "to delete the main todo"):boolean
        @desc "use this tool to delete a todo"
        @example(id="understanding the reasoning of humans", target="index:1")
}
```

### `prompt.agent`

```text
export prompt MainPrompt(user_name:string){
    """
    You are a research agent. Your task is to understand the user's queries
    Ask questions using the Dialogue intents if unclear or needs more context
    Use Planning intents to layout your plans on how you would solve the problem
    Use the todo tools to create, delete or view todos
    Use the search tool to carry out web search of different queries

    Always use the appropriate tools
    Remember to update the todo when a task or a todo is done
    Be smart and also precise when the question is simple

    user's name : {{user_name}}

    """
}
```

### `custom_intents.agent`

```text
export intent Planing{
    description: "when asked a complex/research question use this to lay out your approach in solving it"
    fields {
        planing:string
    }
}

export intent Dialogue{
    description: "when asked complex question it good to have conversation with the user to understand the context and requirements"
    fields {
        question:string
    }
}@example(question = "please before i proceed i will like to ask few questions to understand the context...")
```

## Example session

With valid keys in `.env`, you can chat with the agent and use questions that trigger search or todos.

![Sample chat session](https://blog-images-cdn.henryantwi.me/auwgent-blog-images/screenshots/06-chat-transcript.png)

## If something goes wrong

- **API errors** — Check `.env` in the project root and variable names. Confirm keys at [Groq](https://console.groq.com/keys) and [Tavily](https://app.tavily.com/home).
- **Import errors for `generated`** — Run commands from the project root, or run `auwgent generate` after changing `.agent` sources.
- **Python** — Use 3.11 or newer.

---

**Links:** [Auwgent docs](https://auwgent.snrraptopack.workers.dev) · [Repository](https://github.com/snrraptopack/a-simple-research-agent) · [Auwgent CLI (npm)](https://www.npmjs.com/package/@snrraptopack/auwgent-cli) · [Tavily](https://app.tavily.com/home) · [Groq API keys](https://console.groq.com/keys)
