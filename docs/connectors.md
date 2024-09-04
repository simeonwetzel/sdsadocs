# Connectors
Connectors are Python modules that implement classes for interfacing the framework with data resources, such as geospatial data servers, metadata catalogs, or local files.
So far, the following `connectors` are implemented:

| Connector              | Description                                                                                                                                                                                                                                                                                      | Reference of the connector |
|------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------|
| [OSM/Geojson connector](https://sdsadocs.readthedocs.io/en/latest/connectors.html#geojson-connector-semantic-search-for-geospatial-osm-features)  | Can read GeoJSON, including a [FeatureCollection](https://datatracker.ietf.org/doc/html/rfc7946#page-12) with [OSM data](https://www.openstreetmap.org/#map=6/51.331/10.459).                                                                                                                     | [Source](https://github.com/52North/innovation-prize/blob/dev/search-app/server/connectors/geojson_osm.py) |
| Pygeoapi connector     | Can read collections from a [pygeoapi](https://pygeoapi.io/) instance.                                                                                                                                                                                                                            | [Source](https://github.com/52North/innovation-prize/blob/dev/search-app/server/connectors/pygeoapi_retriever.py) |
| Concept Store connector | Can connect to taxonomies, controlled vocabularies, and ontologies (e.g., [GCMD](https://gcmd.earthdata.nasa.gov/KeywordViewer/) or [GEMET](https://www.eionet.europa.eu/gemet/en/about/)) that provide an RDF/SPARQL endpoint. This connector allows concepts and related terms (e.g., broader/narrower terms) to be loaded into the vector database. It enables finding semantically similar terms for potential search queries, so users do not need to interact directly with a SPARQL interface. | [Source](#concept-store-connector) |

## GeoJSON-Connector: Semantic Search for Geospatial (OSM) Features

This guide outlines how the [GeoJSON-connector](https://github.com/52North/innovation-prize/blob/dev/search-app/server/connectors/geojson_osm.py) is used to prepare [embeddings](https://huggingface.co/blog/getting-started-with-embeddings) for OpenStreetMap (OSM) features that can be utilized for a semantic search system. The goal is to enable users to search for geospatial data using natural language queries, retrieving both specific features (e.g., "Eiffel Tower") and collections of features (e.g., "hospitals in Berlin").

### Overview

The process involves the following main steps:

1. Fetching and processing OSM data
2. Creating meaningful textual representations of features
3. Generating two types of documents:
   a. Individual feature documents
   b. Feature collection documents
4. Loading documents into a vector store and implementing the semantic search

### Detailed Steps

#### 1. Fetching and Processing OSM Data
The [GeoJSON-connector](https://github.com/52North/innovation-prize/blob/dev/search-app/server/connectors/geojson_osm.py) is designed to work with OpenStreetMap (OSM) datasets in GeoJSON format. These datasets contain features with geometries and associated properties. Here's how to set it up and use it effectively:

##### Providing Data Sources
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
##### Filtering Specific Feature Types
Some OSM features have generic types (e.g., `building:yes`) that may not be useful for specific category searches. To focus on more specific feature types:
```
geojson_osm_connector = GeoJSON(tag_name="building")
```

This initialization will filter out features where the `building` tag is just `yes`, keeping only those with more specific building types (e.g., `building:hospital`).

#### 2. Creating Meaningful Textual Representations

To enable semantic search on OSM features, it's essential to generate meaningful textual representations (embeddings) for the features. This requires converting the structured metadata from the OSM features into unstructured text documents, which can then be passed to an [embedding model](https://huggingface.co/docs/chat-ui/configuration/embeddings).

##### Understanding OSM Features
OSM features are represented using key-value pairs, with most of the semantic information stored in the `tags` key. Here's an example of a feature's tags:

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
**Creating Text Documents for Embeddings**

To generate embeddings, we need to transform these key-value pairs into a structured text document. Here's an example of how this might look:
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

**Enhancing Feature Descriptions**:

OSM features often use concise tags (e.g., `"building": "church"`) that lack detailed descriptions. To enrich these descriptions, our connector provides functionality to fetch more verbose explanations from the [OSM-Wiki](https://wiki.openstreetmap.org/wiki/Main_Page).

**Key Method** `_get_osm_tag_description_async`:

This asynchronous method fetches detailed descriptions for OSM tags from the OpenStreetMap wiki. It's designed for efficiency, allowing quick retrieval of multiple tag descriptions.
*How it works*:
1. It uses the `tag_name` parameter (specified when instantiating the connector) to determine which tags need descriptions.
2. For each relevant tag, it queries the OSM Wiki asynchronously.
3. It then incorporates the fetched descriptions into the feature's text document.
   
*Example Transformation*:
- Original tag: "building": "church"
- Enhanced description: `"**Description**: A building that was built as a church. It includes cases where the building is no longer used for its original purpose."` ([Source](https://wiki.openstreetmap.org/wiki/Tag:building%3Dchurch))

This enrichment process significantly improves the semantic quality of the feature descriptions, making them more suitable for accurate embeddings and effective semantic search.

#### 3. Generating Documents

Two types of documents are generated:

a. Individual Feature Documents:
   - Created for features with names (e.g., specific landmarks).
   - Includes the feature name, description, and all properties.
   - Useful for queries about specific places.
     
*Example search for a feature stored in a individual feature document: in this case "Frauenkirche in Dresden"*:
![image](https://github.com/user-attachments/assets/8d2561ca-23be-471c-ac6d-cd342b8a871d)


b. Feature Collection Documents:
   - Groups features by their OSM tag (e.g., all hospitals).
   - Includes a description of the tag and a count of features.
   - Useful for queries about types of places.

*Example search for features that share a similar category and are stored in a feature collection document: in this case "Hospitals in Dresden*:
![image](https://github.com/user-attachments/assets/e102bc68-8cf0-4e29-8f8b-8246a1a9753e)


The `_features_to_docs` method handles the creation of both document types.

#### 4. Creating Embeddings, Loading Documents into a Vector Store, and implementing Semantic Search

The final steps are generating embeddings from the documents, loading these documents into a vector store, and then using the same embedding model to retrieve documents from the vector store. This entire process is managed by the [Indexer Class](https://github.com/52North/innovation-prize/blob/6b1ab1c2532eefe30d2e6849ea28ced2e50e49e1/search-app/server/indexing/indexer.py), which is thoroughly [documented here](https://sdsadocs.readthedocs.io/en/latest/architecture.html#indexer-class).

## Pygeoapi connector: Accessing Collections via OGC API Interface"
