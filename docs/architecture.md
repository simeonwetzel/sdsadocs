## Modules

<img width="80%" height="80%" alt="Server_Architecture" src="https://github.com/simeonwetzel/sdsadocs/assets/90314129/7d2ec2c8-6ecd-4ee5-817c-d7d981003713">

### Indexing Module and Retrieval Module

#### Indexing Module
This module integrates external resources and prepares the collected data for loading into a vector database. It consists of connectors and an indexer. The connectors are abstract Python classes designed to gather data from external sources and format it into a structured format using [LangChain Documents](https://api.python.langchain.com/en/latest/documents/langchain_core.documents.base.Document.html). This structured data is then used by the indexer, which creates embeddings from the documents and loads them into the vector database.

Each connector is specific to a particular type of data resource (e.g., a geojson/OSM connector for reading OSM-type geojsons). Therefore, there is a unique connector for each resource type. In contrast, the indexer is generic and can handle the outputs from all connectors, meaning only one indexer is needed for all connectors.

##### Connectors

So far, the following `connectors` are implemented:

| Connector              | Description                                                                                                                                                                                                                                                                                      | Reference of the connector |
|------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------|
| OSM/Geojson connector  | Can read GeoJSON, including a [FeatureCollection](https://datatracker.ietf.org/doc/html/rfc7946#page-12) with [OSM data](https://www.openstreetmap.org/#map=6/51.331/10.459).                                                                                                                     | [Source](https://github.com/52North/innovation-prize/blob/dev/search-app/server/connectors/geojson_osm.py) |
| Pygeoapi connector     | Can read collections from a [pygeoapi](https://pygeoapi.io/) instance.                                                                                                                                                                                                                            | [Source](https://github.com/52North/innovation-prize/blob/dev/search-app/server/connectors/pygeoapi_retriever.py) |
| Concept Store connector | Can connect to taxonomies, controlled vocabularies, and ontologies (e.g., [GCMD](https://gcmd.earthdata.nasa.gov/KeywordViewer/) or [GEMET](https://www.eionet.europa.eu/gemet/en/about/)) that provide an RDF/SPARQL endpoint. This connector allows concepts and related terms (e.g., broader/narrower terms) to be loaded into the vector database. It enables finding semantically similar terms for potential search queries, so users do not need to interact directly with a SPARQL interface. | [Source](#concept-store-connector) |

#### Indexer Class

The `Indexer` class is responsible for creating and managing embeddings of documents for efficient retrieval and loading them into a vector database. It utilizes the LangChain library for embedding and indexing operations. The `Indexer` class is used by both the **Indexing Module** to load documents into the vector database and the **Retrieval Module** to retrieve documents based on queries.

##### Key Features

- **Embeddings**: Supports both [GPT-4 All](https://www.nomic.ai/gpt4all) and [HuggingFace](https://huggingface.co/) models for generating embeddings from documents. The default model is `nomic-ai/nomic-embed-text-v1.5-GGUF` by [Nomic AI](https://www.nomic.ai/). Any other embedding model available via HuggingFace can also be used. To use a HuggingFace model, set the `use_hf_model` flag to `true` and specify the model name with the `embedding_model` parameter (e.g., `embedding_model='sentence-transformers/all-MiniLM-L6-v2'`).
- **Vector Storage**: Uses [ChromaDB](https://docs.trychroma.com/) for storing and managing vectors persistently.
- **Record Management**: Integrates with an SQLRecordManager for efficient indexing and retrieval operations, maintaining a schema in a local SQLite database.
- **Document Retrieval**: Provides a flexible retriever for similarity searches with configurable parameters for the number of results (`k`) and score threshold.

##### Usage

The `Indexer` class offers several methods to interact with the indexed data:

- `_index(documents: List[Document])`: Indexes a list of documents.
- `_clear()`: Clears the current index.
- `_get_doc_by_id(_id: str)`: Retrieves a document by its ID.
- `_delete_doc_from_index(_id: str)`: Deletes a document from the index and updates the record manager accordingly.

##### Example Initialization

```python
from your_module import Indexer  # Replace with your actual module path

indexer = Indexer(
    index_name="my_index",
    persist_directory="./chroma_db",
    embedding_model="nomic-embed-text-v1.5.f16.gguf",
    use_hf_model=False,
    k=3,
    score_treshold=0.6
)
```

:information_source: The score_threshold argument sets the minimum relevance score that documents must have to be retrieved for a query. Only documents with a relevance score above this threshold will be returned.

#### Retrieval Module
The Retrieval Module uses the Indexer class to retrieve documents from the vector database based on user queries. It enables contextual retrieval through dense retrieval techniques, ensuring relevant documents are fetched efficiently.

##### Endpoints
Each index created by the Indexing Module can be accessed through specific retrieval endpoints. These endpoints are dynamically created based on the available indexes.
This setup ensures that each index can be queried individually, providing a flexible and scalable retrieval mechanism.

##### Example Usage
Below is an example of how to set up the retrieval endpoints:
```python
# Create a dictionary of indexes
indexes = {
    "pygeoapi": Indexer(index_name="pygeoapi"),
}

# Add retrieval routes for each index
for index_name, index_instance in indexes.items():
    add_routes(app, index_instance.retriever, path=f"/retrieve_{index_name}")

```
These endpoints can be accessed with POST requests, allowing you to retrieve documents for specific queries. For example, to retrieve documents from the pygeoapi index, you can use the following endpoint:
```
curl -X POST \
  http://localhost:8000/retrieve_pygeoapi/invoke \
  -H 'Content-Type: application/json' \
  -d '{
    "input": "Precipitation"
}'
```
### Search Criteria and Answer Generation Modules
#### Search Criteria Module
#### Answer Generation Module

