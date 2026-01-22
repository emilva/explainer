# RLM vs. Standard Agents (Claude Code, Codex, etc.)

This document outlines the fundamental differences between **Recursive Language Models (RLMs)** and standard agentic workflows (like Claude Code, context folding, etc.), based on research by Alex Zhang and his tweet https://x.com/a1zhang/status/2014337263287804260

## The Core Distinction: Symbolic Recursion

The critical difference is not about saving files or using a REPL; it is about **where the control flow lives**.

*   **Standard Agents (Claude Code/Codex):** The **LLM** is the control flow. To iterate over tasks, the LLM must sequentially generate a tool call for every single iteration.
*   **RLM:** The **Interpreter (REPL)** is the control flow. The LLM writes a program (a loop), and the *program* handles the iteration and spawning of sub-agents.

This enables **Symbolic Recursion**: the ability to embed a recursive call to an AI model *inside* a block of logic (like a `for` loop or `if` statement) that is executed by a computer, not the AI.

---

## Visualizing the Comparison: The "1 Million Files" Task

### 1. Standard Agent Case (Claude Code / Codex)
*Reliance on "Linear Generation"*

In this scenario, the agent is manually "driving" the iteration. It has to stay awake and focused enough to output a tool call for every single file.

```text
[Context Window]
User: "Check 1M files for a weird property."

[Model Generation Cycle]
Model: Use Tool -> scan_file("file_1")
   <-- Tool Output: "Nothing found"
Model: Use Tool -> scan_file("file_2")
   <-- Tool Output: "Nothing found"
Model: Use Tool -> scan_file("file_3")
   <-- Tool Output: "Nothing found"
...
... (repeats 999,997 more times) ...
...
Model: *Hallucinates or hits token limit* "I'm done."
```

**The Problem:** The LLM acts as the for-loop. If the context window fills up, or the model loses "focus," the loop breaks. "Context Folding" here is usually just summarizing history, which is lossy and prone to error at scale.

### 2. The RLM Case
*Reliance on "Programmatic Execution"*

The agent acts as the *programmer*, offloading repetitive logic to a runtime environment that is guaranteed to be correct (e.g., Python).

```text
[Context Window]
User: "Check 1M files for a weird property."

[Model Generation Cycle]
Model: *Writes Code*
       ```python
       results = []
       files = list_all_files()
       
       # The LOOP is here, in code, not in the model's head
       for f in files: 
           # SYMBOLIC RECURSION: The code calls the sub-agent
           if sub_agent_check(f, "weird property"):
               results.append(f)
       
       print(results)
       ```

[REPL Execution Cycle]
REPL: Runs the loop. 
      - Spawns isolated sub-agent for file_1
      - Spawns isolated sub-agent for file_2
      ...
      - Spawns isolated sub-agent for file_1,000,000
      (This happens outside the main model's context)

[Final Context State]
Model: "Here is the list of files matching your criteria: [list]"
```

**The Solution:** The main model's context **never grows** regardless of the number of files. It wrote 5 lines of code, and the REPL performed 1 million operations. The context was effectively "folded" into the execution state of the variable `results`.

---

## Summary Table

| Feature | Standard Agents (CC/Codex) | Recursive Language Models (RLM) |
| :--- | :--- | :--- |
| **Control Flow** | The Model (Probabilistic) | The REPL (Deterministic) |
| **Iteration** | Model outputs tool calls sequentially | Model writes a `for` loop |
| **Reliability** | Degrades as task count increases | Constant (Code works for 1 or 1M items) |
| **Sub-Agents** | External tools called by the model | Functions embedded in the programming language |
| **Context** | Grows linearly with steps (must be pruned) | Constant (only the code and final result) |
