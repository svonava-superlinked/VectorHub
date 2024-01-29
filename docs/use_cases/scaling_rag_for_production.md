<!-- SEO SUMMARY:  Retrieval-augmented Generation (RAG) combines Large Language Models (LLMs) with external data, addressing the issue of machine hallucinations, that is, incorrect information generated by AI. When developing RAG systems, scalability is often an afterthought. This can make it difficult to move from development to production and creates unnecessary costs. By using the right tools and designing a RAG pipeline with production workloads in mind, teams can scale their solutions much more efficiently. -->

# Scaling RAG for Production

![](assets/use_cases/recommender_systems/cover.jpg) <Placeholder>

## Know the difference

The goals and requirements of development and production are often very different. This is particularly true for new technologies like Large Language Models (LLMs) and Retrieval-augmented Generation (RAG), where organizations often prioritize rapid experimentation to test the waters before committing more resources. Once the important stakeholders are convinced, the focus shifts from demonstrating that something *can* create value to actually *creating* value. And how to do that? It's time for production.

But what exactly does it mean to "productionize" something? In the context of RAG systems and similar technologies, to productionize means to transition from a prototype or test environment to a stable, operational state where the system is readily accessible and reliable for end users. This involves ensuring the system can be accessed remotely, such as via a URL, and will remain operational independently of any single user's machine. Additionally, productionizing involves scaling the system to handle varying levels of user demand and traffic, ensuring consistent performance and availability.

The harsh truth is that until a system is put into production, its Return on Investment (ROI) is typically zero. However, the hurdles involved in making this happen are often underestimated by management. Productionizing is always a trade-off between performance and costs, and this is no different for Retrieval-augmented Generation (RAG) systems. In case you’re not familiar with what RAG is or simply want to refresh the basics, consider reading an introductory article first.

## The basics of RAG

Let’s review the most basic RAG workflow:

1. Submit a text query to an embedding model, which converts it into a semantically meaningful vector embedding.
2. Send the resulting query vector embedding to where your document embeddings are stored - typically a vector database.
3. Retrieve the most relevant document chunks, determined by the proximity of the query vector embedding to the embedded document chunks.
4. Add the retrieved document chunks as context to the query vector embedding and send it to the LLM.
5. The LLM generates a response utilizing the retrieved context.

According to this workflow, the following components are required: an embedding model, a store for document and vector embeddings, a retriever, and a LLM. While RAG workflows can become significantly more complex, incorporating methods like metadata filtering and retrieval reranking, it’s essential to first establish a strong foundation with these basic elements.

## The LangChain question

LangChain has arguably become the most prominent LLM library to this date. A lot of developers are using it to build Proof-of-Concepts (PoC) and Minimal Viable Products (MVPs) or to simply test new ideas. While there has been a lot of discussion about LangChain in production, *most* of the criticism can be boiled down to personal preference and the fact that LangChain was originally built to address problems occurring much earlier in the development cycle.

So what to do? Keep in mind that this is merely my personal opinion since there are no gold standards for which tools to use yet, but I’m convinced that there is no universal answer to this question. All of the major LLM and RAG libraries - LangChain, LlamaIndex and Haystack, to name my personal top three - have what it takes to producitonize a RAG system. And there’s a simple reason for this: they all have integrations for third-party libraries and providers that will handle the production requirements. I would try to view these tools as interfaces between all the other components. Which one you’d want to choose will depend on the details of your existing tech stack and use case.

## The right tools for this tutorial

Alright, but what will *we* choose for this tutorial? One of the first decisions to make will be where we want to run our system: should we use a cloud service, or should we run it within our own network? Since tutorials should aim to reduce complexity and avoid proprietary solutions where possible, we will opt not to use the cloud option here. While the aforementioned libraries support cloud deployment for AWS, Azure, and GCP, the details of cloud deployment heavily depend on the specific cloud provider you choose. Instead, we will utilize [Ray](https://github.com/ray-project/ray).

Ray is a Python framework for productionizing and scaling machine learning (ML) workloads. It is adaptable for both local environments and Kubernetes, efficiently managing all workload requirements. Ray's design focuses on making the scaling of ML systems seamless, thanks to its range of autoscaling features. While we could opt for Ray integrations like LangChain, LlamaIndex, or Haystack, it's worth considering using Ray directly. This approach might provide more universally applicable insights, given that these integrations are all built upon the same underlying framework.

Before diving in, it's worth mentioning LangServe, a recent addition to the LangChain ecosystem. LangServe is designed to bridge the gap in production tooling. Although it hasn't been widely adopted yet and may take some time to gain traction, the LangChain team is actively responding to feedback to enhance the production experience.

## The Data

### Gathering the data

Every ML journey starts with the data and data needs to be stored somewhere. We will use a part of the LangChain documentation for this tutorial. We will first download the html files and then create a [Ray dataset](https://docs.ray.io/en/latest/data/data.html) of them.

We start with installing all the dependencies that we will use in this tutorial:

```console
pip install ray langchain sentence-transformers qdrant-client einops openai tiktoken fastapi "ray[serve]"
```

Since we will use the OpenAI API in this tutorial, we will need an API key. We export our API key as an environmental variable and then we initialize our Ray environment like this:

```python
import os
import ray

working_dir = "downloaded_docs"

if not os.path.exists(working_dir):
    os.makedirs(working_dir)

# Setting up our Ray environment
ray.init(runtime_env={
    "env_vars": {
        "OPENAI_API_KEY": os.environ["OPENAI_API_KEY"],
    },
    "working_dir": str(working_dir)
})
```

In order to work with the LangChain documentation, we need to download the html files and process them. Scraping html files can get very tricky and the details depend heavily on the structure of the website you’re trying to scrape. The functions below are only meant to be used in the context of this tutorial. 

```python
import requests
from bs4 import BeautifulSoup
from urllib.parse import urlparse, urljoin
from concurrent.futures import ThreadPoolExecutor, as_completed
import re

def sanitize_filename(filename):
    filename = re.sub(r'[\\/*?:"<>|]', '', filename)  # Remove problematic characters
    filename = re.sub(r'[^\x00-\x7F]+', '_', filename)  # Replace non-ASCII characters
    return filename

def is_valid(url, base_domain):
    parsed = urlparse(url)
    valid = bool(parsed.netloc) and parsed.path.startswith("/docs/expression_language/")
    return valid

def save_html(url, folder):
    try:
        headers = {'User-Agent': 'Mozilla/5.0'}
        response = requests.get(url, headers=headers)
        response.raise_for_status()

        soup = BeautifulSoup(response.content, 'html.parser')
        title = soup.title.string if soup.title else os.path.basename(urlparse(url).path)
        sanitized_title = sanitize_filename(title)
        filename = os.path.join(folder, sanitized_title.replace(" ", "_") + ".html")

        if not os.path.exists(filename):
            with open(filename, 'w', encoding='utf-8') as file:
                file.write(str(soup))
            print(f"Saved: {filename}")

            links = [urljoin(url, link.get('href')) for link in soup.find_all('a') if link.get('href') and is_valid(urljoin(url, link.get('href')), base_domain)]
            return links
        else:
            return []
    except Exception as e:
        print(f"Error processing {url}: {e}")
        return []

def download_all(start_url, folder, max_workers=5):
    visited = set()
    to_visit = {start_url}

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        while to_visit:
            future_to_url = {executor.submit(save_html, url, folder): url for url in to_visit}
            visited.update(to_visit)
            to_visit.clear()

            for future in as_completed(future_to_url):
                url = future_to_url[future]
                try:
                    new_links = future.result()
                    for link in new_links:
                        if link not in visited:
                            to_visit.add(link)
                except Exception as e:
                    print(f"Error with future for {url}: {e}")
```

Because the documentation is very large, we will only download a subset of it. We will use the documentation of LangChains Expression Language (LCEL), which consists of 28 html pages.

```python
base_domain = "python.langchain.com"
start_url = "https://python.langchain.com/docs/expression_language/"
folder = working_dir

download_all(start_url, folder, max_workers=10)
```

Now that we have downloaded the files, we can use them to create our Ray dataset:

```python
from pathlib import Path

# Ray dataset
document_dir = Path(folder)
ds = ray.data.from_items([{"path": path.absolute()} for path in document_dir.rglob("*.html") if not path.is_dir()])
print(f"{ds.count()} documents")
```

Great! But there is something left to do before we can move on to the next phase of our workflow. We still need to extract the relevant text from our html files and clean up all the html syntax. For this, we will import BeautifulSoup to parse the files and find relevant html tags.

```python
from bs4 import BeautifulSoup, NavigableString

def extract_text_from_element(element):
    texts = []
    for elem in element.descendants:
        if isinstance(elem, NavigableString):
            text = elem.strip()
            if text:
                texts.append(text)
    return "\n".join(texts)

def extract_main_content(record):
    with open(record["path"], "r", encoding="utf-8") as html_file:
        soup = BeautifulSoup(html_file, "html.parser")

    main_content = soup.find(['main', 'article'])  # Add any other tags or class_="some-class-name" here
    if main_content:
        text = extract_text_from_element(main_content)
    else:
        text = "No main content found."

    path = record["path"]
    return {"path": path, "text": text}

```

We can now use this extraction process by utilizing Ray’s map() function. This let’s us run multiple processes in parallel.

```python
# Extract content
content_ds = ds.map(extract_main_content)
content_ds.count()

```

Awesome, this will be our dataset. Ray Datasets are optimized for performance at scale - which will make productionizing our system easier for us. Having to manually adjust code while your application grows can get very costly and is prone to errors.

### Processing the data

The next three processing steps will consist of chunking, embedding and indexing our data source. Chunking is the process of splitting your documents into multiple smaller parts. Not only will this be necessary to make your data meet the LLM’s context length limits, it also helps to keep contexts specific enough to remain relevant. On the other hand, if your chunks are too small, the information retrieved might become too narrow. The exact chunk size will depend on your data, the models used and your use case. We will use a standard value here that has been used in a lot of applications.

Let’s define our text splitting logic first, we will use a standard text splitter from LangChain:

```python
from functools import partial
from langchain.text_splitter import RecursiveCharacterTextSplitter

# Defining our text splitting function
def chunking(document, chunk_size, chunk_overlap):
    text_splitter = RecursiveCharacterTextSplitter(
        separators=["\n\n", "\n"],
        chunk_size=chunk_size,
        chunk_overlap=chunk_overlap,
        length_function=len)

    chunks = text_splitter.create_documents(
        texts=[document["text"]], 
        metadatas=[{"path": document["path"]}])
    return [{"text": chunk.page_content, "path": chunk.metadata["path"]} for chunk in chunks]
```

Again, utilize map() for scalability:

```python
chunks_ds = content_ds.flat_map(partial(
    chunking, 
    chunk_size=512, 
    chunk_overlap=50))
print(f"{chunks_ds.count()} chunks")
```

### Embedding the data

Why are we doing all this again? To make our data retrievable in an efficient way, right. We want relevant answers to our questions. And to find the most relevant text sections for a query, we can use a  pretrained model to create vector embeddings for both our data chunks and the query itself. By measuring the distance between the chunk embeddings and the query embedding, we can identify the most relevant chunks, typically referred to as the 'top-k' chunks. There are various pretrained models suitable for this task. We will be using the popular 'bge-base-en-v1.5' model because, at the time of writing this tutorial, it ranks as the highest-performing model of its size on the [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard). For convenience, we will continue using LangChain:

```python
from langchain.embeddings import OpenAIEmbeddings
from langchain.embeddings.huggingface import HuggingFaceEmbeddings
import numpy as np
from ray.data import ActorPoolStrategy

def get_embedding_model(embedding_model_name, model_kwargs, encode_kwargs):
    embedding_model = HuggingFaceEmbeddings(
            model_name=embedding_model_name,
            model_kwargs=model_kwargs,
            encode_kwargs=encode_kwargs)
    return embedding_model
```

This time we will define a class because we want to use map_batches() instead of map(). This function requires a class object with a **call** method.

```python
class EmbedChunks:
    def __init__(self, model_name):
        self.embedding_model = get_embedding_model(
            embedding_model_name=model_name,
            model_kwargs={"device": "cuda"},
            encode_kwargs={"device": "cuda", "batch_size": 100})
    def __call__(self, batch):
        embeddings = self.embedding_model.embed_documents(batch["text"])
        return {"text": batch["text"], "path": batch["path"], "embeddings": embeddings}

# Embedding our chunks
embedding_model_name = "BAAI/bge-base-en-v1.5"
embedded_chunks = chunks_ds.map_batches(
    EmbedChunks,
    fn_constructor_kwargs={"model_name": embedding_model_name},
    batch_size=100, 
    num_gpus=1,
    concurrency=1)
```

### Indexing the data

Our chunks are embedded, and now we need to store them somewhere. For the sake of this tutorial, we will utilize Qdrant’s new in-memory feature. This feature allows us to experiment with our code rapidly without the need to set up a fully-fledged instance. However, for deployment in a production environment, it is advisable to rely on more robust and scalable solutions — these might be hosted either within your own network or by a third-party provider. Detailed guidance on setting up such solutions is beyond the scope of this tutorial.

```python
from qdrant_client import QdrantClient
from qdrant_client.http.models import Distance, VectorParams

# Initalizing a local client in-memory
client = QdrantClient(":memory:")

client.recreate_collection(
   collection_name="documents",
   vectors_config=VectorParams(size=embedding_size, distance=Distance.COSINE),
)
```

We could use Ray again, but for the purpose of this tutorial, we will choose to use pandas. The reason is that with Ray, the next processing step would require more than 2 CPU cores, which would make this tutorial incompatible with the free tier of Google Colab. Fortunately, Ray allows us to convert our dataset into a pandas DataFrame with a single line of code.

```python
emb_chunks_df = embedded_chunks.to_pandas()
```

Now we define and execute our data storage function:

```python
from qdrant_client.models import PointStruct

def store_results(df, collection_name="documents", client=client):
	  # Defining our data structure
    points = [
        # PointStruct is the data classs used in Qdrant
        PointStruct(
            id=hash(path),  # Unique ID for each point
            vector=embedding,
            payload={
                "text": text,
                "source": path
            }
        )
        for text, path, embedding in zip(df["text"], df["path"], df["embeddings"])
    ]
		
		# Adding our data points to the collection
    client.upsert(
        collection_name=collection_name,
        points=points
    )

store_results(emb_chunks_df)
```

This wraps up the data processing part! Our data is now stored in our vector database and ready to be retrieved.

## The Retrieval

When retrieving data from a vector storage, it is important to use the same embedding model for your query that was used for the source data. Otherwise, the comparison of the vectors would not be meaningful.

```python
import numpy as np 

# Embed query
embedding_model = HuggingFaceEmbeddings(model_name=embedding_model_name)
query = "How to run agents?"
query_embedding = np.array(embedding_model.embed_query(query))
len(query_embedding)
```

Recall the concept of 'top-k' chunks? In Qdrant’s search, the 'limit' parameter is equivalent to 'k'. By default, the search uses cosine similarity as the metric, and the 5 chunks closest to our query embedding will be retrieved from our database:

```python
hits = client.search(
    collection_name="documents",
    query_vector=query_embedding,
    limit=5  # Return 5 closest points
)

context_list = [hit.payload["text"] for hit in hits]
context = "\n".join(context_list)
```

And we will rewrite this as a function for later use:

```python
def semantic_search(query, embedding_model, k):
    query_embedding = np.array(embedding_model.embed_query(query))
    hits = client.search(
      collection_name="documents",
      query_vector=query_embedding,
      limit=5  # Return 5 closest points
    )

    context_list = [{"id": hit.id, "source": str(hit.payload["source"]), "text": hit.payload["text"]} for hit in hits]
    return context_list
```

## The Generation

We are so close to getting our answers! We set up everything we need to query our LLM — and we did so in a scalable way. Instead of simply querying the model for a response, we will first retrieve relevant context from our vector database and then add it to the query. We can think of this as a query that is informed by our data. 

For this, we will use a simplified version of the implementation provided in Ray's [LLM repository](https://github.com/ray-project/llm-applications/blob/main/rag/generate.py). This version is adapted to our code and leaves out a bunch of advanced retrieval techniques, such as reranking and hybrid search. We will use gpt-3.5-turbo as our LLM and we will query it via the OpenAI API. 

```python
from openai import OpenAI

def get_client(llm):
  api_key = os.environ["OPENAI_API_KEY"]
  client = OpenAI(api_key=api_key)
  return client

def generate_response(
    llm,
    max_tokens=None,
    temperature=0.0,
    stream=False,
    system_content="",
    assistant_content="",
    user_content="",
    max_retries=1,
    retry_interval=60,
):
    """Generate response from an LLM."""
    retry_count = 0
    client = get_client(llm=llm)
    messages = [
        {"role": role, "content": content}
        for role, content in [
            ("system", system_content),
            ("assistant", assistant_content),
            ("user", user_content),
        ]
        if content
    ]
    while retry_count <= max_retries:
        try:
            chat_completion = client.chat.completions.create(
                model=llm,
                max_tokens=max_tokens,
                temperature=temperature,
                stream=stream,
                messages=messages,
            )
            return prepare_response(chat_completion, stream=stream)

        except Exception as e:
            print(f"Exception: {e}")
            time.sleep(retry_interval)  # default is per-minute rate limits
            retry_count += 1
    return ""

def response_stream(chat_completion):
    for chunk in chat_completion:
        content = chunk.choices[0].delta.content
        if content is not None:
            yield content

def prepare_response(chat_completion, stream):
    if stream:
        return response_stream(chat_completion)
    else:
        return chat_completion.choices[0].message.content
```

This is how we would generate a response:

```python
# Generating our response
query = "How to run agents?"
response = generate_response(
    llm="gpt-3.5-turbo",
    temperature=0.0,
    stream=True,
    system_content="Answer the query using the context provided. Be succinct.",
    user_content=f"query: {query}, context: {context_list}")
# Stream response
for content in response:
    print(content, end='', flush=True)
```

However, to make using our application even more convenient, we will implement our workflow within a single class. This again is a simplified and adapted version of Ray’s official implementation. This QueryAgent class will take care of all the steps we implemented above for us, including a few additional utility functions.

```python
import tiktoken

def get_num_tokens(text):
    enc = tiktoken.get_encoding("cl100k_base")
    return len(enc.encode(text))


def trim(text, max_context_length):
    enc = tiktoken.get_encoding("cl100k_base")
    return enc.decode(enc.encode(text)[:max_context_length])

class QueryAgent:
    def __init__(
        self,
        embedding_model_name="BAAI/bge-base-en-v1.5",
        llm="gpt-3.5-turbo",
        temperature=0.0,
        max_context_length=4096,
        system_content="",
        assistant_content="",
    ):
        # Embedding model
        self.embedding_model = get_embedding_model(
            embedding_model_name=embedding_model_name,
            model_kwargs={"device": "cuda"},
            encode_kwargs={"device": "cuda", "batch_size": 100},
        )

        # LLM
        self.llm = llm
        self.temperature = temperature
        self.context_length = int(
            0.5 * max_context_length
        ) - get_num_tokens(  # 50% of total context reserved for input
            system_content + assistant_content
        )
        self.max_tokens = int(
            0.5 * max_context_length
        )  # max sampled output (the other 50% of total context)
        self.system_content = system_content
        self.assistant_content = assistant_content

    def __call__(
        self,
        query,
        num_chunks=5,
        stream=True,
    ):
        # Get top_k context
        context_results = semantic_search(
            query=query, embedding_model=self.embedding_model, k=num_chunks
        )

        # Generate response
        document_ids = [item["id"] for item in context_results]
        context = [item["text"] for item in context_results]
        sources = [item["source"] for item in context_results]
        user_content = f"query: {query}, context: {context}"
        answer = generate_response(
            llm=self.llm,
            max_tokens=self.max_tokens,
            temperature=self.temperature,
            stream=stream,
            system_content=self.system_content,
            assistant_content=self.assistant_content,
            user_content=trim(user_content, self.context_length),
        )

        # Result
        result = {
            "question": query,
            "sources": sources,
            "document_ids": document_ids,
            "answer": answer,
            "llm": self.llm,
        }
        return result
```

And this is how we can use it:

```python
import json 

query = "How to run an agent?"
system_content = "Answer the query using the context provided. Be succinct."
agent = QueryAgent(
    embedding_model_name="BAAI/bge-base-en-v1.5",
    llm="gpt-3.5-turbo",
    max_context_length=4096,
    system_content=system_content)
result = agent(query=query, stream=False)
print(json.dumps(result, indent=2))
```

## The Serving

Finally! Our application is running and we are about to serve it. Fortunately, Ray makes this very straightforward with their [Ray Serve](https://docs.ray.io/en/latest/serve/index.html) module. We will use Ray Serve in combination with FastAPI and pydantic. The @serve.deployment decorator lets us define how many replicas and compute resources we want to use and Ray’s autoscaling will handle the rest. Two Ray serve decorators are all we need to modify our FastAPI application for production.

```python
import pickle
import requests
from typing import List

from fastapi import FastAPI
from pydantic import BaseModel
from ray import serve

# Initialize application
app = FastAPI()

class Query(BaseModel):
    query: str

class Response(BaseModel):
    llm: str
    question: str
    sources: List[str]
    response: str

@serve.deployment(num_replicas=1, ray_actor_options={"num_cpus": 2, "num_gpus": 1})
@serve.ingress(app)
class RayAssistantDeployment:
    def __init__(self, embedding_model_name, embedding_dim, llm):
        
        # Query agent
        system_content = "Answer the query using the context provided. Be succinct. " \
            "Contexts are organized in a list of dictionaries [{'text': <context>}, {'text': <context>}, ...]. " \
            "Feel free to ignore any contexts in the list that don't seem relevant to the query. "
        self.gpt_agent = QueryAgent(
            embedding_model_name=embedding_model_name,
            llm="gpt-3.5-turbo",
            max_context_length=4096,
            system_content=system_content)

    @app.post("/query")
    def query(self, query: Query) -> Response:
        result = self.gpt_agent(
            query=query.query, 
            stream=False
            )
        return Response.parse_obj(result)
```

And now we will deploy our application:

```python
# Deploying our application with Ray Serve
deployment = RayAssistantDeployment.bind(
    embedding_model_name="BAAI/bge-base-en-v1.5",
    embedding_dim=768,
    llm="gpt-3.5.-turbo")

serve.run(deployment, route_prefix="/")
```

Our FastAPI endpoint can then be queried like any other API, while Ray handles the workload automatically:

```python
# Performing inference
data = {"query": "How to run an agent?"}
response = requests.post(
"https://127.0.0.1:8000/query", json=data
)

try:
  print(response.json())
except:
  print(response.text)
```

Wow! This was quite the journey. I’m glad you made it this far and we hope you have learned as much as we did over the course of this tutorial. Here's one final reminder — and a suggestion for what to explore next.

### Production is only the start

Often, reaching production is viewed as the primary goal, while maintenance is overlooked. However, the reality is that maintaining an application is a continuous and important task.

Regular assessment and improvement of your application are essential. This might include routinely updating your data to guarantee that your application has the latest information, or keeping an eye on performance to prevent any degradation. For smoother operations, integrating your workflows with CI/CD pipelines is recommended.

Finally, there are are other critical aspects to consider that were outside of the scope of this article:

- **Advanced Development** Pre-training, finetuning, prompt engineering and other in-depth development techniques
- **Evaluation** LLM Evaluation can get very tricky due to randomness and qualitative metrics, RAG also consits of multiple complex parts
- **Compliance** Adhering to data privacy laws and regulations, especially when handling personal or sensitive information.

---
## Contributors

- [Pascal Biese, author](https://www.linkedin.com/in/pascalbiese/)