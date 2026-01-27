# Browser Use Architecture Guide

This guide provides a comprehensive overview of the `browser-use` library's architecture, explaining how it integrates browser automation, DOM processing, and LLMs to create an autonomous browser agent.

## 1. Core Architecture Overview

At a high level, `browser-use` operates as a continuous feedback loop:

1.  **Observe**: The **Agent** requests the current state from the **BrowserSession**.
2.  **Parse**: The **DomService** constructs a rich DOM tree, which `DOMTreeSerializer` converts into a simplified, LLM-friendly string representation.
3.  **Think**: The **MessageManager** constructs a prompt (including task, history, and browser state) and sends it to the **LLM**.
4.  **Act**: The **LLM** returns an action (e.g., click, type, scroll). The **Agent** uses **Tools** to execute this action via the **BrowserSession**.
5.  **Repeat**: The loop continues until the task is completed or the maximum number of steps is reached.

```mermaid
graph TD
    User[User Task] --> Agent
    Agent -->|Step Loop| MessageManager

    subgraph Browser Automation
        BrowserSession -->|CDP| Chrome[Chrome Browser]
        DomService -->|Builds Tree| BrowserSession
    end

    subgraph Processing
        DomService -->|EnhancedDOMTreeNode| DOMTreeSerializer
        DOMTreeSerializer -->|String State| MessageManager
    end

    subgraph AI
        MessageManager -->|Prompt| LLM[LLM (e.g. GPT-4o)]
        LLM -->|Action JSON| Tools
    end

    Tools -->|Execute| BrowserSession
```

---

## 2. Component Breakdown

### 2.1 Agent (`browser_use.agent`)

The `Agent` class is the main entry point and orchestrator.

*   **Responsibility**: Manages the execution loop, handles errors, and coordinates between the browser, LLM, and tools.
*   **Key Methods**:
    *   `run()`: The main loop. Executes steps until `max_steps` is reached.
    *   `step()`: Executes a single iteration:
        1.  Calls `_prepare_context()` to get the current browser state.
        2.  Calls `_get_next_action()` to query the LLM.
        3.  Calls `_execute_actions()` to perform the chosen action.
*   **State Management**: Maintains `AgentHistoryList` (actions and results) and `AgentState`.

### 2.2 Browser Session (`browser_use.browser`)

The `BrowserSession` manages the connection to the browser instance and handles low-level automation.

*   **CDP Integration**: Unlike many libraries that use Playwright or Selenium, `browser-use` interacts directly with the **Chrome DevTools Protocol (CDP)** via `cdp_use`. This allows for deeper control, such as accessing the accessibility tree and handling network events directly.
*   **Key Classes**:
    *   `BrowserSession`: Manages the browser process (local or remote), tabs (targets), and CDP connection.
    *   `BrowserProfile`: Configuration for the browser (launch args, proxy, user agent, extensions).
*   **Watchdogs**: It uses various "watchdogs" to monitor events like downloads, navigation, and crashes.

### 2.3 DOM Service (`browser_use.dom`)

This component is responsible for "seeing" the page. It bridges the gap between raw HTML and what the LLM needs to understand the page structure.

*   **DomService**:
    *   Uses CDP to fetch the DOM tree and the **Accessibility Tree** (AXTree).
    *   Constructs a tree of `EnhancedDOMTreeNode` objects.
    *   Calculates visibility, coordinates, and interactivity for each element.
*   **DOMTreeSerializer**:
    *   Converts the `EnhancedDOMTreeNode` tree into a simplified string format.
    *   Filters out invisible or irrelevant elements.
    *   Assigns unique **indices** to interactive elements (e.g., `[12] <button>Submit</button>`). The LLM uses these indices to interact with elements.

### 2.4 Tools (`browser_use.tools`)

The `Tools` component defines the actions the agent can perform.

*   **Registry**: Actions are registered in a `Registry` and exposed to the LLM as function calls (or structured output).
*   **Built-in Actions**:
    *   `click(index=...)`: Clicks an element by its assigned index.
    *   `type(index=..., text=...)`: Types text into an input field.
    *   `scroll(amount=...)`: Scrolls the page.
    *   `navigate(url=...)`: Goes to a URL.
    *   `extract(query=...)`: Uses a secondary LLM call to extract specific information from the page content.
    *   `done(text=...)`: Signals task completion.
*   **Execution**: The `Tools` class receives the action model from the LLM and calls the corresponding method in `BrowserSession`.

### 2.5 Message Manager (`browser_use.agent.message_manager`)

Manages the context window and constructs messages for the LLM.

*   **Responsibility**:
    *   Maintains the `SystemPrompt` (instructions on how to behave).
    *   Manages **History**: Compresses or truncates conversation history to fit context limits.
    *   Constructs **State Messages**: Combines the current DOM state (from `DOMTreeSerializer`), screenshots, and recent actions into a message for the LLM.
    *   **Privacy**: Filters sensitive data (credentials) from the history before sending to the LLM.

---

## 3. LLM Integration

`browser-use` is model-agnostic but optimized for chat models that support function calling or structured output.

*   **Interface**: It uses `langchain`'s `BaseChatModel` interface, allowing integration with OpenAI (`ChatOpenAI`), Anthropic (`ChatAnthropic`), and others.
*   **Structured Output**: The agent expects the LLM to return structured data (JSON) conforming to the `AgentOutput` schema, which includes:
    *   `current_state`: The agent's internal thought process.
    *   `action`: A list of actions to execute.
*   **Vision**: If the model supports vision (e.g., GPT-4o), screenshots are included in the prompt to enhance understanding.

## 4. Execution Flow Example

1.  **Start**: `agent = Agent(task="Go to google.com and search for 'browser-use'", llm=...)`
2.  **Step 1**:
    *   **Browser**: Opens. `BrowserSession` connects via CDP.
    *   **DOM**: `DomService` sees an empty tab or initial page.
    *   **LLM**: Receives state. Decides to call `navigate(url="https://google.com")`.
    *   **Tools**: Executes navigation.
3.  **Step 2**:
    *   **Browser**: Page loads.
    *   **DOM**: `DomService` parses Google homepage. `DOMTreeSerializer` identifies the search box as index `[5]`.
    *   **LLM**: Receives state string showing `[5] <input type="text" ...>`. Decides to call `type(index=5, text="browser-use")` and then `click(index=...)`.
    *   **Tools**: Executes typing and clicking.
4.  **Completion**: Agent finds the result, calls `done()`.
