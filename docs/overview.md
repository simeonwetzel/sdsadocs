# Overview

## What is SDSA?

SDSA (Spatial Data Search Assistant) is a server-based application designed to facilitate the development of an advanced search system for geospatial data. Leveraging cutting-edge Large Language Models (LLMs), it enables a search capability that surpasses conventional methods found in metadata catalogs. The application features four essential modules, each fulfilling distinct functions crucial for delivering high-quality retrieval:

- **Indexing Module**: Allows indexing metadata from external resources (e.g. metadata catalogues) into a local vector database enabling semantic search on indexed documents. The vector database can be considered as a search index.
- **Retrieval Module**: Enables contextual retrieval from the vector database using dense retrieval.
- **Search Criteria Module**: A sophisticated workflow to handle full-text inquiries in a conversational chat-bot (based on 🦜🕸️LangGraph).
- **Answer Generation Module**: Generates answers based on the user inputs (queries) and uses retrieved documents from the index as context to provide useful answers (using a RAG-approach).

These modules collectively empower SDSA to efficiently index, retrieve, apply search criteria, and generate accurate answers, thereby enhancing the overall search experience for geospatial data.


## Architecture
<img width="80%" height="80%" alt="Server_Architecture" src="https://github.com/simeonwetzel/sdsadocs/assets/90314129/7d2ec2c8-6ecd-4ee5-817c-d7d981003713">

