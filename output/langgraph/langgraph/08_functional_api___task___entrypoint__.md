# Chapter 8: Functional API (@task, @entrypoint)

In the last chapter on the [Checkpointer](07_checkpointer_.md), we saw how LangGraph can save and load the state of our application, enabling persistence and long-running agents. So far, we have built all our graphs using the `StateGraph` class, explicitly adding nodes and edges. This approach is powerful and gives you fine-grained control over your workflow.

But what if your workflow is simpler? What if it just looks like a regular Python script where you call some functions, maybe in parallel, and then combine their results? For these cases, LangGraph offers a more direct, intuitive, and "Pythonic" way to build graphs: the Functional API.

## From Blueprints to Recipes

Think of the `StateGraph` API as creating a detailed architectural blueprint. You define every room (`node`) and every hallway (`edge`) explicitly. This is great for complex buildings with many interconnected parts.

The Functional API, on the other hand, is like writing a recipe. You just list the steps:
1.  Chop the vegetables (`@task`).
2.  Sauté the onions (`@task`).
3.  Combine everything and simmer.

You write your workflow as a set of Python functions, and LangGraph automatically wires them together into a powerful, parallelized graph behind the scenes. The main tools for this are two decorators: `@task` and `@entrypoint`.

*   `@task`: Marks a function as a single, independent computational step. Think of it as one instruction in your recipe.
*   `@entrypoint`: Marks the main function that defines the overall flow of your recipe. This is where you call your tasks.

## Building a Parallel Processing Pipeline

Let's build a simple application that takes a list of topics, generates a short sentence for each one in parallel, and then combines them. This "fan-out, fan-in" pattern is a perfect use case for the Functional API.

### 1. Define Your Task

First, we define our "worker" function. This function will take a single topic and return a sentence. We decorate it with `@task` to tell LangGraph that this is a node in our graph.

```python
from langgraph.func import task

@task
def generate_sentence(topic: str) -> str:
    """Generates a sentence for a given topic."""
    # In a real app, this might be an LLM call.
    return f"This is a sentence about {topic}."
```
That's it! `generate_sentence` is now a LangGraph task. When we call it from our main workflow, LangGraph will know how to manage it.

### 2. Define Your Entrypoint

Next, we define the main "recipe" function. This function will orchestrate the work. It takes the list of topics, calls our task for each one, and gathers the results. We decorate it with `@entrypoint`.

```python
from langgraph.func import entrypoint

@entrypoint
def process_topics(topics: list[str]) -> list[str]:
    """Processes a list of topics in parallel."""
    futures = [generate_sentence(topic) for topic in topics]
    
    # .result() waits for the task to finish and gets the output
    return [f.result() for f in futures]
```
Let's break this down:
*   `@entrypoint`: This decorator compiles the `process_topics` function into a runnable LangGraph application.
*   `generate_sentence(topic)`: This is the magic part. When you call a `@task` from within an `@entrypoint`, it **doesn't run immediately**. Instead, it returns a special "future" object. This future is a promise that the result will be available later.
*   `f.result()`: This is how you redeem the promise. Calling `.result()` on a future will pause the execution of the entrypoint until that specific task has finished and its output is ready.

Because the calls to `generate_sentence` don't block, LangGraph can schedule all of them to run at the same time, in parallel!

### 3. Run It!

The `@entrypoint` decorator has already turned our `process_topics` function into a complete, compiled graph. We can just call `.invoke()` on it directly.

```python
input_topics = ["dogs", "cats", "langgraph"]
output = process_topics.invoke(input_topics)

print(output)
```
**Output:**
```
['This is a sentence about dogs.', 'This is a sentence about cats.', 'This is a sentence about langgraph.']
```
It works! LangGraph took our simple Python-like code, created a graph with three parallel `generate_sentence` nodes, executed them, waited for them all to finish, and returned the collected results.

## What's Happening Under the Hood?

This might seem like magic, but it's a clever abstraction built on top of the same [Pregel](05_pregel_.md) engine we've been using all along.

1.  **Compilation:** When Python loads your code, the `@entrypoint` decorator immediately takes your `process_topics` function and compiles it into a `Pregel` graph object. Your `process_topics` variable no longer holds a regular function, but a fully runnable graph.
2.  **Invocation:** When you call `process_topics.invoke()`, the Pregel engine starts. It begins by running the code inside your `process_topics` function as the first step.
3.  **Task Scheduling:** Inside `process_topics`, the line `generate_sentence(topic)` is executed. This doesn't call your Python function directly. Instead, it calls a special `call` function provided by LangGraph. This function tells the Pregel engine: "Please add a new task to your to-do list: run the `generate_sentence` node with the argument `topic`." It then returns a future object, which is like a receipt for your request. This happens for all topics in the list.
4.  **Parallel Execution:** At the end of the first step, the Pregel engine looks at its to-do list. It sees three `generate_sentence` tasks that are ready to run and have no dependencies on each other. It executes all of them in parallel.
5.  **Result Collection:** Meanwhile, the `process_topics` function is paused at the `f.result()` line. This tells the Pregel engine to wait until the task associated with the future `f` is complete. Once a task finishes, Pregel makes its result available, and the `.result()` call returns the value.
6.  **Completion:** Once all the results have been collected, the `process_topics` function finishes its execution and returns the final list, which ends the graph run.

Here is a simplified diagram of the process:

```mermaid
sequenceDiagram
    participant User
    participant Entrypoint as "process_topics()"
    participant Pregel as Pregel Engine
    participant Task as "generate_sentence()"

    User->>Entrypoint: .invoke(["dogs", "cats"])
    Entrypoint->>+Pregel: Start Execution
    Pregel->>+Entrypoint: Run entrypoint code
    Entrypoint->>Pregel: Schedule Task: generate_sentence("dogs")
    Entrypoint->>Pregel: Schedule Task: generate_sentence("cats")
    Entrypoint->>Entrypoint: Wait on future.result()
    Pregel->>+Task: Execute("dogs")
    Pregel->>+Task: Execute("cats")
    Task-->>-Pregel: Return "sentence about dogs"
    Task-->>-Pregel: Return "sentence about cats"
    Pregel-->>-Entrypoint: Provide results
    Entrypoint-->>-Pregel: Return final list
    Pregel-->>-User: Return final list
```

### Diving into the Code

The logic that makes this possible lives in `langgraph/func/__init__.py`.

The `@entrypoint` decorator is a class. When you apply it to a function, its `__call__` method is triggered. This method is responsible for building the `Pregel` graph.

```python
# Simplified from libs/langgraph/langgraph/func/__init__.py
class entrypoint:
    # ...
    def __call__(self, func: Callable[..., Any]) -> Pregel:
        # Get the runnable version of the user's function
        bound = get_runnable_for_entrypoint(func)
        
        # Build and return a Pregel graph object
        return Pregel(
            nodes={
                func.__name__: PregelNode(bound=bound, ...)
            },
            # ... channel and stream configuration ...
        )
```
The `@task` decorator is simpler. It wraps your function in a `_TaskFunction` object.

```python
# Simplified from libs/langgraph/langgraph/func/__init__.py
def task(...):
    def decorator(func):
        # Return a callable object that holds the function
        # and its configuration (like retry policies).
        return _TaskFunction(func, ...)
    # ...
```
The final piece of the puzzle is the `call` function in `langgraph/pregel/_call.py`. When you call `generate_sentence(...)` inside the entrypoint, you are actually invoking this `call` function. It gets access to the current Pregel engine via the run's configuration and tells it to schedule the task.

## Conclusion

Congratulations! You've completed the introductory tour of LangGraph. You've now seen the two primary ways to build stateful, agentic applications.

*   **`StateGraph` API:** An explicit, class-based approach that gives you maximum control. It's like building with LEGO bricks—you define the state, add each node, and connect every edge manually. This is ideal for complex graphs with intricate branching and looping logic.
*   **Functional API (`@task`, `@entrypoint`):** A decorator-based approach that feels more like writing a standard Python script. It's more concise and automatically handles common patterns like parallel execution. This is perfect for workflows that can be expressed as a series of function calls.

You started by learning the core concepts of `StateGraph`, `Channels`, `Nodes & Edges`. You then explored powerful prebuilt components like `ToolNode` and learned about the `Pregel` engine that powers everything. Finally, you saw how `Interrupt` and `Checkpointer` enable human-in-the-loop workflows and persistent memory. With the Functional API, you now have a second, powerful tool in your toolkit.

You are now equipped with the fundamental knowledge to start building your own sophisticated, multi-step AI applications. Happy building

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)