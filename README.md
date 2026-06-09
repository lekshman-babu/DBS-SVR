# DBS-SVR

**DBS-SVR** is a Python-based agent orchestration framework designed to reduce hallucinations, repeated tool calls, unsafe actions, and unnecessary token usage in ReAct-style tool-calling agents.

Instead of allowing an agent to directly choose a tool call or final response in one step, DBS-SVR breaks each decision into a structured reasoning pipeline:

```text
StateBuilder -> BudgetScope -> Simulator -> Verifier -> Replanner -> FinalOutput
```

The project is especially useful for benchmark environments such as tau-bench / tau-trait, customer-support agents, healthcare-style task agents, and any tool-calling AI system where action reliability matters.

---

## Overview

Large language model agents often make mistakes when interacting with tools. Common problems include:

- Calling the same tool repeatedly
- Using unsupported or invalid tool names
- Inventing missing tool arguments
- Taking risky actions without confirmation
- Producing malformed JSON actions
- Spending too many tokens on simple tasks
- Responding before enough information is known

DBS-SVR addresses these issues through a test-time scaling approach. It separates the agent's reasoning process into specialized modules that build state, decide reasoning budget, simulate actions, verify them, replan when needed, and finally produce a valid environment action.

---

## Key Features

- **Structured state tracking**  
  Extracts the user goal, known facts, missing facts, constraints, previous actions, risk level, and current subgoal.

- **Dynamic reasoning budget**  
  Uses a low-cost path for simple tasks and a high-budget reasoning path for complex, risky, or incomplete tasks.

- **Action simulation before execution**  
  Proposes a candidate next action without directly executing it.

- **Verification layer**  
  Checks whether the proposed action is valid, safe, goal-directed, non-repetitive, and compatible with the available tools.

- **Replanning support**  
  Revises the plan when the verifier rejects an action.

- **Loop prevention**  
  Detects repeated tool calls and repeated response loops.

- **Strict final action formatting**  
  Produces only the JSON action format expected by the external environment.

---

## Repository Structure

```text
DBS-SVR/
│
├── chat_react_agent.py
│
└── DBSVR/
    ├── orchestrator.py
    │
    └── Prompts/
        ├── BudgetScope.yaml
        ├── FinalOutput.yaml
        ├── Replan.yaml
        ├── Simulator.yaml
        ├── StateBuilder.yaml
        └── Verifier.yaml
```

---

## File Descriptions

### `chat_react_agent.py`

This file acts as the adapter between the external agent environment and the DBS-SVR orchestration system.

It defines a `ChatReActAgent` class that:

- Builds the system prompt from the task environment
- Initializes the DBS-SVR orchestrator
- Sends conversation history into the orchestrator
- Parses the returned action JSON
- Executes the action inside the environment
- Tracks reward, messages, and task completion status

This file is mainly responsible for connecting DBS-SVR to a ReAct-style benchmark or tool-calling environment.

---

### `DBSVR/orchestrator.py`

This is the core engine of the project.

The orchestrator coordinates multiple specialized LLM-driven modules:

1. StateBuilder
2. BudgetScope
3. Simulator
4. Verifier
5. Replanner
6. FinalOutput

For each user turn, the orchestrator builds a structured state, determines whether the task needs low or high reasoning budget, simulates a candidate action, verifies that action, replans if needed, and finally emits a valid executable action.

The orchestrator also includes fallback behavior for repeated failures or repeated response loops.

---

## Prompt Modules

### `StateBuilder.yaml`

Builds a structured JSON representation of the conversation.

It extracts:

- User goal
- Current subgoal
- Known facts
- Missing facts
- User constraints
- Actions already taken
- Risk level
- Intent type

This module helps the agent avoid forgetting previously established information and prevents it from asking for facts that are already known.

---

### `BudgetScope.yaml`

Decides whether the current turn should use a low or high reasoning budget.

The **LOW** path is used for simple, low-risk tasks where enough information is already available.

The **HIGH** path is used when:

- Required facts are missing
- Tool use is needed
- The action has side effects
- The task involves multiple constraints
- The task is risky or irreversible
- The agent may need verification or replanning

This helps reduce token usage while preserving reliability for difficult tasks.

---

### `Simulator.yaml`

Proposes one candidate next action.

The simulator does not execute the action. Instead, it decides what the agent should do next, such as:

- Calling a tool
- Asking the user for missing information
- Responding with a final message

It is designed to avoid repeating the same tool call with the same arguments.

---

### `Verifier.yaml`

Checks whether the simulated action should be accepted.

The verifier evaluates the candidate action using conditions such as:

- Does the action advance the current goal?
- Are all required inputs available?
- Does the selected tool exist?
- Are the tool arguments valid?
- Is the action safe?
- Is the action repetitive?
- Does the action require user confirmation?

If the action fails verification, the system moves to the replanning step.

---

### `Replan.yaml`

Revises the plan when verification fails.

The replanner must change something meaningful, such as:

- Asking for a missing fact
- Choosing a different tool
- Updating the subgoal
- Avoiding a repeated action
- Escalating when the task cannot be completed safely

This module prevents the agent from getting stuck in invalid or repetitive behavior.

---

### `FinalOutput.yaml`

Converts the verified plan into the exact action format required by the environment.

The output must follow this format:

```text
Action: {"name": "...", "arguments": {...}}
```

The final output module prevents:

- Invalid JSON
- Extra commentary
- Markdown formatting
- Placeholder arguments
- Unsupported tool names
- Invented values

---

## Architecture

The DBS-SVR decision process can be summarized as:

```text
Conversation History
        ↓
StateBuilder
        ↓
Structured State
        ↓
BudgetScope
        ↓
LOW Budget ----------------> FinalOutput
        ↓
HIGH Budget
        ↓
Simulator
        ↓
Verifier
        ↓
Accepted? ---- Yes --------> FinalOutput
        ↓
        No
        ↓
Replan
        ↓
Retry Simulation / Verification
        ↓
Final Action
```

---

## Example Output Format

DBS-SVR is designed to emit actions in the following structure:

```text
Action: {"name": "respond", "arguments": {"content": "Could you please provide the missing order ID?"}}
```

Example tool call:

```text
Action: {"name": "get_order_details", "arguments": {"order_id": "12345"}}
```

The exact tool names and arguments depend on the environment where DBS-SVR is used.

---

## Installation

Clone the repository:

```bash
git clone https://github.com/lekshman-babu/DBS-SVR.git
cd DBS-SVR
```

Create and activate a virtual environment:

```bash
python -m venv venv
source venv/bin/activate
```

For Windows:

```bash
python -m venv venv
venv\Scripts\activate
```

Install required dependencies:

```bash
pip install litellm pyyaml
```

Depending on the benchmark or environment used, additional packages may be required.

---

## Configuration

DBS-SVR uses LiteLLM for model calls.

Set your model provider API key before running the agent.

Example for OpenAI-compatible usage:

```bash
export OPENAI_API_KEY="your_api_key_here"
```

For Windows PowerShell:

```powershell
setx OPENAI_API_KEY "your_api_key_here"
```

The model name can be configured inside the orchestrator or passed through the agent setup depending on your integration.

---

## Usage

A simplified usage pattern looks like this:

```python
from DBSVR.orchestrator import DBSVRAgent

agent = DBSVRAgent(
    model="gpt-4o-mini"
)

messages = [
    {"role": "system", "content": "You are a helpful tool-using assistant."},
    {"role": "user", "content": "I need help checking my order status."}
]

result = agent.orchestrate(messages)

print(result)
```

Expected output:

```text
Action: {"name": "respond", "arguments": {"content": "Please provide your order ID so I can check the status."}}
```

---

## Integration with ReAct Agents

The `chat_react_agent.py` file provides an integration wrapper for ReAct-style environments.

Instead of directly asking the LLM for the next action, the agent sends the current conversation state into DBS-SVR:

```python
action_text = self.dbsvr.orchestrate(messages)
```

The returned action is then parsed and executed in the environment.

This allows DBS-SVR to act as a safer reasoning layer between the LLM and the environment tools.

---

## Use Cases

DBS-SVR can be used for:

- Tool-calling AI agents
- Customer support automation
- Healthcare or appointment-management agents
- Order-management agents
- Billing and account-support agents
- Agent benchmark experiments
- Tau-bench / tau-trait style evaluations
- Reducing hallucinated tool arguments
- Preventing repeated tool calls
- Improving action safety in agent systems

---

## Why DBS-SVR Matters

Standard LLM agents often combine reasoning, planning, tool selection, and final response generation into a single step. This can lead to unreliable behavior.

DBS-SVR separates those responsibilities into multiple stages.

This makes the agent:

- More reliable
- Easier to debug
- Safer around tool use
- Less likely to hallucinate
- Less likely to repeat actions
- More cost-efficient on simple tasks

---

## Limitations

- The framework depends on the quality of the underlying LLM.
- Complex tasks may still require multiple retries.
- Tool schemas must be clearly defined by the environment.
- The current implementation is primarily designed around ReAct-style benchmark environments.
- Some configuration may need to be adapted for different model providers or agent frameworks.

---

## Future Improvements

Possible future enhancements include:

- Adding automated tests
- Adding benchmark result comparisons
- Supporting more agent frameworks
- Adding structured logging
- Adding configurable retry limits
- Adding cost and latency dashboards
- Improving prompt modularity
- Adding examples for different tool environments

---

## Project Summary

DBS-SVR is an experimental agent-orchestration framework that improves ReAct-style tool-calling agents by adding structured state tracking, dynamic budget selection, action simulation, verification, replanning, and strict final output generation.

Its main goal is to reduce hallucinations, invalid tool calls, repeated actions, unsafe behavior, and unnecessary token usage in agentic AI systems.
