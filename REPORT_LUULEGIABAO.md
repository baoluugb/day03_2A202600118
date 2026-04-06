# Individual Report: Lab 3 - Chatbot vs ReAct Agent

- **Student Name**: Luu Le Gia Bao
- **Student ID**: 2A202600118
- **Date**: April 6, 2026

---

## I. Technical Contribution (15 Points)

_Describe your specific contribution to the codebase (e.g., implemented a specific tool, fixed the parser, etc.)._

### Modules Implemented

1. **`src/tools/weather_tool.py`** - Core weather tool implementation
   - `WeatherTool` class that accepts any `LLMProvider` implementation
   - Prompt generation for weather forecasts using a location and date input
   - Regex-based parsing of the LLM response into structured fields
   - Metadata attachment for `model`, `latency_ms`, and `raw_response`

2. **`examples_weather_tool.py`** - Integration example
   - Demonstrates how to initialize the tool with `LocalProvider`
   - Shows how to call `to_tool_dict()` for ReAct agent integration
   - Includes sample executions for specific dates and natural-language dates

### Code Highlights

**WeatherTool.execute() method** - Core functionality:

```python
def execute(self, location: str, date: str) -> Dict[str, Any]:
  """Get weather information for a location on a specific date."""
  prompt = f"""Provide a weather forecast for {location} on {date}.
  Format your response with these exact labels on separate lines:
  Temperature: [value in Celsius and Fahrenheit]
  Condition: [sunny/cloudy/rainy/snowy/etc]
  Humidity: [percentage value]
  Wind Speed: [wind speed in km/h or mph]"""

  response = self.llm.generate(prompt, system_prompt)
  parsed_weather = self._parse_weather_response(content, location, date)
  return parsed_weather
```

**Response Parsing** - Regex-based extraction:

```python
def _parse_weather_response(self, response: str, location: str, date: str) -> Dict[str, Any]:
  weather_dict = {
    "location": location,
    "date": date,
    "temperature": "Not available",
    "condition": "Not available",
    "humidity": "Not available",
    "wind_speed": "Not available"
  }

  # Extract structured data from the LLM response using regex patterns.
  temp_match = re.search(r'Temperature:\s*(.+?)(?:\n|$)', response, re.IGNORECASE)
  if temp_match:
    weather_dict["temperature"] = temp_match.group(1).strip()

  # Similar parsing is applied for condition, humidity, and wind speed.
```

**Tool Definition** - Agent-facing metadata:

```python
def to_tool_dict(self) -> Dict[str, Any]:
  return {
    "name": self.name,
    "description": self.description,
    "parameters": {
      "location": "The location to get weather for",
      "date": "The date (YYYY-MM-DD format or descriptive like 'tomorrow')"
    }
  }
```

### ReAct Flow Support

The weather tool is designed to support a ReAct agent through the tool dictionary returned by `to_tool_dict()` and the structured output returned by `execute()`.

1. **System Prompt**: Tool listed with description
   - The agent can see: `get_weather: Get the weather forecast for a specific location on a given date`

2. **Agent Reasoning**: LLM generates Thought and Action
   - Example: `Action: get_weather("San Francisco", "2024-01-15")`

3. **Tool Execution**: The example integration calls `weather_tool.execute("San Francisco", "2024-01-15")`

4. **Observation**: Formatted weather data returned to agent
   - Result includes `location`, `date`, `temperature`, `condition`, `humidity`, `wind_speed`, `model`, `latency_ms`, and `raw_response`

5. **Next Step**: Agent uses observation for reasoning and Final Answer
   - The agent can summarize the parsed weather data into a user-facing response

---

## II. Debugging Case Study (10 Points)

_Analyze a specific failure event you encountered during the lab using the logging system._

### Problem Description

**Issue**: The LLM sometimes returned partial or unstructured weather text, which made parsing incomplete.

**Failure Sequence**:

```text
LLM output: "It will likely be warm and sunny. Temperature: 22°C (72°F)"
Expected: All labels present on separate lines
Result: Only the temperature field was parsed; other fields stayed at "Not available"
```

The issue came from depending on model compliance with the required response format.

### Root Cause Analysis

**Diagnosis**: The parser relies on exact labels, so missing labels or merged sentences reduce the quality of the structured output.

**Original prompt requirement**:

```text
Temperature: [value in Celsius and Fahrenheit]
Condition: [sunny/cloudy/rainy/snowy/etc]
Humidity: [percentage value]
Wind Speed: [wind speed in km/h or mph]
```

This works well when the model follows instructions, but it can fail when the response is informal or incomplete.

### Solution Implementation

**Improved prompt constraints**:

```python
"Provide your response with these exact labels on separate lines:\n"
"Temperature: ...\nCondition: ...\nHumidity: ...\nWind Speed: ..."
```

**Fallback parsing behavior**:

```python
weather_dict = {
  "temperature": "Not available",
  "condition": "Not available",
  "humidity": "Not available",
  "wind_speed": "Not available"
}
```

**Verification in tests**:

```text
Complete responses, partial responses, and invalid responses are all covered by unit tests.
```

---

## III. Agent Reasoning with Weather Tool (10 Points)

_Analyze how the weather tool enhances the ReAct agent's decision-making capability._

### Tool Integration in ReAct Loop

The weather tool improves the ReAct loop by turning a natural-language forecast request into structured data the agent can reason over.

**Step 1: Problem Recognition**

- Agent analyzes user query: "What's the weather in San Francisco tomorrow?"
- Identifies need for external weather data
- Determines relevant tool: `get_weather`

**Step 2: Action Formulation**

```text
Thought: The user is asking for weather information. I need to use the get_weather tool.

Action: get_weather("San Francisco", "tomorrow")
```

**Step 3: Tool Execution & Observation**

- Weather tool constructs a prompt with the requested location and date
- LLM generates weather forecast with structured labels:

```text
Temperature: 22°C (72°F)
Condition: sunny
Humidity: 65%
Wind Speed: 10 km/h
```

- Tool parses the response and returns structured observation

**Step 4: Reasoning with Observation**

```text
Observation:
Location: San Francisco
Date: tomorrow
Temperature: 22°C (72°F)
Condition: sunny
Humidity: 65%
Wind Speed: 10 km/h

Thought: I now have the weather data. The conditions show sunny weather with moderate temperature.

Final Answer: The weather in San Francisco tomorrow will be sunny with a temperature of 22°C (72°F).
```

### Key Design Decisions

**Argument Format**: The tool accepts location and date as separate arguments

- The code expects separate `location` and `date` parameters
- Flexible date strings are supported, such as `2024-01-15`, `tomorrow`, and `next Monday`
- Example call: `weather_tool.execute("Paris", "2024-01-20")`

**Structured Response Parsing**: Regex-based extraction ensures reliability

- Extracts temperature, condition, humidity, wind speed
- Handles missing fields gracefully with "Not available" defaults
- Independent of LLM response format variations

**Tool Return Format**: Optimized for agent interpretation

- Dictionary with location, date, and all weather metrics
- Includes model and latency metadata for debugging
- Keeps the raw LLM response available for inspection

---

## IV. Future Improvements (5 Points)

_How would you scale this for a production-level AI agent system?_

### Scalability Improvements

**Async Tool Execution**:

- Current: Tool calls block the agent loop
- Future: `asyncio` queue for parallel tool execution

```python
# Execute multiple tools concurrently
tasks = [get_weather("SF"), get_weather("LA")]
results = await asyncio.gather(*tasks)
```

- Benefit: Supports multi-location queries or batch weather checks

**Tool Registry Pattern**:

- Current: Tool wiring is done manually by the caller
- Future: Dynamic tool discovery from plugin system

```python
tools = load_tools_from_registry(
  categories=["weather", "hotel", "flight"]
)
```

- Benefit: Add or remove tools without modifying agent code

**Caching Layer**:

- Current: Every query re-fetches weather data
- Future: Cache weather results with TTL

```python
@cache(ttl=3600)  # Cache for 1 hour
def execute(self, location: str, date: str):
  return self.llm.generate(prompt)
```

- Benefit: Reduce LLM API calls and latency for repeated queries

### Safety Improvements

**Supervisor LLM Pattern**:

- Implement a second LLM to audit agent actions

```python
supervisor = LLM("gpt-4")
if supervisor.is_action_safe(agent_action):
  execute_action()
else:
  request_user_confirmation()
```

- Use cases:
  - Prevent tool abuse (excessive API calls)
  - Catch hallucinated tool names before execution
  - Validate agent reasoning matches user intent

**Input Validation & Sanitization**:

- Current: The tool assumes valid `location` and `date` strings are passed in
- Future: Strict schema validation

```python
schema = {
  "location": {"type": "string", "max_length": 100},
  "date": {"type": "string", "pattern": r"^\d{4}-\d{2}-\d{2}$"}
}
validate(args, schema)
```

- Benefit: Prevent injection attacks and limit resource usage

**Rate Limiting & Quotas**:

- Track tool usage per user/session
- Prevent DoS attacks through excessive tool calls

```python
if user_tool_calls_today > rate_limit:
  return "Rate limit exceeded. Try again tomorrow."
```

### Performance Improvements

**Vector Database for Tool Retrieval**:

- Current: Agent searches all tools every time
- Future: Semantic search for relevant tools

```python
# Embed user query
query_embedding = embed(user_input)
# Find most relevant tools using vector similarity
relevant_tools = vector_db.search(query_embedding, top_k=3)
```

- Benefit: In systems with 100+ tools, the agent selects only relevant ones

**LLM Response Streaming**:

- Current: Agent waits for full LLM response
- Future: Stream tokens and start processing early

```python
for token in llm.stream(prompt):
  process_token_for_action_extraction(token)
  if action_complete:
    execute_early()
    break
```

- Benefit: Reduces perceived latency

**Structured Output Format**:

- Current: Weather output is parsed with regex
- Future: Force structured JSON from LLM

```python
response = llm.generate(
  prompt,
  response_format={"type": "json_schema", "schema": REACT_ACTION_SCHEMA}
)
```

- Benefit: Eliminates parsing ambiguity and improves extraction reliability

**Monitoring & Observability**:

- Log all agent decisions with confidence scores
- Track success rates for each tool
- Alert on unusual patterns such as loops or excessive retries

```python
metrics.record("tool_call", tool_name, success=True, latency_ms=150)
```
