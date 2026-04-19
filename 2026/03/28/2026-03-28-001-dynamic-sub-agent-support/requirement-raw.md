# Dynamic Sub Agent Support for Vaga Main Agent

## Requirement

The vv main agent should support dynamic running of sub agents with the following capabilities:

1. **Prompt-based Sub Task Definition**: Use prompts to define sub tasks that the main agent needs to accomplish
2. **Dynamic Parameter Preparation**: Dynamically prepare parameters for sub agents based on the task context
3. **Build and Run Sub Agent**: Build sub agent instances with the prepared parameters and execute them

## Key Features

- Main agent can receive a prompt describing work to be done
- Main agent analyzes the prompt and breaks it into sub tasks
- For each sub task, the main agent dynamically determines and prepares the required parameters
- Main agent builds/creates sub agent instances with the appropriate configuration
- Main agent runs the sub agents and collects their results
- Support for flexible, prompt-driven workflow orchestration
