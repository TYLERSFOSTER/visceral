# VISCERAL Internal Architecture Documentation

## OpenClaw + OpenAI Agent Runtime Model

### Version 0.1 --- High‑Level Operational Overview

------------------------------------------------------------------------

# Core mental model

There are three layers:

    YOU (director)
       ↓
    OpenClaw gateway (persistent agent runtime)
       ↓
    Provider models (OpenAI via API key)

OpenClaw is the runtime. OpenAI is the brain.

OpenClaw handles:

-   long-running agents
-   memory persistence
-   filesystem workspace
-   tool execution (browser, shell, files)
-   scheduling
-   multi-agent separation

OpenAI handles:

-   reasoning
-   planning
-   language understanding
-   tool selection decisions

------------------------------------------------------------------------

# Division architecture

Each division becomes an independent agent workspace.

Physically:

    visceral/
    ├── division_0_office/
    ├── division_1_rl/
    ├── division_2_robotics/

Logically:

    Agent: office
    Workspace: division_0_office/

    Agent: rl
    Workspace: division_1_rl/

    Agent: robotics
    Workspace: division_2_robotics/

Each agent has:

-   its own memory
-   its own files
-   its own tasks
-   its own planning loop

All share the same OpenAI API key through OpenClaw.

------------------------------------------------------------------------

# Agent execution loop

The gateway exposes an agent runtime that continuously performs this
loop:

    loop:
        read agent state
        read memory
        read workspace files
        call OpenAI model
        model decides next action
        OpenClaw executes action
        update memory
        repeat

Actions include:

-   browser search
-   HTTP requests
-   writing files
-   reading files
-   executing commands
-   scheduling tasks

This is what turns OpenAI into an autonomous R&D worker.

------------------------------------------------------------------------

# Internet searching capability

OpenAI models alone cannot browse the internet from your infrastructure.

They can only reason about text you send them.

OpenClaw enables actual browsing through tools like:

-   browser automation
-   HTTP fetch
-   crawling tools
-   plugin integrations

Without OpenClaw, OpenAI cannot autonomously search the web for your
company.

With OpenClaw, it can.

------------------------------------------------------------------------

# Provider configuration model

One OpenAI provider shared by three agents.

Conceptually:

    OpenClaw gateway
        provider: OpenAI
        api_key: YOUR_KEY

    agents:
        office:
            workspace: division_0_office/

        rl:
            workspace: division_1_rl/

        robotics:
            workspace: division_2_robotics/

Each agent independently calls OpenAI via the gateway.

------------------------------------------------------------------------

# Example execution flow

Example: robotics agent investigating robot arm suppliers

Flow:

    robotics agent wakes up
        ↓
    reads workspace files
        ↓
    asks OpenAI:
        "what should I research next?"
        ↓
    OpenAI replies:
        "search for harmonic drive suppliers"
        ↓
    OpenClaw executes browser search
        ↓
    collects results
        ↓
    feeds results back to OpenAI
        ↓
    OpenAI writes report
        ↓
    OpenClaw saves report in division_2_robotics/

This repeats indefinitely or on schedule.

------------------------------------------------------------------------

# Minimal configuration requirements

Only three essential things:

1)  OpenAI API key

2)  OpenClaw provider configuration

3)  Agent workspace directories

That is sufficient.

No additional OpenAI configuration needed.

------------------------------------------------------------------------

# Recommended model assignments

Default:

    gpt-5

Optional split:

    office: gpt-5-mini
    rl: gpt-5
    robotics: gpt-5

This reduces cost for administrative workloads.

------------------------------------------------------------------------

# Key architectural principle

OpenAI models cannot autonomously run persistent divisions without a
runtime.

OpenClaw provides:

-   persistence
-   filesystem access
-   browser access
-   scheduling
-   memory

OpenAI provides cognition.

Both are required.

------------------------------------------------------------------------

# Operational readiness state

Gateway running: yes

Provider configured: pending

Agents created: pending

System ready for agent deployment.

------------------------------------------------------------------------

# Next operational step

Create first agent and validate end-to-end execution.

Once validated, replicate across all three divisions.

------------------------------------------------------------------------

# End of document
