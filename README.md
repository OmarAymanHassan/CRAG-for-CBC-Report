# MultiIndexRAG: Corrective RAG for Medical CBC Reports

## Table of Contents
1. [Overview](#overview)
2. [RAG Pipeline](#rag-pipeline)
3. [Vectorstore Contents](#vectorstore-contents)
4. [Features](#features)
5. [End-to-End Pipeline](#end-to-end-pipeline)
6. [How It Works](#how-it-works)
7. [Project Structure](#project-structure)
8. [Example Workflow](#example-workflow)
9. [Notable Design Choices](#notable-design-choices)
10. [Key Technologies](#key-technologies)
11. [Customization](#customization)
12. [Project Presentation](#project-presentation)
13. [Credits](#credits)
14. [License](#license)
15. [Contact](#contact)

## Overview

**MultiIndexRAG** is an advanced Retrieval-Augmented Generation (RAG) pipeline designed for the medical domain, specifically for analyzing and interpreting Complete Blood Count (CBC) reports. The system leverages multi-vector summarization, corrective retrieval, and LLM-based reasoning to provide accurate, context-aware answers to clinical questions about patient CBCs, using both local medical literature and web search as fallback.

---

## RAG Pipeline

![RAG Pipeline](./RAG%20Pipeline.png)
*Figure: The full LangGraph and RAG pipeline, from CBC report ingestion to answer generation.*

---

## Vectorstore Contents

The vectorstore in this project contains:
- **Medical recommendations and guidelines** for how to treat CBC abnormalities, based on authoritative sources and clinical best practices.
- **Detailed information about anemia types** and related blood disorders, including diagnostic criteria, management strategies, and clinical insights.
- **Contextual explanations** to help interpret abnormal CBC findings and guide next steps in patient care.

This ensures that, when a CBC report is processed, the system can retrieve not only relevant guidelines for treatment but also in-depth information about the specific type of anemia or abnormality detected.

---

## Features

- **MultiVectorStore Summarization**:  
  Each medical document (PDFs, web articles) is split into chunks and summarized using an LLM. These summaries are embedded and stored in a vector database (Chroma/FAISS), enabling efficient semantic retrieval.
- **Patient Report Summarization**:  
  When a new CBC report is uploaded, it is parsed, structured, and summarized by an LLM to extract abnormalities and suggest possible diagnoses (e.g., "Microcytic Hypochromic Anemia").
- **Semantic Retrieval via MultiVectorStore**:  
  At query time, the system compares the summary of the patient report to the stored document summaries. If a match is found (using vector similarity), the system retrieves the **original full document(s)** corresponding to the most relevant summaries, not just the summary chunk.
- **Corrective RAG with Grading and Web Search**:  
  After retrieval, a grading LLM checks if the majority of retrieved documents are relevant to the clinical question (majority voting).  
  - If **relevant**, the answer is generated from the retrieved local documents.  
  - If **not relevant**, the system automatically reformulates the query and performs a web search, retrieving and incorporating up-to-date external information before generating the answer.

---

## End-to-End Pipeline

The workflow is orchestrated as a LangGraph state machine, handling:

1. CBC report detection and parsing
2. The main parameters of the CBC report are added to the database (JSON/CSV).
3. The report is summarized and a diagnosis is suggested
4. Retrieval and grading
5. Corrective web search if needed
6. Final answer generation

---

## How It Works

### 1. Document Ingestion & Summarization

- All medical PDFs in the `RAG/` folder are loaded and split into token-based chunks.
- Each chunk is summarized using an LLM (e.g., Llama-3-70B or Gemini).
- Summaries are embedded and stored in a vector database (Chroma/FAISS).
- The original full documents are stored in a byte store, linked by unique IDs.

### 2. CBC Report Processing

- A new CBC report (PDF) is uploaded.
- The system checks if the file is a valid CBC report (using an LLM classifier).
- The report is parsed and structured (patient info, CBC panel, comments).
- The CBC values are summarized, and abnormalities are identified (e.g., low Hb, low MCV).
- A concise, retrieval-friendly query is generated from the summary (e.g., "Microcytic Hypochromic Anemia").
- **The main parameters of the CBC report are saved in a JSON file, and then stored in a database as a `.csv` file for further analysis and record-keeping.**

### 3. Retrieval & Grading

- The system retrieves the most relevant document summaries from the vector store using the patient report summary/query.
- The corresponding full documents are fetched from the byte store.
- Each retrieved document is graded for relevance to the clinical question using an LLM.
- Majority voting determines if the retrieval is sufficient.

### 4. Corrective RAG (Web Search Fallback)

- If the majority of retrieved documents are **not relevant**:
  - The system reformulates the query using an LLM.
  - A web search is performed (e.g., via Tavily).
  - Web results are appended to the document set for answer generation.

### 5. Answer Generation

- The final answer is generated using an LLM, grounded in the retrieved (and, if needed, web-searched) documents.
- The answer is context-aware, medically accurate, and references the most relevant sources.

---

## Project Structure

```
.
├── MultiIndexRAG.ipynb      # Main notebook (all code, pipeline, and experiments)
├── RAG/                     # Folder with medical PDFs (knowledge base)
├── cbc_report.pdf           # Example CBC report for testing
├── cbc_output.json          # Output: structured CBC reports (main parameters saved here)
├── cbc_reports.csv          # Output: flattened CBC reports for analysis (database)
├── requirements.txt         # Python dependencies
├── Presentation.pptx        # Project presentation (overview, pipeline, features)
├── RAG Pipeline.png         # Visual pipeline diagram
└── README.md                # (this file)
```

---

## Example Workflow

1. **Add new medical PDFs** to the `RAG/` folder.
2. **Run the notebook** to build the multi-vector index and summaries.
3. **Upload a CBC report PDF**.
4. The system:
   - Parses and summarizes the report.
   - Suggests a likely diagnosis (e.g., "Microcytic Hypochromic Anemia").
   - Retrieves and grades relevant medical documents.
   - If needed, performs a corrective web search.
   - Generates a final, referenced answer.

---

## Notable Design Choices

- **MultiVectorStore**:  
  Enables fine-grained, chunk-level semantic search, but always returns the full parent document for answer generation, ensuring rich context.
- **Summary-to-Summary Matching**:  
  Retrieval is based on comparing the LLM-generated summary of the patient report to the LLM-generated summaries of the knowledge base, improving precision.
- **Corrective RAG**:  
  If the system's retrieval is insufficient (as judged by LLM majority voting), it automatically falls back to web search, reformulates the query, and tries again—ensuring robust, up-to-date answers.
- **LLM-Driven Grading and Query Rewriting**:  
  All critical steps (grading, summarization, query rewriting) are handled by LLMs for maximum flexibility and domain adaptation.

---

## Key Technologies

- **LangChain / LangGraph**: Orchestration, state management, and chaining.
- **LLMs**: Gemini, Llama-3-70B, etc. for summarization, grading, and generation.
- **Chroma/FAISS**: Vector database for semantic search.
- **MultiVectorRetriever**: Links summary chunks to full documents for context-rich retrieval.
- **PyPDFLoader**: PDF parsing.
- **TavilySearch**: Web search fallback for corrective RAG.

---

## Customization

- **Knowledge Base**: Add or remove PDFs in the `RAG/` folder.
- **LLM Models**: Swap out LLMs (Gemini, Llama, etc.) as needed.
- **Chunking/Summarization**: Adjust chunk size or summarization prompts for your domain.
- **Web Search**: Configure or replace the web search tool as needed.

---

## Project Presentation

A project presentation is available as **Presentation.pptx** in this repository. This presentation provides a concise overview of the pipeline, features, and usage of the CRAG for CBC Report project. It is ideal for quick understanding, sharing with collaborators, or presenting the workflow and results to others.

---

## Credits

- Built with [LangChain](https://github.com/langchain-ai/langchain), [Chroma](https://www.trychroma.com/), [FAISS](https://github.com/facebookresearch/faiss), [PyPDFLoader](https://github.com/langchain-ai/langchain), and [Tavily](https://www.tavily.com/).
- LLMs: Gemini, Llama-3-70B, etc.

---

## License

This project is for research and educational purposes. Please ensure compliance with all data privacy and medical information regulations when using real patient data.

---

## Contact

For questions or collaboration, please open an issue or contact the project maintainer.

