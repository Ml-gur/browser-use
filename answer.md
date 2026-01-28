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

## Summary

The `Agent` treats the LLM as a black box that accepts messages and returns a structured `AgentOutput` object. The specific LLM implementation (`ChatOpenAI`, `ChatAnthropic`, etc.) handles the nuances of the provider's API (JSON mode vs. Tool calling) to ensure the output matches the required schema.
