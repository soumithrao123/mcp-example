# Assignment: Build and Call Your Own MCP Tool

In this assignment you will add a new tool to the MCP server in this project, then write a small program that discovers the tool and uses it. Everything runs in VS Code. You do not need Claude Desktop or any other AI app.

**What you will learn**

- How a Python function becomes an MCP **tool**
- How an MCP **client** discovers tools and calls them
- How the server keeps an **audit log** of every call

**Time:** about 45 minutes.

---

## Before you start

You need these installed:

- [Python 3.10 or newer](https://www.python.org/downloads/)
- [uv](https://docs.astral.sh/uv/getting-started/installation/) (the tool that installs and runs the project)
- [VS Code](https://code.visualstudio.com/)
- [Git](https://git-scm.com/downloads)

> **Tip:** If you use Anaconda and your terminal prompt starts with `(base)`, run `conda deactivate` first.

---

## Part 1: Get the project running

1. Open a terminal and run:

   ```bash
   git clone https://github.com/gitmystuff/mcp-example
   cd mcp-example
   ```

2. Open the folder in VS Code (**File → Open Folder** → choose `mcp-example`), then open a terminal inside VS Code (**Terminal → New Terminal**).

3. Install everything:

   ```bash
   uv sync
   ```

4. Run the demo:

   ```bash
   uv run python mcp_client_demo.py
   ```

   The first run takes a few minutes. When it finishes, scroll up to the top of the output. It should say **Found 8 tools**.

- [ ] I ran the demo and saw "Found 8 tools".

> **How this works:** `mcp_client_demo.py` is an MCP *client*. It starts the server (`mcp_ml_server.py`) in the background, asks it which tools it has, then calls them one by one. That is exactly what an AI app like Claude Desktop does. Here, a script does it instead.

---

## Part 2: Add a new tool to the server

The server has one tool for each model (`predict_logistic_regression` and `predict_knn`). You will add a tool called `predict_both` that runs **both** models in one call and reports whether they agree.

1. Open `mcp_ml_server.py`.

2. Scroll to the bottom. Find this line, near line 271:

   ```python
   if __name__ == "__main__":
   ```

3. Paste the code below on the empty lines **just above** it:

   ```python
   @server.tool()
   def predict_both(
       input_data: Dict[str, Any] = Field(description=INPUT_DATA_DESCRIPTION),
       request_id: str = RequestId,
   ) -> Dict[str, Any]:
       """Predict with BOTH models in one call and report whether they agree."""
       def work():
           lr = ml.predict_with_model(ml.load_model("logistic_regression"), input_data)
           knn = ml.predict_with_model(ml.load_model("knn"), input_data)
           return {"status": "success",
                   "logistic_regression": lr["predicted_class"],
                   "knn": knn["predicted_class"],
                   "models_agree": lr["predicted_class"] == knn["predicted_class"]}
       return _run("predict_both", request_id, work,
                   lambda r: {"models_agree": r["models_agree"]})
   ```

4. Save the file (**Ctrl+S**, or **Cmd+S** on a Mac).

**What each part does**

| Code | What it does |
| --- | --- |
| `@server.tool()` | Tells the MCP server "offer this function as a tool" |
| `def predict_both(...)` | The function name becomes the tool's name |
| `"""Predict with BOTH models..."""` | This description is what an AI reads to decide when to use the tool |
| `input_data`, `request_id` | The inputs the tool needs. `request_id` is a label you choose; it is saved in the log |
| `work()` | The actual job: run both models and compare their answers |
| `_run(...)` | A safety wrapper already in the file. It catches errors and logs every call |

- [ ] I added `predict_both` and saved the file.

---

## Part 3: Write a client that uses your tool

1. In VS Code, create a new file in the project folder called `my_client.py`.

2. Paste this code into it and save:

   ```python
   # my_client.py
   # A small MCP client: it starts the server, asks what tools it has,
   # then uses predict_both on every test patient.
   import asyncio

   from agents.mcp import MCPServerStdio
   from mcp_client_demo import params, call   # reuse the demo's connection settings


   async def main():
       async with MCPServerStdio(params=params, client_session_timeout_seconds=120) as server:
           # 1. Ask the server what it can do (this is MCP's "tools/list")
           tools = await server.list_tools()
           print(f"The server offers {len(tools)} tools:")
           for tool in tools:
               print("  -", tool.name)

           # 2. Make sure the models are trained
           await call(server, "train_models", {"request_id": "hw-train"})

           # 3. Check every test patient with your new tool (this is MCP's "tools/call")
           disagreements = []
           for i in range(114):
               sample = await call(server, "get_sample_input", {"index": i, "request_id": f"hw-sample-{i}"})
               result = await call(server, "predict_both", {
                   "input_data": sample["input_data"],
                   "request_id": f"hw-both-{i}",
               })
               if not result["models_agree"]:
                   disagreements.append(i)
                   print(f"Patient {i}: true = {sample['true_class']}, "
                         f"logistic regression = {result['logistic_regression']}, knn = {result['knn']}")

           print(f"\nThe models disagree on {len(disagreements)} of 114 test patients.")


   asyncio.run(main())
   ```

3. Run it:

   ```bash
   uv run python my_client.py
   ```

   It takes about a minute, because it checks 114 patients.

- [ ] My client listed the tools and printed the patients where the models disagree.

> **Notice:** You did not change anything in the client code to "register" `predict_both`. The client asked the server what tools it has, and the new tool was simply there. That is the main idea of MCP: tools are *discovered*, not hard-coded.

---

## Part 4: Check the audit log

The server records every tool call in a log file. Run this to see the last 10 calls:

```bash
uv run python -c "import pandas as pd; print(pd.read_parquet('data_parquet/tool_audit.parquet')[['request_id','tool','status']].tail(10))"
```

You should see your `hw-both-...` calls at the bottom.

- [ ] I can see my `predict_both` calls in the audit log.

---

## Questions

Answer these in a file called `answers.md`.

1. How many tools did the server list in Part 1, and how many in Part 3? What did you have to change in the **client** to make the new tool appear?
2. On how many of the 114 test patients do the two models disagree? For each of those patients, which model was right?
3. Overall, which model was right more often when they disagreed?
4. In your own words: why does the description in triple quotes (`"""..."""`) matter when an AI app uses this tool?
5. Why is it useful for the server to log every tool call?

---

## What to submit

- `answers.md` with your answers
- A screenshot of the terminal output from Part 3

---

## Stretch goal (optional)

The KNN model compares patients by measuring the distance between their numbers. But the measurements use very different scales: some are over 1,000, others are around 0.1. The big numbers drown out the small ones.

Fix this by putting every measurement on the same scale before KNN sees it:

1. Open `ml_pipeline.py`. Near the top, with the other `from sklearn...` lines, add:

   ```python
   from sklearn.pipeline import make_pipeline
   from sklearn.preprocessing import StandardScaler
   ```

2. Find the function `train_knn` (near line 144) and change this line:

   ```python
   model = KNeighborsClassifier(n_neighbors=n_neighbors)
   ```

   to:

   ```python
   model = make_pipeline(StandardScaler(), KNeighborsClassifier(n_neighbors=n_neighbors))
   ```

3. Save and run `uv run python my_client.py` again.

**Stretch question:** How many patients do the models disagree on now? Did KNN get better or worse?

---

## If something goes wrong

| Problem | Fix |
| --- | --- |
| `uv: command not found` (or "not recognized") | Install uv, then close and reopen VS Code |
| The client only lists 8 tools | You didn't save `mcp_ml_server.py`, or the code was pasted below `if __name__ == "__main__":` instead of above it |
| `IndentationError` | The pasted code must start at the far left of the line, with no spaces before `@server.tool()` |
| `KeyError: 'models_agree'` | The tool returned an error. Add `print(result)` just above the `if not result[...]` line to see the message |
| The server prints nothing when you run `mcp_ml_server.py` on its own | That's normal. It is waiting for a client. Press **Ctrl+C** to stop it |

To start over at any time, run `git checkout .` to undo your changes to the project files. Your new `my_client.py` file is kept.
