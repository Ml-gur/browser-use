# How `browser-use` Calls the LLM

The `browser-use` library orchestrates LLM calls through a modular system involving the `Agent` class, a generic `BaseChatModel` interface, and provider-specific implementations.

## 1. High-Level Orchestration: `Agent` Class

The core logic resides in `browser_use/agent/service.py`. The `Agent` class manages the interaction loop.

The method `get_model_output` is responsible for calling the LLM:

```python
async def get_model_output(self, input_messages: list[BaseMessage]) -> AgentOutput:
    # ... (URL processing)

    # Prepare arguments, passing the AgentOutput class as the expected format
    kwargs: dict = {'output_format': self.AgentOutput, 'session_id': self.session_id}

    try:
        # Call the LLM asynchronously
        response = await self.llm.ainvoke(input_messages, **kwargs)
        parsed: AgentOutput = response.completion

        # ... (Post-processing)
        return parsed
```

- **`self.llm`**: This is an instance of a class adhering to the `BaseChatModel` protocol (e.g., `ChatOpenAI`, `ChatAnthropic`).
- **`input_messages`**: A list of messages (System, User, Assistant) forming the conversation history.
- **`output_format`**: The `AgentOutput` Pydantic model (defined in `browser_use/agent/views.py`) which defines the expected structure of the response (thoughts, actions, etc.).

## 2. The Interface: `BaseChatModel`

Defined in `browser_use/llm/base.py`, `BaseChatModel` is a Protocol that enforces a standard interface for all LLM providers.

```python
class BaseChatModel(Protocol):
    async def ainvoke(
        self,
        messages: list[BaseMessage],
        output_format: type[T] | None = None,
        **kwargs: Any
    ) -> ChatInvokeCompletion[T] | ChatInvokeCompletion[str]: ...
```

The key method is `ainvoke`, which takes messages and an optional `output_format` for structured output generation.

## 3. Provider Implementations

Different providers implement `ainvoke` differently to achieve structured output.

### OpenAI (`browser_use/llm/openai/chat.py`)

The `ChatOpenAI` class implements `ainvoke` by leveraging OpenAI's `response_format` feature (specifically `json_schema`).

1.  **Serialization**: Converts `BaseMessage` objects to OpenAI's message format.
2.  **Schema Generation**: Uses `SchemaOptimizer` to convert the `output_format` (Pydantic model) into a JSON Schema.
3.  **API Call**:
    ```python
    response = await self.get_client().chat.completions.create(
        model=self.model,
        messages=openai_messages,
        response_format=ResponseFormatJSONSchema(json_schema=response_format, type='json_schema'),
        **model_params,
    )
    ```
4.  **Parsing**: Validates and parses the JSON response string back into the `AgentOutput` Pydantic model using `output_format.model_validate_json(...)`.

### Anthropic (`browser_use/llm/anthropic/chat.py`)

The `ChatAnthropic` class implements `ainvoke` using Anthropic's **Tool Use** capabilities, as Anthropic models are optimized to return structured data via tools.

1.  **Tool Definition**: It creates a tool definition named after the output format (e.g., `AgentOutput`) with the schema derived from the Pydantic model.
2.  **Forced Tool Use**: It forces the model to use this tool by setting `tool_choice`.
    ```python
    tool = ToolParam(
        name=tool_name,
        input_schema=schema,
        # ...
    )
    tool_choice = ToolChoiceToolParam(type='tool', name=tool_name)

    response = await self.get_client().messages.create(
        model=self.model,
        messages=anthropic_messages,
        tools=[tool],
        tool_choice=tool_choice,
        # ...
    )
    ```
3.  **Extraction**: It extracts the input arguments from the `tool_use` block in the response and validates them against the `output_format` model.

## 4. Tool Execution

Once the `Agent` receives the `AgentOutput` (containing a list of actions), it executes them.

### The Flow
1.  **`Agent.multi_act`** (`browser_use/agent/service.py`):
    Iterates through the list of actions returned by the LLM.
    ```python
    for i, action in enumerate(actions):
        # ...
        result = await self.tools.act(action, browser_session=self.browser_session, ...)
    ```

2.  **`Tools.act`** (`browser_use/tools/service.py`):
    The `Tools` service (acting as a facade) delegates the execution to the `Registry`.
    ```python
    for action_name, params in action.model_dump(exclude_unset=True).items():
        # ...
        result = await self.registry.execute_action(
            action_name=action_name,
            params=params,
            browser_session=browser_session,
            # ...
        )
    ```

3.  **`Registry.execute_action`** (`browser_use/tools/registry/service.py`):
    The registry manages the actual function calls.
    -   **Lookup**: Finds the registered action function by name (e.g., 'click', 'type').
    -   **Validation**: Validates the parameters against the Pydantic model defined for that action.
    -   **Dependency Injection**: Injects context objects like `browser_session`, `page_extraction_llm`, or `file_system` if the action function requires them.
    -   **Execution**: Calls the decorated function (e.g., `_click_by_index`).

### Action Registration
Actions are defined in `browser_use/tools/service.py` using the `@self.registry.action` decorator.

```python
@self.registry.action(
    'Click element by index...',
    param_model=ClickElementAction,
)
async def click(params: ClickElementAction, browser_session: BrowserSession):
    # Implementation...
```

The registry ensures that when the LLM requests a `click` action, this function is called with the correct parameters and the active `browser_session`.

## 5. When and What Tools Are Called

### When
The LLM calls tools iteratively during the `Agent.step` loop (in `browser_use/agent/service.py`):

1.  **Context Preparation**: `_prepare_context` captures the current browser state (DOM, screenshot).
2.  **LLM Call**: `_get_next_action` calls the LLM with this state and history.
3.  **Decision**: The LLM analyzes the screenshot/text and decides the next step(s).
4.  **Action Execution**: `_execute_actions` (via `multi_act`) runs the tools selected by the LLM.

### What
The core tools available to the LLM are defined in `browser_use/tools/service.py` and mapped to input models in `browser_use/tools/views.py`:

-   **`search`** (`SearchAction`): Google/Bing search.
-   **`navigate`** (`NavigateAction`): Go to a specific URL.
-   **`go_back`** (`NoParamsAction`): Navigate back in history.
-   **`click`** (`ClickElementAction`): Click an element by index (from DOM state) or coordinates.
-   **`input`** (`InputTextAction`): Type text into an element.
-   **`scroll`** (`ScrollAction`): Scroll the page (up/down/to element).
-   **`send_keys`** (`SendKeysAction`): Send keyboard shortcuts (e.g., Enter, Esc).
-   **`switch_tab` / `switch`** (`SwitchTabAction`): Switch browser tabs.
-   **`close_tab` / `close`** (`CloseTabAction`): Close the current tab.
-   **`extract`** (`ExtractAction`): Extract structured data from page text using a secondary LLM call.
-   **`upload_file`** (`UploadFileAction`): Upload a file to an input element.
-   **`screenshot`** (`NoParamsAction`): Explicitly take a screenshot.
-   **`dropdown_options`** (`GetDropdownOptionsAction`): List options in a select menu.
-   **`select_dropdown`** (`SelectDropdownOptionAction`): Select an option.
-   **`done`** (`DoneAction` or `StructuredOutputAction`): Signal task completion with optional data/files.

### How (Data Flow)
1.  **Schema**: The `AgentOutput` model (in `views.py`) contains a list of `ActionModel` objects.
2.  **Mapping**: The `ActionModel` is dynamically constructed (in `browser_use/tools/registry/service.py`) to include all registered actions as optional fields.
3.  **Output**: The LLM outputs JSON like:
    ```json
    {
      "thinking": "I need to click the login button.",
      "action": [
        {"click": {"index": 12}}
      ]
    }
    ```
4.  **Execution**: `Agent.multi_act` parses this JSON, identifies the key `click`, and calls the corresponding registered function.
