# Enhanced Observability for ADK with AgentOps

AgentOps provides a powerful observability platform for testing, debugging, and monitoring AI agents and LLM applications, including those built with Google's Agent Development Kit (ADK). By integrating AgentOps, developers can gain deep insights into their ADK agent's behavior, LLM interactions, and tool usage.

For more general information about AgentOps and its features, please visit the [official AgentOps documentation](https://docs.agentops.ai/v2/introduction).

## Why AgentOps for ADK?

Google ADK includes its own OpenTelemetry-based tracing system, primarily aimed at providing developers with a way to trace the basic flow of execution within their agents. AgentOps enhances this by offering a dedicated and more comprehensive observability platform with:

*   **Unified Tracing:** Consolidate traces from ADK and other components of your AI stack.
*   **Rich Visualization:** Intuitive dashboards to visualize agent execution flow, LLM calls, and tool performance.
*   **Detailed Debugging:** Drill down into specific spans, view prompts, completions, token counts, and errors.
*   **Performance Monitoring:** Track latencies, costs (via token usage), and identify bottlenecks.
*   **Simplified Setup:** Get started with just a few lines of code.

## Getting Started with AgentOps and ADK

Integrating AgentOps into your ADK application is straightforward:

1.  **Install AgentOps:**
    ```bash
    pip install agentops
    ```

2.  **Initialize AgentOps:**
    Add the following lines at the beginning of your ADK application script (e.g., your main Python file running the ADK `Runner`):

    ```python
    import agentops
    import os
    from dotenv import load_dotenv

    # Load environment variables (optional, if you use a .env file for API keys)
    load_dotenv()

    agentops.init(
        os.getenv("AGENTOPS_API_KEY"), # Your AgentOps API Key
        trace_name="my-adk-app-trace"  # Optional: A name for your trace
        # auto_start_session=True is the default.
        # Set to False if you want to manually control session start/end.
    )
    ```

    !!! tip "API Key"
        You can find your AgentOps API key on your [AgentOps Dashboard](https://app.agentops.ai/) after signing up. It's recommended to set it as an environment variable (`AGENTOPS_API_KEY`).

Once initialized, AgentOps will automatically begin instrumenting your ADK application.

## How AgentOps Instruments ADK

AgentOps employs a sophisticated strategy to provide seamless observability without conflicting with ADK's native telemetry:

1.  **Neutralizing ADK's Native Telemetry:**
    AgentOps detects ADK and intelligently patches ADK's internal OpenTelemetry tracer (typically `trace.get_tracer('gcp.vertex.agent')`). It replaces it with a `NoOpTracer`, ensuring that ADK's own attempts to create telemetry spans are effectively silenced. This prevents duplicate traces and allows AgentOps to be the authoritative source for observability data.

2.  **AgentOps-Controlled Span Creation:**
    With ADK's tracing disabled, AgentOps takes control by wrapping key ADK methods to create a logical hierarchy of spans:

    *   **Agent Execution Spans (e.g., `adk.agent.MySequentialAgent`):**
        When an ADK agent (like `BaseAgent`, `SequentialAgent`, or `LlmAgent`) starts its `run_async` method, AgentOps initiates a parent span for that agent's execution.

    *   **LLM Interaction Spans (e.g., `adk.llm.gemini-pro`):**
        For calls made by an agent to an LLM (via ADK's `BaseLlmFlow._call_llm_async`), AgentOps creates a dedicated child span, typically named after the LLM model. This span captures request details (prompts, model parameters) and, upon completion (via ADK's `_finalize_model_response_event`), records response details like completions, token usage, and finish reasons.

    *   **Tool Usage Spans (e.g., `adk.tool.MyCustomTool`):**
        When an agent uses a tool (via ADK's `functions.__call_tool_async`), AgentOps creates a single, comprehensive child span named after the tool. This span includes the tool's input parameters and the result it returns.

3.  **Rich Attribute Collection:**
    AgentOps cleverly reuses ADK's internal data extraction logic. It patches ADK's specific telemetry functions (e.g., `google.adk.telemetry.trace_tool_call`, `trace_call_llm`). The AgentOps wrappers for these functions take the detailed information ADK gathers and attach it as attributes to the *currently active AgentOps span*.

## Visualizing Your ADK Agent in AgentOps

When you instrument your ADK application with AgentOps, you gain a clear, hierarchical view of your agent's execution in the AgentOps dashboard.

1.  **Initialization:**
    When `agentops.init()` is called (e.g., `agentops.init(trace_name="my_adk_application")`), an initial parent span is created if `auto_start_session=True` (the default). This span, often named like `my_adk_application.session`, will be the root for all operations within that trace.

2.  **ADK Runner Execution:**
    When an ADK `Runner` executes a top-level agent (e.g., a `SequentialAgent` orchestrating a workflow), AgentOps creates a corresponding agent span under the session trace. This span will reflect the name of your top-level ADK agent (e.g., `adk.agent.YourMainWorkflowAgent`).

3.  **Sub-Agent and LLM/Tool Calls:**
    As this main agent executes its logic, including calling sub-agents, LLMs, or tools:
    *   Each **sub-agent execution** will appear as a nested child span under its parent agent.
    *   Calls to **Large Language Models** will generate further nested child spans (e.g., `adk.llm.<model_name>`), capturing prompt details, responses, and token usage.
    *   **Tool invocations** will also result in distinct child spans (e.g., `adk.tool.<your_tool_name>`), showing their parameters and results.

This creates a waterfall of spans, allowing you to see the sequence, duration, and details of each step in your ADK application. All relevant attributes, such as LLM prompts, completions, token counts, tool inputs/outputs, and agent names, are captured and displayed.

For a practical demonstration, you can explore a sample Jupyter Notebook that illustrates a human approval workflow using Google ADK and AgentOps:
[Google ADK Human Approval Example on GitHub](https://github.com/AgentOps-AI/agentops/blob/main/examples/google_adk_example/adk_human_approval_example.ipynb).

This example showcases how a multi-step agent process with tool usage is visualized in AgentOps.

### Example AgentOps Dashboard View

The following image shows an example of how a typical multi-step ADK agent's execution would appear in the AgentOps dashboard. You can see the hierarchical structure of spans, including the main agent workflow, individual sub-agents, LLM calls, and tool executions.

![AgentOps Dashboard showing an ADK trace with nested agent, LLM, and tool spans.](../assets/agentops-adk-trace-example.jpg)

*AgentOps dashboard displaying a trace from an ADK application. Note the clear hierarchy: the main workflow agent span contains child spans for various sub-agent operations, LLM calls, and tool executions.*

## Benefits

*   **Effortless Setup:** Minimal code changes for comprehensive ADK tracing.
*   **Deep Visibility:** Understand the inner workings of complex ADK agent flows.
*   **Faster Debugging:** Quickly pinpoint issues with detailed trace data.
*   **Performance Optimization:** Analyze latencies and token usage.

By integrating AgentOps, ADK developers can significantly enhance their ability to build, debug, and maintain robust AI agents. 