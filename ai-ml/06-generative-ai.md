# Chapter 6: Generative AI on GCP

This chapter is especially relevant given your RAG and LLM experience at Fortinet. It will help you translate your knowledge to GCP's offerings.

## 6.1 Generative AI Landscape on GCP

```
┌─────────────────────────────────────────────────────────────────┐
│              Generative AI on Google Cloud                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                Foundation Models                          │   │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────────────┐   │   │
│  │  │   Gemini   │ │   PaLM 2   │ │    Imagen/Codey   │   │   │
│  │  │  (latest)  │ │  (legacy)  │ │ (Image/Code gen)  │   │   │
│  │  └────────────┘ └────────────┘ └────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Tools & Services                       │   │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────────────┐   │   │
│  │  │  Vertex AI │ │ Agent      │ │   Vertex AI        │   │   │
│  │  │  Studio    │ │ Builder    │ │   Search           │   │   │
│  │  └────────────┘ └────────────┘ └────────────────────┘   │   │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────────────┐   │   │
│  │  │ Embeddings │ │  Grounding │ │   Model Garden     │   │   │
│  │  │    API     │ │            │ │   (OSS models)     │   │   │
│  │  └────────────┘ └────────────┘ └────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                Enterprise Features                        │   │
│  │  • Data governance • VPC Service Controls                │   │
│  │  • Custom model tuning • Responsible AI                  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.2 Gemini Models

Google's most capable AI model family.

### Model Variants

| Model | Context Window | Use Case |
|-------|----------------|----------|
| **Gemini 1.5 Pro** | 1M tokens | Complex reasoning, long documents |
| **Gemini 1.5 Flash** | 1M tokens | Fast, cost-effective |
| **Gemini 1.0 Pro** | 32K tokens | General tasks |
| **Gemini 1.0 Pro Vision** | 16K tokens | Multimodal (text + images) |

### Using Gemini API

```python
import vertexai
from vertexai.generative_models import GenerativeModel, Part

vertexai.init(project='my-project', location='us-central1')

# Text generation
model = GenerativeModel('gemini-1.5-pro')
response = model.generate_content("Explain quantum computing in simple terms.")
print(response.text)

# Chat conversation
chat = model.start_chat()
response = chat.send_message("Hello, I need help with GCP.")
print(response.text)
response = chat.send_message("How do I set up a VPC?")
print(response.text)

# Multimodal (text + image)
image = Part.from_uri(
    "gs://bucket/image.jpg",
    mime_type="image/jpeg"
)
response = model.generate_content([
    "Describe what you see in this image:",
    image
])
print(response.text)
```

### Generation Configuration

```python
from vertexai.generative_models import GenerationConfig

config = GenerationConfig(
    temperature=0.7,        # Creativity (0.0-1.0)
    top_p=0.95,            # Nucleus sampling
    top_k=40,              # Top-k sampling
    max_output_tokens=1024, # Max response length
    stop_sequences=["END"]  # Stop generation at these sequences
)

response = model.generate_content(
    "Write a story about...",
    generation_config=config
)
```

### Safety Settings

```python
from vertexai.generative_models import HarmCategory, HarmBlockThreshold

safety_settings = {
    HarmCategory.HARM_CATEGORY_HATE_SPEECH: HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE,
    HarmCategory.HARM_CATEGORY_DANGEROUS_CONTENT: HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE,
    HarmCategory.HARM_CATEGORY_SEXUALLY_EXPLICIT: HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE,
    HarmCategory.HARM_CATEGORY_HARASSMENT: HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE,
}

response = model.generate_content(
    prompt,
    safety_settings=safety_settings
)
```

---

## 6.3 Embeddings API

Convert text to vectors for semantic search and similarity.

### Creating Embeddings

```python
from vertexai.language_models import TextEmbeddingModel

model = TextEmbeddingModel.from_pretrained("textembedding-gecko@003")

# Single text
embeddings = model.get_embeddings(["Hello, world!"])
vector = embeddings[0].values  # 768-dimensional vector

# Batch embeddings
texts = ["First document", "Second document", "Third document"]
embeddings = model.get_embeddings(texts)
for i, embedding in enumerate(embeddings):
    print(f"Text {i}: {len(embedding.values)} dimensions")
```

### Task Types

Different embeddings for different tasks:

```python
# For retrieval (documents and queries)
embeddings = model.get_embeddings(
    texts=["document text"],
    task_type="RETRIEVAL_DOCUMENT"
)

query_embeddings = model.get_embeddings(
    texts=["user query"],
    task_type="RETRIEVAL_QUERY"
)

# For similarity/clustering
embeddings = model.get_embeddings(
    texts=["text for similarity"],
    task_type="SEMANTIC_SIMILARITY"
)

# For classification
embeddings = model.get_embeddings(
    texts=["text to classify"],
    task_type="CLASSIFICATION"
)
```

### Use Cases

1. **Semantic Search**: Find similar documents
2. **RAG**: Retrieve relevant context for LLM
3. **Clustering**: Group similar items
4. **Anomaly Detection**: Find outliers
5. **Recommendations**: Find similar products/content

---

## 6.4 RAG on GCP (Your Expertise!)

This maps directly to your RAG experience at Fortinet.

### RAG Architecture on GCP

```
┌─────────────────────────────────────────────────────────────────┐
│                     RAG System on GCP                            │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Ingestion Pipeline                     │   │
│  │                                                           │   │
│  │  Documents    Chunking     Embeddings    Vector Store    │   │
│  │  (GCS)    →   (Cloud     →   (Vertex   →   (Vertex AI    │   │
│  │               Functions)     AI)           Vector Search) │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Query Pipeline                         │   │
│  │                                                           │   │
│  │  User Query → Embed → Vector Search → Retrieve → LLM     │   │
│  │              Query    (find similar)  Context    (Gemini) │   │
│  │                                                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Option 1: Vertex AI Search (Managed RAG)

The easiest way - Google handles the RAG pipeline:

```python
from google.cloud import discoveryengine_v1 as discoveryengine

# Create data store
client = discoveryengine.DataStoreServiceClient()
data_store = client.create_data_store(
    parent=f"projects/{project}/locations/global/collections/default_collection",
    data_store=discoveryengine.DataStore(
        display_name="my-knowledge-base",
        industry_vertical="GENERIC",
        solution_types=["SOLUTION_TYPE_SEARCH"]
    ),
    data_store_id="my-data-store"
)

# Import documents (from GCS, BigQuery, or websites)
# Done via Cloud Console or API

# Search with RAG
search_client = discoveryengine.SearchServiceClient()
response = search_client.search(
    serving_config=f"projects/{project}/locations/global/collections/default_collection/dataStores/my-data-store/servingConfigs/default_config",
    query="How do I configure VPC?",
    page_size=10
)
```

### Option 2: Custom RAG with Vertex AI Vector Search

Full control (similar to what you built at Fortinet):

```python
from google.cloud import aiplatform
from vertexai.language_models import TextEmbeddingModel

# 1. Create embeddings for documents
embedding_model = TextEmbeddingModel.from_pretrained("textembedding-gecko@003")

documents = [
    {"id": "doc1", "text": "GCP VPC is a global resource..."},
    {"id": "doc2", "text": "Cloud Storage provides object storage..."},
    # ... more documents
]

# Create embeddings
for doc in documents:
    embedding = embedding_model.get_embeddings(
        [doc["text"]],
        task_type="RETRIEVAL_DOCUMENT"
    )[0].values
    doc["embedding"] = embedding

# 2. Create Vector Search index
aiplatform.init(project='my-project', location='us-central1')

# Create index
index = aiplatform.MatchingEngineIndex.create_tree_ah_index(
    display_name="my-rag-index",
    dimensions=768,  # gecko embedding dimension
    approximate_neighbors_count=10,
    distance_measure_type="DOT_PRODUCT_DISTANCE"
)

# Create index endpoint
index_endpoint = aiplatform.MatchingEngineIndexEndpoint.create(
    display_name="my-rag-endpoint",
    public_endpoint_enabled=True
)

# Deploy index to endpoint
index_endpoint.deploy_index(
    index=index,
    deployed_index_id="deployed-rag-index"
)

# 3. Query function
def rag_query(user_query: str) -> str:
    # Embed query
    query_embedding = embedding_model.get_embeddings(
        [user_query],
        task_type="RETRIEVAL_QUERY"
    )[0].values

    # Search for similar documents
    response = index_endpoint.find_neighbors(
        deployed_index_id="deployed-rag-index",
        queries=[query_embedding],
        num_neighbors=5
    )

    # Get document texts from results
    context_docs = []
    for neighbor in response[0]:
        doc_id = neighbor.id
        # Fetch document text from your storage
        doc_text = get_document_by_id(doc_id)
        context_docs.append(doc_text)

    # Generate response with context
    context = "\n\n".join(context_docs)
    prompt = f"""Based on the following context, answer the question.

Context:
{context}

Question: {user_query}

Answer:"""

    model = GenerativeModel('gemini-1.5-pro')
    response = model.generate_content(prompt)
    return response.text
```

### RAG Best Practices

**Chunking strategies**:
- Fixed size (e.g., 512 tokens)
- Semantic (by section/paragraph)
- Sliding window with overlap
- Recursive (split by separators)

**Retrieval optimization**:
- Hybrid search (keyword + semantic)
- Re-ranking retrieved results
- Query expansion/rewriting
- Metadata filtering

**Generation optimization**:
- Include source citations
- Handle "I don't know" cases
- Use system prompts for consistency
- Implement safety guardrails

---

## 6.5 Grounding

Reduce hallucinations by grounding responses in data.

### Grounding with Google Search

```python
from vertexai.generative_models import GenerativeModel, Tool, grounding

model = GenerativeModel('gemini-1.5-pro')

# Ground with Google Search
tool = Tool.from_google_search_retrieval(
    grounding.GoogleSearchRetrieval()
)

response = model.generate_content(
    "What are the latest GCP announcements?",
    tools=[tool]
)

print(response.text)
# Response includes grounding metadata with sources
```

### Grounding with Vertex AI Search

```python
from vertexai.generative_models import grounding

# Ground with your own data store
tool = Tool.from_retrieval(
    grounding.Retrieval(
        grounding.VertexAISearch(
            datastore=f"projects/{project}/locations/global/collections/default_collection/dataStores/{datastore_id}"
        )
    )
)

response = model.generate_content(
    "How do I configure the API?",
    tools=[tool]
)
```

---

## 6.6 Function Calling

Enable LLMs to call external functions/APIs.

### Defining Functions

```python
from vertexai.generative_models import (
    GenerativeModel,
    Tool,
    FunctionDeclaration
)

# Define function schema
get_weather_func = FunctionDeclaration(
    name="get_weather",
    description="Get the current weather for a location",
    parameters={
        "type": "object",
        "properties": {
            "location": {
                "type": "string",
                "description": "The city name"
            },
            "unit": {
                "type": "string",
                "enum": ["celsius", "fahrenheit"]
            }
        },
        "required": ["location"]
    }
)

create_ticket_func = FunctionDeclaration(
    name="create_support_ticket",
    description="Create a support ticket for a customer issue",
    parameters={
        "type": "object",
        "properties": {
            "title": {"type": "string"},
            "description": {"type": "string"},
            "priority": {
                "type": "string",
                "enum": ["low", "medium", "high", "critical"]
            }
        },
        "required": ["title", "description"]
    }
)

# Create tool
tool = Tool(function_declarations=[get_weather_func, create_ticket_func])

model = GenerativeModel(
    'gemini-1.5-pro',
    tools=[tool]
)
```

### Handling Function Calls

```python
def handle_function_call(function_call):
    """Execute the function and return result."""
    if function_call.name == "get_weather":
        location = function_call.args["location"]
        unit = function_call.args.get("unit", "celsius")
        # Call actual weather API
        return {"temperature": 22, "unit": unit, "condition": "sunny"}

    elif function_call.name == "create_support_ticket":
        title = function_call.args["title"]
        description = function_call.args["description"]
        priority = function_call.args.get("priority", "medium")
        # Create ticket in your system
        return {"ticket_id": "TKT-12345", "status": "created"}

# Chat with function calling
chat = model.start_chat()
response = chat.send_message("What's the weather in Singapore?")

# Check if model wants to call a function
if response.candidates[0].function_calls:
    function_call = response.candidates[0].function_calls[0]

    # Execute the function
    result = handle_function_call(function_call)

    # Send function result back to model
    response = chat.send_message(
        Part.from_function_response(
            name=function_call.name,
            response=result
        )
    )

    print(response.text)  # Model's natural language response
```

---

## 6.7 Model Tuning

Customize models for specific use cases.

### Tuning Types

| Type | Data Required | Use Case |
|------|---------------|----------|
| **Prompt tuning** | Few examples | Quick customization |
| **Supervised tuning** | 100+ examples | Specific tasks |
| **RLHF** | Human feedback | Alignment |

### Supervised Fine-tuning

```python
from vertexai.language_models import TextGenerationModel

# Prepare training data
# Format: {"input_text": "...", "output_text": "..."}
training_data = "gs://bucket/training_data.jsonl"

# Load base model
model = TextGenerationModel.from_pretrained("text-bison@002")

# Start tuning job
tuning_job = model.tune_model(
    training_data=training_data,
    train_steps=300,
    tuning_job_location="us-central1",
    tuned_model_location="us-central1"
)

# Wait for completion
tuned_model = tuning_job.result()

# Use tuned model
response = tuned_model.predict("Your custom prompt...")
```

---

## 6.8 Vertex AI Agent Builder

Build conversational AI agents.

### Agent Types

**Conversational Agents** (Dialogflow CX):
- Multi-turn conversations
- Intent detection
- Entity extraction
- Integration with backends

**Search Agents**:
- Answer questions from documents
- RAG-based
- Citations included

**Generative Agents**:
- LLM-powered responses
- Function calling
- Grounded in your data

### Creating a Basic Agent

```python
from google.cloud import dialogflowcx_v3 as dialogflow

# Create agent
agent_client = dialogflow.AgentsClient()
agent = agent_client.create_agent(
    parent=f"projects/{project}/locations/{location}",
    agent=dialogflow.Agent(
        display_name="GCP Support Agent",
        default_language_code="en",
        time_zone="America/Los_Angeles",
        enable_stackdriver_logging=True,
        gen_app_builder_settings=dialogflow.Agent.GenAppBuilderSettings(
            engine=f"projects/{project}/locations/global/collections/default_collection/engines/my-engine"
        )
    )
)
```

---

## 6.9 Responsible AI

Google's approach to ethical AI.

### Key Principles

1. **Be socially beneficial**
2. **Avoid creating or reinforcing unfair bias**
3. **Be built and tested for safety**
4. **Be accountable to people**
5. **Incorporate privacy design principles**
6. **Uphold high standards of scientific excellence**
7. **Be made available for uses that accord with these principles**

### Implementation in Vertex AI

**Content filtering**: Built-in safety filters
**Explainability**: Model cards, feature importance
**Fairness**: Bias detection tools
**Privacy**: Data governance, VPC-SC

---

## 6.10 Common Gen AI Troubleshooting

### Issues and Solutions

**High latency**:
1. Use streaming for long responses
2. Consider Gemini Flash for speed
3. Reduce output token limit
4. Cache frequent queries

**Poor response quality**:
1. Improve prompts (be specific)
2. Add examples (few-shot)
3. Use system instructions
4. Consider fine-tuning
5. Add grounding

**Hallucinations**:
1. Enable grounding
2. Use RAG with verified data
3. Lower temperature
4. Add fact-checking prompts
5. Include "I don't know" options

**Rate limiting**:
1. Implement exponential backoff
2. Request quota increase
3. Use batch API for bulk
4. Implement caching

**Token limits exceeded**:
1. Summarize long inputs
2. Use chunking strategies
3. Upgrade to higher context model (Gemini 1.5)

---

## Chapter 6 Review Questions

1. When would you use Vertex AI Search vs custom RAG implementation?

2. How does grounding help reduce hallucinations?

3. Explain the difference between embeddings for retrieval vs similarity.

4. A customer's RAG system is returning irrelevant results. How do you debug?

5. What factors affect Gemini response latency and how can you optimize?

6. How would you implement a support chatbot that can access customer data and create tickets?

---

## Chapter 6 Hands-On Exercises

### Exercise 6.1: Basic Gemini
1. Use Gemini API for text generation
2. Experiment with different temperature settings
3. Try multimodal prompts with images

### Exercise 6.2: Build a RAG System
1. Create embeddings for a set of documents
2. Store in Vertex AI Vector Search
3. Build a query pipeline
4. Test retrieval quality

### Exercise 6.3: Grounded Chatbot
1. Create a Vertex AI Search data store
2. Import your documents
3. Build a chat interface with grounding
4. Verify responses include citations

---

## Connecting to Your Experience

Your RAG work at Fortinet maps directly to GCP:

| Your Experience | GCP Equivalent |
|-----------------|----------------|
| Document chunking | Same concepts apply |
| Embedding pipeline | Vertex AI Embeddings API |
| Vector storage | Vertex AI Vector Search |
| LLM integration | Gemini API |
| Chatbot interface | Agent Builder / Dialogflow |

**Interview tip**: Be ready to discuss your RAG architecture and how you'd implement it on GCP. This is a strong differentiator for this AI/ML-focused role.

---

## Key Takeaways

1. **Gemini is the flagship** - Use for most generative AI tasks
2. **Grounding reduces hallucinations** - Essential for enterprise use
3. **RAG is a pattern, not a product** - Can build custom or use Vertex AI Search
4. **Function calling enables actions** - Connect LLMs to real systems
5. **Responsible AI is built in** - But know how to configure it

---

[Next Chapter: TCP/IP & The Network Stack →](./07-networking-tcpip.md)
