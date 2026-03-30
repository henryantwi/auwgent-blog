# Building a Simple Research Agent with Auwgent and Python

![Cover: Auwgent research agent](https://blog-images-cdn.henryantwi.me/auwgent-blog-images/screenshots/auwgent-thumbnail1.webp)

## What is Auwgent?

Auwgent is a compiler-first framework for building AI agents. You define your agent using a DSL — a purpose-built language that describes what the agent is, what it can do, and how it should behave. The compiler takes that definition and generates typed bindings for your target language. Your application then imports those bindings and wires up the implementations.

The key idea behind Auwgent is a clean boundary between what the AI can do and how your application does it. You define the agent's capabilities in the DSL. You implement the actual logic in your language of choice. The two never interfere with each other.

The [official documentation](https://auwgent.snrraptopack.workers.dev) covers the full framework. This post walks through building a working research agent using Python.

---

## What we are building

A CLI research agent that can:

- Search the web using [Tavily](https://app.tavily.com/home)
- Manage an in-memory todo list across a conversation
- Use custom intents to plan its approach and ask clarifying questions

The model is served through [Groq's](https://console.groq.com/keys) OpenAI-compatible API.

---

## Prerequisites

- Python 3.11 or later
- Node.js 18 or later
- A `GROQ_API_KEY` from [Groq](https://console.groq.com/keys)
- A `TAVILY_API_KEY` from [Tavily](https://app.tavily.com/home)

Auwgent ships as two packages. Install the CLI globally — this is what compiles your `.agent` files:

```bash
npm install -g @snrraptopack/auwgent-cli
```

Install the Python SDK in your project — the generated bindings import from it at runtime:

```bash
pip install auwgent-sdk
```

You will not use the SDK directly in your application code. The CLI abstracts the configuration and generates everything you need. The SDK just needs to be present because the generated files depend on it.

---

## Project layout

```
a-simple-research-agent/
├── .env
├── auwgent.yaml
├── auwgent_src/
│   ├── custom_intents.agent
│   ├── main.agent
│   └── prompt.agent
├── generated/
│   ├── main.agent.json
│   └── main_types.py
├── index.py
├── requirements.txt
└── tools.py
```

![Project layout in the editor](https://blog-images-cdn.henryantwi.me/auwgent-blog-images/screenshots/01-project-layout.png)

---

## Step 1 — Clone the repository

```bash
git clone https://github.com/snrraptopack/a-simple-research-agent.git
cd a-simple-research-agent
```

---

## Step 2 — Set up the environment

```bash
python -m venv .venv
```

Activate it:

- **macOS / Linux:** `source .venv/bin/activate`
- **Windows (PowerShell):** `.\.venv\Scripts\Activate.ps1`
- **Windows (cmd):** `.\.venv\Scripts\activate.bat`

Then install dependencies:

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

**`requirements.txt`**
```text
auwgent-sdk
python-dotenv
tavily-python
```

---

## Step 3 — Add your API keys

Create a `.env` file in the project root. Do not commit it.

```env
GROQ_API_KEY=your_groq_key_here
TAVILY_API_KEY=your_tavily_key_here
```

---

## Understanding the project config — `auwgent.yaml`

Before looking at the agent definition, it is worth understanding what `auwgent.yaml` does. This file is the entry point for the Auwgent compiler. It tells the compiler three things:

- **`source`** — where your `.agent` files live. This can be a single file or a folder. If it is a folder, the compiler picks up every `.agent` file inside it.
- **`output`** — where the compiler should write the generated files.
- **`targets`** — which languages to compile to. Auwgent is not tied to any one language. This project targets Python, but the same DSL can compile to TypeScript or any other supported target.

```yaml
source: "auwgent_src"
output: "generated"
targets:
  - py
```

When you run `auwgent generate`, the compiler reads everything in `auwgent_src/`, compiles it, and writes the output into `generated/`. In this project that produces two files: `main.agent.json` (the compiled IR) and `main_types.py` (the typed Python bindings your application imports).

---

## Understanding intents

Before diving into the agent definition, it is important to understand what intents are — because they are central to how Auwgent works.

An intent is how the model communicates with your application. Every action the model takes is expressed as a named intent. Even a simple text reply is an intent — `response_text`. When the model calls a tool, that is a `tool_call` intent. When a tool returns a result, that is a `tool_result` intent.

Intents matter because they give your application actionable, named keys to react to. Instead of parsing raw model output, you register a handler and respond to specific intent names. The model does not need to be prompted to format its output in a particular way — the intent system handles that boundary.

You can also define **custom intents** specific to your agent. This is how you teach the model to express structured behaviour — like laying out a plan before acting, or asking the user a clarifying question — without writing complex prompt instructions. The intent declaration is enough.

---

## The agent definition

### `auwgent.yaml`

Already covered above — source, output, target.

### `auwgent_src/custom_intents.agent`

```ts
export intent Planing {
    description: "when asked a complex/research question use this to lay out your approach in solving it"
    fields {
        planing: string
    }
}

export intent Dialogue {
    description: "when asked a complex question it is good to have a conversation with the user to understand the context and requirements"
    fields {
        question: string
    }
} @example(question = "please before i proceed i will like to ask a few questions to understand the context...")
```

These are custom intents. `Planing` tells the model to express its reasoning before acting. `Dialogue` tells it to ask the user a question when context is unclear. Defining these as intents means the model uses them naturally based on the description — no elaborate prompt engineering required.

### `auwgent_src/prompt.agent`

```ts
export prompt MainPrompt(user_name: string) {
    """
    You are a research agent. Your task is to understand the user's queries.
    Ask questions using the Dialogue intent if unclear or if you need more context.
    Use the Planning intent to lay out your approach before solving complex problems.
    Use the todo tools to create, read, and delete todos as you work.
    Use the search tool to carry out web searches.

    Always use the appropriate tools.
    Remember to update todos when a task is done.
    Be precise when the question is simple.

    User's name: {{user_name}}
    """
}
```

### `auwgent_src/main.agent`

```ts
import { Planing, Dialogue } from "./custom_intents"
import { MainPrompt } from "./prompt"

agent MainAgent {
    default config {
        model: custom("my-groq-api", "https://api.groq.com/openai/v1/chat/completions", "openai/gpt-oss-120b")
        prompt: MainPrompt(ctx.user_name)
    }

    context {
        user_name: string
    }

    intent: Planing + Dialogue

    tool search_web(query: string): string
        @desc "use this tool to search the web for information"

    tool create_todo(id: string @desc "should be descriptive", tasks: string[]): string[]
        @desc "use this tool to create todos you will use to solve the problem"
        @example(id = "understanding the reasoning of humans", tasks = ["list of the steps you will take"])

    tool read_todo(id: string @desc "a valid todo id"): string[]
        @desc "use this tool to read a todo"
        @example(id = "understanding the reasoning of humans")

    tool delete_todo(id: string @desc "a valid todo id", target_task?: string @desc "specific todo in the list", main?: boolean @desc "to delete the main todo"): boolean
        @desc "use this tool to delete a todo"
        @example(id = "understanding the reasoning of humans", target = "index:1")
}
```

---

## Tools, @desc, and @example

Tools are capabilities you declare in the DSL that the model can call during a conversation. You declare the name, the arguments it takes, and what it returns. The actual implementation lives in your application — Auwgent just defines the contract.

Two optional annotations help the model understand and use tools correctly:

**`@desc`** — a plain language description of what the tool does or what an argument means. The model reads this when deciding whether and how to call the tool. Without it the model has to guess from the name alone. A good description significantly improves how reliably the model uses the tool.

`@desc` can appear at the tool level to describe the whole tool, or inline on a specific argument to clarify what it expects:

```ts
tool delete_todo(
    id: string @desc "a valid todo id",
    target_task?: string @desc "a specific task inside the todo to remove",
    main?: boolean @desc "set to true to delete the entire todo"
): boolean
    @desc "use this tool to delete a todo or a specific task inside one"
```

**`@example`** — a concrete example of how to call the tool, expressed as named argument values. The model uses this to learn the expected format, especially when the format is not obvious from the type alone:

```ts
tool create_todo(id: string, tasks: string[]): string[]
    @desc "use this tool to create a todo list"
    @example(id = "understanding the reasoning of humans", tasks = ["research the topic", "summarise findings"])
```

Both annotations are optional. But providing them — especially for tools with non-obvious arguments — reduces malformed tool calls and means less prompt engineering on your side.

Auwgent limits the total number of examples included in the compiled context to 5 across all tools. If you annotate more than 5 tools with `@example`, the compiler selects the most relevant ones. This keeps the model context lean regardless of how many tools your agent has.

---

## Step 4 — Generate the bindings

Run the compiler:

```bash
auwgent generate
```

This reads your `.agent` files and writes the generated output into `generated/`. You must run this every time you change your agent definition. It is not optional — the generated files are what your application imports, and they must stay in sync with your DSL.

![auwgent generate in the terminal](https://blog-images-cdn.henryantwi.me/auwgent-blog-images/screenshots/04-auwgent-generate.png)

---

## The application code

### `tools.py`

This is where the actual tool logic lives. Notice that these are plain async functions — no framework-specific wrappers, no special base classes. Auwgent does not touch this file. You implement the logic however you want and wire it up through the generated types.

```python
from tavily import TavilyClient
from dotenv import load_dotenv
from generated.main_types import MainAgentTools
from typing import Optional
import os

load_dotenv()

tavily_client = TavilyClient(api_key=os.getenv("TAVILY_API_KEY"))
todo_state: dict[str, list[str]] = {}


async def search_web(query: str) -> str:
    response = tavily_client.search(query)
    return response["results"][0]["content"]


async def read_todo(id: str) -> list[str]:
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


tools: MainAgentTools = {
    "create_todo": create_todo,
    "read_todo": read_todo,
    "delete_todo": delete_todo,
    "search_web": search_web,
}
```

### `index.py`

```python
import asyncio
import os
from typing import Any

from dotenv import load_dotenv
from generated.main_types import MainAgentContext, MainAgentIntentName, MainAgentIntentValue, auwgent
from tools import tools

load_dotenv()

context: MainAgentContext = {
    "user_name": "Your name",
}

agent = auwgent({
    "apiKeys": {
        "my_groq_apiApiKey": os.getenv("GROQ_API_KEY", "")
    },
    "context": context,
    "tools": tools
})


def handle_intent(name: MainAgentIntentName, value: MainAgentIntentValue, agent_name: str):
    if name == "response_text":
        print(value.get("delta", ""), end="", flush=True)


async def handle_intent_full(name: MainAgentIntentName, value: MainAgentIntentValue, agent_name: str):
    if name != "response_text":
        print(f"{name}: {value}")


agent.on_intent_partial(handle_intent)
agent.on_intent(handle_intent_full)


async def main():
    print("Chat with the agent (type 'exit' or 'quit' to stop)\n")

    while True:
        try:
            user_input = input("\nYou: ").strip()

            if user_input.lower() in ["exit", "quit", "q"]:
                print("Goodbye!")
                break

            if not user_input:
                continue

            print("\nAgent: ", end="", flush=True)
            await agent.run(user_input)
            print()

        except KeyboardInterrupt:
            print("\n\nGoodbye!")
            break
        except Exception as e:
            print(f"\nError: {e}")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Understanding the intent handlers

The application registers two handlers — `on_intent_partial` and `on_intent`. They serve different purposes and are worth understanding clearly.

**`on_intent_partial`** fires as the model is streaming its response. For `response_text` intents, each partial fires with a `delta` — the next chunk of text. You print it immediately to give the user a live streaming experience:

```python
def handle_intent(name: MainAgentIntentName, value: MainAgentIntentValue, agent_name: str):
    if name == "response_text":
        print(value.get("delta", ""), end="", flush=True)
```

**`on_intent`** fires when an intent is fully complete. For `response_text` we let streaming handle the output so we skip it here. For everything else — tool calls, tool results, custom intents like `Planing` and `Dialogue` — we print the name and value so you can see what the model is doing under the hood:

```python
async def handle_intent_full(name: MainAgentIntentName, value: MainAgentIntentValue, agent_name: str):
    if name != "response_text":
        print(f"{name}: {value}")
```

This gives you a clear view into the agent's behaviour during a session — which tools it called, what it planned, what questions it asked — without any extra instrumentation.

---

## Step 5 — Run the agent

With your virtual environment active and `.env` in place:

```bash
python index.py
```

Stop with `exit`, `quit`, `q`, or Ctrl+C.

![Starting the CLI](https://blog-images-cdn.henryantwi.me/auwgent-blog-images/screenshots/05-run-cli.png)

---

## How it fits together

![Data flow diagram](https://blog-images-cdn.henryantwi.me/auwgent-blog-images/screenshots/flow.png)

- **`auwgent_src/`** — your agent definition. Edit this when you want to change the agent's behaviour, tools, or prompts.
- **`auwgent generate`** — compiles the definition and updates `generated/`. Run this every time you change a `.agent` file.
- **`generated/main_types.py`** — the typed Python bindings. Never edit this by hand.
- **`tools.py`** — your tool implementations. Plain functions, no framework coupling.
- **`index.py`** — the application entry point. Wires everything together and runs the conversation loop.

---

## Troubleshooting

- **API errors** — Check that your `.env` file is in the project root and the variable names match exactly. Verify your keys at [Groq](https://console.groq.com/keys) and [Tavily](https://app.tavily.com/home).
- **Import errors for `generated`** — Run all commands from the project root. If you have edited any `.agent` files, run `auwgent generate` to regenerate the bindings.
- **Python version** — Use Python 3.11 or newer.

---

**Links:** [Auwgent docs](https://auwgent.snrraptopack.workers.dev) · [Repository](https://github.com/snrraptopack/a-simple-research-agent) · [Auwgent CLI on npm](https://www.npmjs.com/package/@snrraptopack/auwgent-cli) · [Groq](https://console.groq.com/keys) · [Tavily](https://app.tavily.com/home)
