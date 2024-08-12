# GeoJSON-Connector: Semantic Search for Geospatial (OSM) Features

This guide outlines how the [GeoJSON-connector](https://github.com/52North/innovation-prize/blob/dev/search-app/server/connectors/geojson_osm.py) is used to prepare [embeddings](https://huggingface.co/blog/getting-started-with-embeddings) for OpenStreetMap (OSM) features that can be utilized for a semantic search system. The goal is to enable users to search for geospatial data using natural language queries, retrieving both specific features (e.g., "Eiffel Tower") and collections of features (e.g., "hospitals in Berlin").

## Overview

The process involves the following main steps:

1. Fetching and processing OSM data
2. Creating meaningful textual representations of features
3. Generating two types of documents:
   a. Individual feature documents
   b. Feature collection documents
4. Loading documents into a vector store
5. Implementing the semantic search

## Detailed Steps

### 1. Fetching and Processing OSM Data
The [GeoJSON-connector](https://github.com/52North/innovation-prize/blob/dev/search-app/server/connectors/geojson_osm.py) is designed to work with OpenStreetMap (OSM) datasets in GeoJSON format. These datasets contain features with geometries and associated properties. Here's how to set it up and use it effectively:

#### Providing Data Sources
The [config file](https://github.com/52North/innovation-prize/blob/6b1ab1c2532eefe30d2e6849ea28ced2e50e49e1/search-app/server/config/config.json) offers two options for specifying GeoJSON data sources:

**1. Via PyGeoAPI:** Use this if your data is accessible through a PyGeoAPI instance:
```
"web_geojson_resources": [
   "https://<your-pygeoapi-instance>"
]
```
**2. Local File Storage:** Use this if you have GeoJSON files stored locally on your server:

```
"local_geojson_files": [
   "./data/"
]
```
The connector can then be instantiated in the main application ([server.py](https://github.com/52North/innovation-prize/blob/6b1ab1c2532eefe30d2e6849ea28ced2e50e49e1/search-app/server/app/server.py)) like:

```
from connectors.geojson_osm import GeoJSON

geojson_osm_connector = GeoJSON()`
```
#### Filtering Specific Feature Types
Some OSM features have generic types (e.g., `building:yes`) that may not be useful for specific category searches. To focus on more specific feature types:
```
geojson_osm_connector = GeoJSON(tag_name="building")
```

This initialization will filter out features where the `building` tag is just `yes`, keeping only those with more specific building types (e.g., `building:hospital`).

### 2. Creating Meaningful Textual Representations

To enable semantic search on OSM features, it's essential to generate meaningful textual representations (embeddings) for the features. This requires converting the structured metadata from the OSM features into unstructured text documents, which can then be passed to an [embedding model](https://huggingface.co/docs/chat-ui/configuration/embeddings).

#### Understanding OSM Features

OSM features are represented using key-value pairs, with most of the semantic information stored in the `tags` key. For example:

```json
"tags": {
    "addr:city": "Dresden",
    "addr:country": "DE",
    "addr:postcode": "01067",
    "addr:street": "Neumarkt",
    "alt_name": "Kirche Unserer Lieben Frau",
    "amenity": "place_of_worship",
    "architect": "George Bähr",
    "architect:wikidata": "Q66169",
    "building": "church",
    "building:architecture": "baroque",
    "building:material": "sandstone",
    "building:part": "yes",
    ...,
    "opening_hours": "Mo-Fr 10:00-12:00,13:00-18:00",
    "opening_hours:url": "https://www.frauenkirche-dresden.de/kalender/",
    "wikidata": "Q157229",
    "wikimedia_commons": "Category:Frauenkirche (Dresden)",
    "wikipedia": "de:Frauenkirche (Dresden)",
    "year_of_construction": "1726 - 1743 / 2005"
}
```

To generate embeddings, a text document needs to be prepared by extracting and organizing the information from these tags key-value pairs.

For instance:
```python
feature_doc = """
**Name**: Frauenkirche
**Description**: A building that was built as a church. It includes cases where building is no longer used for original purpose.

**Address**: Neumarkt, Dresden, 01067, DE

**Metadata**:
alt_name: Kirche Unserer Lieben Frau
amenity: place_of_worship
architect: George Bähr
architect:wikidata: Q66169
building: church
building:architecture: baroque
building:material: sandstone
building:part: yes
...
opening_hours: Mo-Fr 10:00-12:00,13:00-18:00
opening_hours:url: https://www.frauenkirche-dresden.de/kalender/
wikidata: Q157229
wikimedia_commons: Category:Frauenkirche (Dresden)
wikipedia: de:Frauenkirche (Dresden)
year_of_construction: 1726 - 1743 / 2005
_osm_type: way"""

```
As the feature category is not described verbosely in OSM-features (e.g. `"building": "church"`), the connector provides functionality to fetch tag descriptions from the [OSM-Wiki](https://wiki.openstreetmap.org/wiki/Main_Page) and use the fetched description to enrich the text description:
- `_get_osm_tag_description_async`: Fetches descriptions for OSM tags from the OpenStreetMap wiki.

As shown in the above example: the `"building": "church"` is enriched to "**Description**: A building that was built as a church. It includes cases where building is no longer used for original purpose."

### 3. Generating Documents

Two types of documents are generated:

a. Individual Feature Documents:
   - Created for features with names (e.g., specific landmarks).
   - Includes the feature name, description, and all properties.
   - Useful for queries about specific places.

b. Feature Collection Documents:
   - Groups features by their OSM tag (e.g., all hospitals).
   - Includes a description of the tag and a count of features.
   - Useful for queries about types of places.

The `_features_to_docs` method handles the creation of both document types.

### 4. Loading Documents into a Vector Store

While not shown in the provided code, the next step would be to load these documents into a vector store. This typically involves:

1. Generating embeddings for each document.
2. Storing these embeddings along with the document metadata in a vector database (e.g., Pinecone, Weaviate, or FAISS).

### 5. Implementing the Semantic Search

To implement the search:

1. Convert the user's query into an embedding using the same model used for the documents.
2. Perform a similarity search in the vector store to find the most relevant documents.
3. Retrieve the corresponding geospatial data (either individual features or feature collections).
4. Return the results to the user, possibly with additional context from the document content.

## Code Highlights

Here are some key methods from the provided class:

```python
async def add_descriptions_to_features(self) -> None:
    # Fetches and adds descriptions to features
    ...

def _group_features_by_tag(self) -> Dict[str, List[dict]]:
    # Groups features by their OSM tag
    ...

def _convert_to_geojson(self, data):
    # Converts features to GeoJSON format
    ...

async def _features_to_docs(self) -> List[Document]:
    # Creates documents for individual features and feature collections
    ...
```

## Considerations

- Performance: The class uses asynchronous methods and semaphores to efficiently fetch tag descriptions.
- Scalability: Consider chunking large datasets and processing them in batches.
- Updates: Implement a system to regularly update your vector store as OSM data changes.
- Query Processing: Develop a robust query understanding system to interpret user intentions (e.g., distinguishing between queries for specific places vs. types of places).

By following these steps and utilizing the provided class, you can create a powerful semantic search system for geospatial data, enabling intuitive and flexible querying of OSM features.
