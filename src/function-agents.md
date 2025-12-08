# Function-Calling Agents

## Challenge

[Function-calling](https://ai.google.dev/gemini-api/docs/function-calling?example=meeting) allows you to connect models to external tools and APIs. The model can decide when and how to call specific functions to interact with real world.

In this challenge, you will build an async function-calling agent. The agent will have access to the [OpenAlex](https://docs.openalex.org/) API to search for academic works and answer research questions.

The agent will:

1. Stream model responses as they are generated
2. Execute function calls as soon as they are detected in the stream
3. Run multiple API searches concurrently when the model makes multiple function calls
4. Continue the conversation loop until the model provides a final answer

An async function-calling agent is more responsive and efficient because we make the API calls once they are streamed (not blocking on the full model response), and the API calls are non-blocking (they can run concurrently).

## Before you start

The following functions or classes are relevant for this chapter. It might be helpful to read their docs before you start:

* [`client.aio.models.generate_content_stream()`](https://googleapis.github.io/python-genai/#generate-content-asynchronous-streaming) for streaming model responses
* [Function calling API](https://ai.google.dev/gemini-api/docs/function-calling?example=meeting) for understanding how to define tools and handle function calls
* `asyncio.create_task()` for spawning concurrent search tasks
* `asyncio.gather()` for waiting on the tasks

### Step 0

To get started, get a Gemini API key from [Google AI Studio](https://aistudio.google.com/app/api-keys). We use the Gemini API because it has a generous free tier and function-calling support.

```bash
export GEMINI_API_KEY="YOUR_API_KEY"
```

Install the required packages:

```bash
pip install google-genai aiohttp
```

### Step 1

In this step, your goal is to implement the OpenAlex search function.

The search function should query the OpenAlex API and return academic works matching the search query. OpenAlex returns abstracts in an inverted index format, so you'll need to parse them.

```python
import aiohttp

async def search_openalex(
    search: str, session: aiohttp.ClientSession
) -> dict[str, Any]:
    """Searches OpenAlex for works matching the search query."""
    pass
```

You can use these helper functions:

```python
OPENALEX_API_URL = "https://api.openalex.org/works"
MAX_RESULTS_PER_PAGE = 10

async def fetch_openalex_works(
    session: aiohttp.ClientSession, params: dict[str, Any]
) -> dict[str, Any]:
    """Fetches works from the OpenAlex API based on the given parameters."""
    async with session.get(OPENALEX_API_URL, params=params) as response:
        response.raise_for_status()
        return await response.json()


def parse_abstract(abstract_inverted_index: dict[str, list[int]]) -> str:
    """Parses the abstract from the inverted index format."""
    if not abstract_inverted_index:
        return ""
    index_to_word = {}
    for word, positions in abstract_inverted_index.items():
        for pos in positions:
            index_to_word[pos] = word
    abstract_words = [index_to_word.get(i, " ") for i in range(len(index_to_word))]
    return " ".join(abstract_words)


def update_abstracts(data: dict[str, Any] | None) -> None:
    """Parses and updates abstracts in the OpenAlex response data."""
    if not data or "results" not in data:
        return
    for result in data["results"]:
        if "abstract_inverted_index" in result:
            result["abstract"] = parse_abstract(result["abstract_inverted_index"])
            del result["abstract_inverted_index"]
```

Your `search_openalex()` function should:

1. Build the query parameters including the search term, sorting, and field selection
2. Fetch the results from OpenAlex
3. Parse the abstracts using `update_abstracts()`
4. Return the data

The fields you should select are: `id`, `title`, `display_name`, `publication_year`, `publication_date`, `cited_by_count`, `primary_location`, `abstract_inverted_index`, and `authorships`.

### Step 2

In this step, your goal is to define the function specification for the model.

The model needs to know what functions are available and how to call them. You'll define this as a function declaration:

```python
search_openalex_spec = {
    "name": "search_openalex",
    "description": "Searches the OpenAlex API to retrieve academic works.",
    "parameters": {
        "type": "object",
        "properties": {
            "search": {
                "type": "string",
                "description": (
                    "Search term (1-10 words optimal). "
                    "Use AND, OR, NOT (UPPERCASE) for boolean searches. "
                    "Use quotes for exact phrases."
                ),
            },
        },
        "required": ["search"],
    },
}
```

### Step 3

In this step, your goal is to implement the main generation loop with streaming.

The key insight here is that you don't need to wait for the entire model response before executing function calls. You can start executing searches as soon as function calls are detected in the stream.

Implement the `generate_content()` function:

```python
async def generate_content(
    question: str,
    client: genai.Client,
    config: types.GenerateContentConfig,
    session: aiohttp.ClientSession,
) -> None:
    """Generate content with streaming and parallel function execution."""
    pass
```

This function should:

1. Initialize the conversation history with the user's question
2. Enter a loop that:
   * Streams the model response using `client.aio.models.generate_content_stream()`
   * Processes each chunk as it arrives
   * Collects function calls and spawns search tasks immediately
   * Prints text content as it streams
   * Waits for all search tasks to complete after the stream finishes
   * Adds function responses back to the conversation history
   * Continues the loop until no more function calls are made

```admonish tip title="Tip"
You'll need to track both the model's response parts (for conversation history) and any search tasks that are spawned during streaming. When you detect a function call in a part, create a task immediately with `asyncio.create_task()` and add it to a list.
```

### Step 4

In this step, your goal is to create the system instruction and main function.

The system instruction guides the model on how to use the search function effectively. It should encourage the model to:

* Make multiple parallel searches (up to 10 at a time)
* Iterate through multiple rounds of searches
* Only provide a final answer after at least 3 rounds

You can use this system instruction:

```python
_SYSTEM_INSTRUCTION = """
You are a research assistant that queries the OpenAlex academic database to
answer research questions.

Your task is to generate strategic search queries, analyze the abstracts,
then provide a concise cited answer.

You have access to the search_openalex function which searches academic
works in OpenAlex.

## Query Strategy

1. **Start broad** with general search terms
2. **Make up to 10 search function calls in parallel** to explore the topic
3. **Iterate with multiple rounds of searches** based on what you learn
4. **Boolean searches**: Use AND, OR, NOT (must be UPPERCASE)
5. **Exact phrases**: Use double quotes

## CRITICAL: Multi-Round Research Process

**YOU MUST COMPLETE AT LEAST 3 ROUNDS OF SEARCHES BEFORE PROVIDING YOUR FINAL ANSWER.**

Each round should:
- Make up to 10 parallel search function calls
- Analyze the abstracts returned
- Identify new keywords, related concepts, or gaps in coverage
- Use insights to formulate better queries for the next round

**DO NOT provide your final answer after just one round.**

## Important Constraints

- **You only have access to titles and abstracts** - not full papers
- Only cite information explicitly stated in abstracts
- Search terms use automatic stemming (e.g., "possums" matches "possum")
- Keep queries concise (1-10 words work best)

## Final Answer Format

**Only after completing at least 3 rounds of searches**, provide your
answer in this exact format:

**Final answer:** [1-2 paragraphs with hyperlinked citations]

### Requirements:
- Start with "Final answer:"
- **Strictly 1-2 paragraphs maximum**
- Include hyperlinked citations: `[Author et al., Year](https://openalex.org/W1234567890)`
- Use OpenAlex link format: `https://openalex.org/W1234567890` (NOT the API URL)
- Synthesize findings across papers
- Briefly acknowledge you're working from abstracts only
- No headers, bullet points, or extended formatting

### Example:

Final answer: Based on available abstracts, microplastics accumulate in marine organisms through multiple pathways [Thompson et al., 2020](https://openalex.org/W3012345678), with bioaccumulation rates varying by species. Filter feeders show particularly high concentrations [Garcia & Lee, 2021](https://openalex.org/W3123456789), and transfer through trophic levels has been documented [Chen et al., 2022](https://openalex.org/W4012345670), suggesting impacts on apex predators.

This analysis is based on article abstracts; detailed methodologies require consulting full papers. The literature indicates widespread microplastic contamination across marine environments [Martinez et al., 2023](https://openalex.org/W4123456781).

## Workflow

1. Understand the user's research question
2. **Round 1**: Generate up to 10 broad search function calls in parallel
3. Review abstracts and identify key papers, terminology, and knowledge gaps
4. **Round 2**: Generate up to 10 refined search queries based on Round 1
5. Analyze new abstracts and identify additional angles or specific subtopics
6. **Round 3+**: Continue iterating with targeted searches until comprehensive
7. **Only then** provide final answer in required format with citations
"""
```

Now implement the main function:

```python
async def main():
    client = genai.Client()
    config = types.GenerateContentConfig(
        tools=[types.Tool(function_declarations=[search_openalex_spec])],
        system_instruction=_SYSTEM_INSTRUCTION,
    )
    question = "How does the gut microbiome influence mental health?"

    async with aiohttp.ClientSession() as session:
        await generate_content(question, client, config, session)


asyncio.run(main())
```

Run your agent and verify that:

* Model responses stream as they are generated
* Multiple searches run in parallel when the model makes multiple function calls
* The conversation continues for multiple rounds
* The model provides a final cited answer

## Going Further

* Modify the system instruction to support different research workflows (e.g., finding papers by a specific author, or papers published in a specific year range).

* Add support for other function calls like fetching full paper metadata or citation graphs.

```admonish success title=""
**Now take some time to attempt the challenge before looking at the solution!**
```

---

## Solution

Below is a walkthrough of one possible solution. Your implementation may differ, and that's okay! The key concepts are streaming function calls and executing them in parallel.

First let's define all the imports and constants:

```python
import asyncio
from typing import Any

import aiohttp
from google import genai
from google.genai import types

# API Configuration
OPENALEX_API_URL = "https://api.openalex.org/works"
MAX_RESULTS_PER_PAGE = 10
```

### Step 1 - Solution

```python
async def fetch_openalex_works(
    session: aiohttp.ClientSession, params: dict[str, Any]
) -> dict[str, Any]:
    """Fetches works from the OpenAlex API based on the given parameters."""
    async with session.get(OPENALEX_API_URL, params=params) as response:
        response.raise_for_status()
        return await response.json()


def parse_abstract(abstract_inverted_index: dict[str, list[int]]) -> str:
    """Parses the abstract from the inverted index format."""
    if not abstract_inverted_index:
        return ""
    index_to_word = {}
    for word, positions in abstract_inverted_index.items():
        for pos in positions:
            index_to_word[pos] = word
    abstract_words = [index_to_word.get(i, " ") for i in range(len(index_to_word))]
    return " ".join(abstract_words)


def update_abstracts(data: dict[str, Any] | None) -> None:
    """Parses and updates abstracts in the OpenAlex response data."""
    if not data or "results" not in data:
        return
    for result in data["results"]:
        if "abstract_inverted_index" in result:
            result["abstract"] = parse_abstract(result["abstract_inverted_index"])
            del result["abstract_inverted_index"]


async def search_openalex(
    search: str, session: aiohttp.ClientSession
) -> dict[str, Any]:
    """Searches OpenAlex for works matching the search query."""
    params = {
        "search": search,
        "sort": "relevance_score:desc",
        "select": ",".join(
            [
                "id",
                "title",
                "display_name",
                "publication_year",
                "publication_date",
                "cited_by_count",
                "primary_location",
                "abstract_inverted_index",
                "authorships",
            ]
        ),
        "per_page": MAX_RESULTS_PER_PAGE,
        "mailto": "your@email.com",
    }
    data = await fetch_openalex_works(session, params)
    update_abstracts(data)
    return data
```

### Step 2 - Solution

```python
search_openalex_spec = {
    "name": "search_openalex",
    "description": "Searches the OpenAlex API to retrieve academic works.",
    "parameters": {
        "type": "object",
        "properties": {
            "search": {
                "type": "string",
                "description": (
                    "Search term (1-10 words optimal). "
                    "Use AND, OR, NOT (UPPERCASE) for boolean searches. "
                    "Use quotes for exact phrases."
                ),
            },
        },
        "required": ["search"],
    },
}
```

### Step 3 - Solution

The key to this solution is executing function calls as they stream in, rather than waiting for the complete response.

```python
async def generate_content(
    question: str,
    client: genai.Client,
    config: types.GenerateContentConfig,
    session: aiohttp.ClientSession,
) -> None:
    """Generate content with streaming and parallel function execution."""
    # Initialize conversation history
    contents = [question]

    while True:
        model_parts = []
        search_tasks = []

        print("\n--- Streaming response ---")
        async for chunk in await client.aio.models.generate_content_stream(
            model="gemini-flash-latest",
            contents=contents,
            config=config,
        ):
            if not chunk.candidates:
                continue

            candidate = chunk.candidates[0]
            if not candidate.content or not candidate.content.parts:
                continue

            for part in candidate.content.parts:
                # Collect all parts for history
                model_parts.append(part)

                # Print any text content as it arrives
                if part.text:
                    print(part.text, end="", flush=True)

                # Execute function calls immediately as they are streamed
                if part.function_call:
                    print(f"\n[Function call detected: {part.function_call.name}]")
                    print(f"[Arguments: {part.function_call.args}]")

                    # Start the search task immediately mid-stream
                    task = asyncio.create_task(
                        search_openalex(**part.function_call.args, session=session)
                    )
                    search_tasks.append(task)

        # Add model's response to conversation history
        if model_parts:
            contents.append(types.Content(role="model", parts=model_parts))

        if not search_tasks:
            # No more function calls to execute
            print("\n--- Conversation complete ---")
            return

        print(f"\n\n[Started {len(search_tasks)} search tasks]")

        search_results = await asyncio.gather(*search_tasks)

        print("\n\n[Search tasks complete]")

        # Prepare function responses and add to conversation history
        function_response_parts = [
            types.Part.from_function_response(
                name="search_openalex", response={"output": result}
            )
            for result in search_results
        ]
        contents.append(types.Content(role="user", parts=function_response_parts))
```

The conversation loop continues until the model stops making function calls and provides its final answer.

Note how:

* Function calls are executed immediately as they appear in the stream using `asyncio.create_task()`.
* Multiple function calls in the same response run in parallel.
* Text content is printed as it streams for real-time feedback.
* The conversation history tracks all turns for context.

### Step 4 - Solution

```python
_SYSTEM_INSTRUCTION = """
You are a research assistant that queries the OpenAlex academic database to
answer research questions.

Your task is to generate strategic search queries, analyze the abstracts,
then provide a concise cited answer.

You have access to the search_openalex function which searches academic
works in OpenAlex.

## Query Strategy

1. **Start broad** with general search terms
2. **Make up to 10 search function calls in parallel** to explore the topic
3. **Iterate with multiple rounds of searches** based on what you learn
4. **Boolean searches**: Use AND, OR, NOT (must be UPPERCASE)
5. **Exact phrases**: Use double quotes

## CRITICAL: Multi-Round Research Process

**YOU MUST COMPLETE AT LEAST 3 ROUNDS OF SEARCHES BEFORE PROVIDING YOUR FINAL ANSWER.**

Each round should:
- Make up to 10 parallel search function calls
- Analyze the abstracts returned
- Identify new keywords, related concepts, or gaps in coverage
- Use insights to formulate better queries for the next round

**DO NOT provide your final answer after just one round.**

## Important Constraints

- **You only have access to titles and abstracts** - not full papers
- Only cite information explicitly stated in abstracts
- Search terms use automatic stemming (e.g., "possums" matches "possum")
- Keep queries concise (1-10 words work best)

## Final Answer Format

**Only after completing at least 3 rounds of searches**, provide your
answer in this exact format:

**Final answer:** [1-2 paragraphs with hyperlinked citations]

### Requirements:
- Start with "Final answer:"
- **Strictly 1-2 paragraphs maximum**
- Include hyperlinked citations: `[Author et al., Year](https://openalex.org/W1234567890)`
- Use OpenAlex link format: `https://openalex.org/W1234567890` (NOT the API URL)
- Synthesize findings across papers
- Briefly acknowledge you're working from abstracts only
- No headers, bullet points, or extended formatting

### Example:

Final answer: Based on available abstracts, microplastics accumulate in marine organisms through multiple pathways [Thompson et al., 2020](https://openalex.org/W3012345678), with bioaccumulation rates varying by species. Filter feeders show particularly high concentrations [Garcia & Lee, 2021](https://openalex.org/W3123456789), and transfer through trophic levels has been documented [Chen et al., 2022](https://openalex.org/W4012345670), suggesting impacts on apex predators.

This analysis is based on article abstracts; detailed methodologies require consulting full papers. The literature indicates widespread microplastic contamination across marine environments [Martinez et al., 2023](https://openalex.org/W4123456781).


## Workflow

1. Understand the user's research question
2. **Round 1**: Generate up to 10 broad search function calls in parallel
3. Review abstracts and identify key papers, terminology, and knowledge gaps
4. **Round 2**: Generate up to 10 refined search queries based on Round 1
5. Analyze new abstracts and identify additional angles or specific subtopics
6. **Round 3+**: Continue iterating with targeted searches until comprehensive
7. **Only then** provide final answer in required format with citations
"""


async def main():
    client = genai.Client()
    config = types.GenerateContentConfig(
        tools=[types.Tool(function_declarations=[search_openalex_spec])],
        system_instruction=_SYSTEM_INSTRUCTION,
    )
    question = "How does the gut microbiome influence mental health?"

    async with aiohttp.ClientSession() as session:
        await generate_content(question, client, config, session)


asyncio.run(main())
```

Now let's run this and observe the output:

```bash
python script.py

--- Streaming response ---

[Function call detected: search_openalex]
[Arguments: {'search': 'gut microbiome mental health'}]

[Function call detected: search_openalex]
[Arguments: {'search': 'microbiota gut brain axis'}]

[Function call detected: search_openalex]
[Arguments: {'search': 'probiotics anxiety depression'}]

[Function call detected: search_openalex]
[Arguments: {'search': 'dysbiosis neurological disorders'}]

[Function call detected: search_openalex]
[Arguments: {'search': 'gut bacteria neurotransmitters'}]

[Function call detected: search_openalex]
[Arguments: {'search': 'microbiome stress response'}]

[Function call detected: search_openalex]
[Arguments: {'search': 'psychobiotics'}]

[Function call detected: search_openalex]
[Arguments: {'search': 'microbiome autism'}]

[Function call detected: search_openalex]
[Arguments: {'search': 'gut brain communication'}]

[Function call detected: search_openalex]
[Arguments: {'search': 'microbiome influence mood'}]


[Started 10 search tasks]


[Search tasks complete]

--- Streaming response ---

[Function call detected: search_openalex]
[Arguments: {'search': 'microbiome Tryptophan kynurenine serotonin'}]

[Function call detected: search_openalex]
[Arguments: {'search': 'Short-chain fatty acids SCFA influence brain function'}]

[Function call detected: search_openalex]
[Arguments: {'search': 'Butyrate brain barrier'}]


[Started 3 search tasks]


[Search tasks complete]

--- Streaming response ---
The gut microbiome influences mental health through the **Microbiota-Gut-Brain (MGB) axis**, a complex, bidirectional communication network linking the gastrointestinal tract and the central nervous system through endocrine, immune, and neural signaling pathways [Cryan et al., 2019](https://openalex.org/W2970686316). This relationship means that alterations in the composition of the gut microbiota (dysbiosis) are associated with a range of neuropsychiatric and neurological disorders, including anxiety, depression, and autism spectrum disorder (ASD) [Socała et al., 2021](https://openalex.org/W3193421084).

The primary mechanisms by which gut microbes modulate brain function involve the production of chemical signals. First, microbial fermentation of dietary fiber produces **short-chain fatty acids (SCFAs)**, such as butyrate, propionate, and acetate. These metabolites are crucial for maintaining the integrity of the gut barrier and the blood-brain barrier (BBB), and they function as signaling molecules that regulate neuro-immunoendocrine pathways, potentially alleviating stress-induced brain alterations [Silva et al., 2020](https://openalex.org/W3003912482); [van de Wouw et al., 2018](https://openalex.org/W2891851874). Second, gut bacteria regulate the metabolism of the amino acid tryptophan, a precursor to the neurotransmitter **serotonin (5-HT)**, which plays a critical role in mood, appetite, and sleep [O’Mahony et al., 2014](https://openalex.org/W1972237932); [Roth et al., 2021](https://openalex.org/W3137164735). Neural communication occurs directly via the **Vagus Nerve**, which transmits signals from the gut to the brain; certain beneficial bacteria strains have demonstrated anxiolytic-like effects that require the integrity of this nerve to modulate central neural pathways [Bravo et al., 2011](https://openalex.org/W2164342861). Furthermore, the microbiota helps regulate the **Hypothalamic-Pituitary-Adrenal (HPA) axis**, the body’s main stress response system, and dysbiosis can exacerbate the inflammatory processes associated with depression and increased stress reactivity [Cryan et al., 2019](https://openalex.org/W2970686316); [Kiecolt-Glaser et al., 2015](https://openalex.org/W2159036260). This research has led to the development of "psychobiotics" (probiotics or prebiotics) aimed at manipulating the MGB axis to improve symptoms of conditions like depression and anxiety [Dinan et al., 2013](https://openalex.org/W1983814959).

***
*This answer is based solely on the analysis of available academic abstracts.*
--- Conversation complete ---
```

Note how:

* Multiple searches run concurrently within each round.
* Searches are triggered as the responses are streamed. This is faster than sequentially waiting on responses and making API calls.
* The model iterates through multiple rounds before providing the final answer.
