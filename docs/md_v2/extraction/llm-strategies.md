# Extracting JSON (LLM)

In some cases, you need to extract **complex or unstructured** information from a webpage that a simple CSS/XPath schema cannot easily parse. Or you want **AI**-driven insights, classification, or summarization. For these scenarios, Crawl4AI provides an **LLM-based extraction strategy** that:

1. Works with **any** large language model supported by [LiteLLM](https://github.com/BerriAI/litellm) (Ollama, OpenAI, Claude, and more).  
2. Automatically splits content into chunks (if desired) to handle token limits, then combines results.  
3. Lets you define a **schema** (like a Pydantic model) or a simpler “block” extraction approach.

**Important**: LLM-based extraction can be slower and costlier than schema-based approaches. If your page data is highly structured, consider using [`JsonCssExtractionStrategy`](./no-llm-strategies.md) or [`JsonXPathExtractionStrategy`](./no-llm-strategies.md) first. But if you need AI to interpret or reorganize content, read on!

---

## 1. Why Use an LLM?

- **Complex Reasoning**: If the site’s data is unstructured, scattered, or full of natural language context.  
- **Semantic Extraction**: Summaries, knowledge graphs, or relational data that require comprehension.  
- **Flexible**: You can pass instructions to the model to do more advanced transformations or classification.

---

## 2. Provider-Agnostic via LiteLLM

You can use LlmConfig, to quickly configure multiple variations of LLMs and experiment with them to find the optimal one for your use case. You can read more about LlmConfig [here](/api/parameters).

```python
llmConfig = LlmConfig(provider="openai/gpt-4o-mini", api_token=os.getenv("OPENAI_API_KEY"))
```

Crawl4AI uses a “provider string” (e.g., `"openai/gpt-4o"`, `"ollama/llama2.0"`, `"aws/titan"`) to identify your LLM. **Any** model that LiteLLM supports is fair game. You just provide:

- **`provider`**: The `<provider>/<model_name>` identifier (e.g., `"openai/gpt-4"`, `"ollama/llama2"`, `"huggingface/google-flan"`, etc.).  
- **`api_token`**: If needed (for OpenAI, HuggingFace, etc.); local models or Ollama might not require it.  
- **`base_url`** (optional): If your provider has a custom endpoint.  

This means you **aren’t locked** into a single LLM vendor. Switch or experiment easily.

---

## 3. How LLM Extraction Works

### 3.1 Flow

1. **Chunking** (optional): The HTML or markdown is split into smaller segments if it’s very long (based on `chunk_token_threshold`, overlap, etc.).  
2. **Prompt Construction**: For each chunk, the library forms a prompt that includes your **`instruction`** (and possibly schema or examples).  
3. **LLM Inference**: Each chunk is sent to the model in parallel or sequentially (depending on your concurrency).  
4. **Combining**: The results from each chunk are merged and parsed into JSON.

### 3.2 `extraction_type`

- **`"schema"`**: The model tries to return JSON conforming to your Pydantic-based schema.  
- **`"block"`**: The model returns freeform text, or smaller JSON structures, which the library collects.  

For structured data, `"schema"` is recommended. You provide `schema=YourPydanticModel.model_json_schema()`.

---

## 4. Key Parameters

Below is an overview of important LLM extraction parameters. All are typically set inside `LLMExtractionStrategy(...)`. You then put that strategy in your `CrawlerRunConfig(..., extraction_strategy=...)`.

1. **`llmConfig`** (LlmConfig): e.g., `"openai/gpt-4"`, `"ollama/llama2"`.    
2. **`schema`** (dict): A JSON schema describing the fields you want. Usually generated by `YourModel.model_json_schema()`.  
3. **`extraction_type`** (str): `"schema"` or `"block"`.  
4. **`instruction`** (str): Prompt text telling the LLM what you want extracted. E.g., “Extract these fields as a JSON array.”  
5. **`chunk_token_threshold`** (int): Maximum tokens per chunk. If your content is huge, you can break it up for the LLM.  
6. **`overlap_rate`** (float): Overlap ratio between adjacent chunks. E.g., `0.1` means 10% of each chunk is repeated to preserve context continuity.  
7. **`apply_chunking`** (bool): Set `True` to chunk automatically. If you want a single pass, set `False`.  
8. **`input_format`** (str): Determines **which** crawler result is passed to the LLM. Options include:  
   - `"markdown"`: The raw markdown (default).  
   - `"fit_markdown"`: The filtered “fit” markdown if you used a content filter.  
   - `"html"`: The cleaned or raw HTML.  
9. **`extra_args`** (dict): Additional LLM parameters like `temperature`, `max_tokens`, `top_p`, etc.  
10. **`show_usage()`**: A method you can call to print out usage info (token usage per chunk, total cost if known).  

**Example**:

```python
extraction_strategy = LLMExtractionStrategy(
    llm_config = LLMConfig(provider="openai/gpt-4", api_token="YOUR_OPENAI_KEY"),
    schema=MyModel.model_json_schema(),
    extraction_type="schema",
    instruction="Extract a list of items from the text with 'name' and 'price' fields.",
    chunk_token_threshold=1200,
    overlap_rate=0.1,
    apply_chunking=True,
    input_format="html",
    extra_args={"temperature": 0.1, "max_tokens": 1000},
    verbose=True
)
```

---

## 5. Putting It in `CrawlerRunConfig`

**Important**: In Crawl4AI, all strategy definitions should go inside the `CrawlerRunConfig`, not directly as a param in `arun()`. Here’s a full example:

```python
import os
import asyncio
import json
from pydantic import BaseModel, Field
from typing import List
from crawl4ai import AsyncWebCrawler, BrowserConfig, CrawlerRunConfig, CacheMode, LLMConfig
from crawl4ai import LLMExtractionStrategy

class Product(BaseModel):
    name: str
    price: str

async def main():
    # 1. Define the LLM extraction strategy
    llm_strategy = LLMExtractionStrategy(
        llm_config = LLMConfig(provider="openai/gpt-4o-mini", api_token=os.getenv('OPENAI_API_KEY')),
        schema=Product.schema_json(), # Or use model_json_schema()
        extraction_type="schema",
        instruction="Extract all product objects with 'name' and 'price' from the content.",
        chunk_token_threshold=1000,
        overlap_rate=0.0,
        apply_chunking=True,
        input_format="markdown",   # or "html", "fit_markdown"
        extra_args={"temperature": 0.0, "max_tokens": 800}
    )

    # 2. Build the crawler config
    crawl_config = CrawlerRunConfig(
        extraction_strategy=llm_strategy,
        cache_mode=CacheMode.BYPASS
    )

    # 3. Create a browser config if needed
    browser_cfg = BrowserConfig(headless=True)

    async with AsyncWebCrawler(config=browser_cfg) as crawler:
        # 4. Let's say we want to crawl a single page
        result = await crawler.arun(
            url="https://example.com/products",
            config=crawl_config
        )

        if result.success:
            # 5. The extracted content is presumably JSON
            data = json.loads(result.extracted_content)
            print("Extracted items:", data)
            
            # 6. Show usage stats
            llm_strategy.show_usage()  # prints token usage
        else:
            print("Error:", result.error_message)

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 6. Chunking Details

### 6.1 `chunk_token_threshold`

If your page is large, you might exceed your LLM’s context window. **`chunk_token_threshold`** sets the approximate max tokens per chunk. The library calculates word→token ratio using `word_token_rate` (often ~0.75 by default). If chunking is enabled (`apply_chunking=True`), the text is split into segments.

### 6.2 `overlap_rate`

To keep context continuous across chunks, we can overlap them. E.g., `overlap_rate=0.1` means each subsequent chunk includes 10% of the previous chunk’s text. This is helpful if your needed info might straddle chunk boundaries.

### 6.3 Performance & Parallelism

By chunking, you can potentially process multiple chunks in parallel (depending on your concurrency settings and the LLM provider). This reduces total time if the site is huge or has many sections.

---

## 7. Input Format

By default, **LLMExtractionStrategy** uses `input_format="markdown"`, meaning the **crawler’s final markdown** is fed to the LLM. You can change to:

- **`html`**: The cleaned HTML or raw HTML (depending on your crawler config) goes into the LLM.  
- **`fit_markdown`**: If you used, for instance, `PruningContentFilter`, the “fit” version of the markdown is used. This can drastically reduce tokens if you trust the filter.  
- **`markdown`**: Standard markdown output from the crawler’s `markdown_generator`.

This setting is crucial: if the LLM instructions rely on HTML tags, pick `"html"`. If you prefer a text-based approach, pick `"markdown"`.

```python
LLMExtractionStrategy(
    # ...
    input_format="html",  # Instead of "markdown" or "fit_markdown"
)
```

---

## 8. Token Usage & Show Usage

To keep track of tokens and cost, each chunk is processed with an LLM call. We record usage in:

- **`usages`** (list): token usage per chunk or call.  
- **`total_usage`**: sum of all chunk calls.  
- **`show_usage()`**: prints a usage report (if the provider returns usage data).

```python
llm_strategy = LLMExtractionStrategy(...)
# ...
llm_strategy.show_usage()
# e.g. “Total usage: 1241 tokens across 2 chunk calls”
```

If your model provider doesn’t return usage info, these fields might be partial or empty.

---

## 9. Example: Building a Knowledge Graph

Below is a snippet combining **`LLMExtractionStrategy`** with a Pydantic schema for a knowledge graph. Notice how we pass an **`instruction`** telling the model what to parse.

```python
import os
import json
import asyncio
from typing import List
from pydantic import BaseModel, Field
from crawl4ai import AsyncWebCrawler, BrowserConfig, CrawlerRunConfig, CacheMode, LLMConfig
from crawl4ai import LLMExtractionStrategy

class Entity(BaseModel):
    name: str
    description: str

class Relationship(BaseModel):
    entity1: Entity
    entity2: Entity
    description: str
    relation_type: str

class KnowledgeGraph(BaseModel):
    entities: List[Entity]
    relationships: List[Relationship]

async def main():
    # LLM extraction strategy
    llm_strat = LLMExtractionStrategy(
        llmConfig = LLMConfig(provider="openai/gpt-4", api_token=os.getenv('OPENAI_API_KEY')),
        schema=KnowledgeGraph.model_json_schema(),
        extraction_type="schema",
        instruction="Extract entities and relationships from the content. Return valid JSON.",
        chunk_token_threshold=1400,
        apply_chunking=True,
        input_format="html",
        extra_args={"temperature": 0.1, "max_tokens": 1500}
    )

    crawl_config = CrawlerRunConfig(
        extraction_strategy=llm_strat,
        cache_mode=CacheMode.BYPASS
    )

    async with AsyncWebCrawler(config=BrowserConfig(headless=True)) as crawler:
        # Example page
        url = "https://www.nbcnews.com/business"
        result = await crawler.arun(url=url, config=crawl_config)

        print("--- LLM RAW RESPONSE ---")
        print(result.extracted_content)
        print("--- END LLM RAW RESPONSE ---")

        if result.success:
            with open("kb_result.json", "w", encoding="utf-8") as f:
                f.write(result.extracted_content)
            llm_strat.show_usage()
        else:
            print("Crawl failed:", result.error_message)

if __name__ == "__main__":
    asyncio.run(main())
```

**Key Observations**:

- **`extraction_type="schema"`** ensures we get JSON fitting our `KnowledgeGraph`.  
- **`input_format="html"`** means we feed HTML to the model.  
- **`instruction`** guides the model to output a structured knowledge graph.  

---

## 10. Best Practices & Caveats

1. **Cost & Latency**: LLM calls can be slow or expensive. Consider chunking or smaller coverage if you only need partial data.  
2. **Model Token Limits**: If your page + instruction exceed the context window, chunking is essential.  
3. **Instruction Engineering**: Well-crafted instructions can drastically improve output reliability.  
4. **Schema Strictness**: `"schema"` extraction tries to parse the model output as JSON. If the model returns invalid JSON, partial extraction might happen, or you might get an error.  
5. **Parallel vs. Serial**: The library can process multiple chunks in parallel, but you must watch out for rate limits on certain providers.  
6. **Check Output**: Sometimes, an LLM might omit fields or produce extraneous text. You may want to post-validate with Pydantic or do additional cleanup.

---

## 11. Conclusion

**LLM-based extraction** in Crawl4AI is **provider-agnostic**, letting you choose from hundreds of models via LiteLLM. It’s perfect for **semantically complex** tasks or generating advanced structures like knowledge graphs. However, it’s **slower** and potentially costlier than schema-based approaches. Keep these tips in mind:

- Put your LLM strategy **in `CrawlerRunConfig`**.  
- Use **`input_format`** to pick which form (markdown, HTML, fit_markdown) the LLM sees.  
- Tweak **`chunk_token_threshold`**, **`overlap_rate`**, and **`apply_chunking`** to handle large content efficiently.  
- Monitor token usage with `show_usage()`.

If your site’s data is consistent or repetitive, consider [`JsonCssExtractionStrategy`](./no-llm-strategies.md) first for speed and simplicity. But if you need an **AI-driven** approach, `LLMExtractionStrategy` offers a flexible, multi-provider solution for extracting structured JSON from any website.

**Next Steps**:

1. **Experiment with Different Providers**  
   - Try switching the `provider` (e.g., `"ollama/llama2"`, `"openai/gpt-4o"`, etc.) to see differences in speed, accuracy, or cost.  
   - Pass different `extra_args` like `temperature`, `top_p`, and `max_tokens` to fine-tune your results.

2. **Performance Tuning**  
   - If pages are large, tweak `chunk_token_threshold`, `overlap_rate`, or `apply_chunking` to optimize throughput.  
   - Check the usage logs with `show_usage()` to keep an eye on token consumption and identify potential bottlenecks.

3. **Validate Outputs**  
   - If using `extraction_type="schema"`, parse the LLM’s JSON with a Pydantic model for a final validation step.  
   - Log or handle any parse errors gracefully, especially if the model occasionally returns malformed JSON.

4. **Explore Hooks & Automation**  
   - Integrate LLM extraction with [hooks](../advanced/hooks-auth.md) for complex pre/post-processing.  
   - Use a multi-step pipeline: crawl, filter, LLM-extract, then store or index results for further analysis.

**Last Updated**: 2025-01-01

---

That’s it for **Extracting JSON (LLM)**—now you can harness AI to parse, classify, or reorganize data on the web. Happy crawling!
