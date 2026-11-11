# CloudOps Sentinel – Enterprise Incident Response Self-RAG Copilot

An AI-powered Incident Response Copilot built using **Self-RAG, intelligent agents, and cloud integrations** to automate incident investigation, troubleshooting, and operational knowledge retrieval.

CloudOps Sentinel is designed as a production-style Self-RAG system for cloud operations and incident response. It retrieves private operational knowledge from Pinecone first, evaluates the quality of retrieved evidence, and corrects weak retrieval or generation before producing an answer.

When private knowledge is insufficient, it uses Tavily as a controlled internet-search fallback.

---

## 1. Business Problem

Cloud operations teams regularly deal with incidents such as:

- API failures and service outages
- High CPU and memory utilization
- Deployment failures
- Database connectivity issues
- Payment and checkout failures
- Infrastructure configuration problems

The information required to troubleshoot these incidents is often spread across runbooks, standard operating procedures (SOPs), postmortems, and internal documentation.

CloudOps Sentinel addresses this problem by providing an AI-powered assistant that retrieves relevant operational knowledge and guides engineers through incident investigation.

---

## 2. Key Features

- **Self-RAG Architecture:** Evaluates retrieved documents and generated responses to improve answer quality.
- **Private Knowledge Base First:** Searches internal operational documentation stored in Pinecone before using external sources.
- **Intelligent Query Rewriting:** Rewrites weak queries to improve retrieval results.
- **Internet Search Fallback:** Uses Tavily only when private knowledge is insufficient.
- **LangGraph Workflow:** Implements conditional routing, retrieval grading, answer generation, and response evaluation.
- **Persistent Conversation Memory:** Uses LangGraph SQLite persistence to retain incident context across follow-up questions.
- **Context-Aware Follow-ups:** Understands follow-up questions using the previous incident conversation.
- **Runbook Vault:** Supports uploading operational documents into the knowledge base.
- **Incident Command Center UI:** Provides a web interface for incident investigation and troubleshooting.

---

## 3. Architecture

### High-Level Architecture

```text
                         USER
                           |
                           v
                  HTML / CSS / JS
                           |
                           v
                       FastAPI
                           |
                           v
                    LangGraph Agent
                           |
                           v
                 SQLite Conversation Memory
                           |
                           v
                  Contextualize Follow-up
                           |
                           v
                    Decide Retrieval
                           |
                           v
                 Private Pinecone Knowledge Base
                           |
                           v
                 Grade Retrieved Documents
                           |
                  +--------+--------+
                  |                 |
             Relevant          Weak / Missing
                  |                 |
                  v                 v
             Generate          Rewrite Query
                  |                 |
                  v                 v
               IsSUP?         Retry Private KB
                  |                 |
             +----+----+            v
             |         |      Internet Search
            Yes        No       (Tavily)
             |         |            |
             v         v            v
          IsUSE?     Revise    Grade Web Evidence
             |                      |
        +----+----+                 v
        |         |              Generate
       Yes        No                |
        |         |                 v
        v         v              Final Answer
    Final Answer  Revise
```

### Self-RAG Workflow

1. The user submits an incident-related question.
2. LangGraph retrieves the existing conversation state using the session's `thread_id`.
3. The contextualization node converts follow-up questions into standalone questions using previous incident context.
4. The workflow decides whether knowledge retrieval is required.
5. Pinecone retrieves relevant documents from the private operational knowledge base.
6. Retrieved documents are evaluated for relevance.
7. If the private evidence is relevant, Gemini generates an answer grounded in the retrieved documents.
8. The generated answer is evaluated using:
   - **IsSUP:** Checks whether the generated answer is supported by the retrieved evidence.
   - **IsUSE:** Evaluates whether the answer is useful for the user's question.
9. If the answer fails evaluation, the workflow revises it.
10. If private evidence is weak or missing, the query is rewritten and private retrieval is retried.
11. If private retrieval remains insufficient, Tavily searches external sources.
12. Web evidence is evaluated before generating the final response.

---

## 4. Technology Stack

| Technology               | Purpose                                                                   |
| ------------------------ | ------------------------------------------------------------------------- |
| LangGraph                | Self-RAG workflow, conditional routing, and persistent thread state       |
| Google Gemini 2.5 Flash  | Query rewriting, response generation, relevance grading, IsSUP, and IsUSE |
| Google Gemini Embeddings | Generating embeddings for private operational documents                   |
| Pinecone                 | Vector database for private runbooks, SOPs, and postmortems               |
| Tavily                   | Controlled internet-search fallback                                       |
| SQLite                   | LangGraph persistence memory and simple audit database for the demo       |
| FastAPI                  | Application and API backend                                               |
| HTML, CSS, JavaScript    | Incident command center UI with document upload                           |
| Docker                   | Application deployment packaging                                          |

---

## 5. Project Structure

```text
CloudOps-Sentinel-Enterprise-Incident-Response-Self-RAG-Copilot/
│
├── app.py
│
├── data_ingestion.py          # ONE file to build the Pinecone KB
│
├── src/
│   ├── config.py
│   ├── db.py
│   ├── ingestion.py
│   ├── models.py
│   ├── self_rag.py
│   └── vectorstore.py
│
├── documents/
│   ├── checkout-api-runbook.md
│   ├── payments-high-cpu-runbook.md
│   └── deployment-rollback-sop.md
│
├── templates/
│   └── index.html
│
├── static/
│   ├── styles.css
│   └── app.js
│
├── uploads/
│
├── data/
│
├── requirements.txt
│
├── Dockerfile
│
├── docker-compose.yml
│
└── .env.example
```

---

## 6. Setup and Installation

### Step 1: Clone the Repository

```bash
git clone <YOUR_REPOSITORY_URL>
cd CloudOps-Sentinel-Enterprise-Incident-Response-Self-RAG-Copilot
```

### Step 2: Create a Virtual Environment

```bash
python -m venv venv
```

Activate the virtual environment.

**Windows:**

```bash
venv\Scripts\activate
```

**macOS / Linux:**

```bash
source venv/bin/activate
```

### Step 3: Install Dependencies

```bash
pip install -r requirements.txt
```

### Step 4: Configure Environment Variables

Create a `.env` file by copying `.env.example`.

Configure the following environment variables:

```env
# Google Gemini API
GEMINI_API_KEY=your_gemini_api_key
GEMINI_MODEL=gemini-2.5-flash

# Gemini Embeddings
EMBEDDING_MODEL=models/gemini-embedding-001

# Pinecone
PINECONE_API_KEY=your_pinecone_api_key
PINECONE_INDEX_NAME=cloudops-sentinel-gemini-self-rag
PINECONE_NAMESPACE=incident-runbooks

# Tavily
TAVILY_API_KEY=your_tavily_api_key
```

**Note:** Keep your actual `.env` file private. Never commit API keys or other secrets to GitHub.

---

## 7. Build the Pinecone Knowledge Base

CloudOps Sentinel uses a single ingestion script to build the private operational knowledge base.

Run:

```bash
python data_ingestion.py
```

The ingestion process:

1. Loads environment variables from `.env`.
2. Creates or validates the configured Pinecone index.
3. Loads operational documents from the `./documents` directory.
4. Splits documents into smaller chunks.
5. Generates embeddings using the configured Google Gemini embedding model.
6. Stores the generated vectors in Pinecone.
7. Uses the configured namespace: `incident-runbooks`.

### Supported Documents

- PDF
- TXT
- Markdown
- DOCX

Stable chunk IDs ensure that rerunning ingestion updates matching vectors instead of creating random duplicate IDs.

After successful ingestion, the Pinecone knowledge base is ready for incident-related queries.

---

## 8. Run the Application

Start the FastAPI application:

```bash
python app.py
```

Alternatively:

```bash
uvicorn app:app --reload
```

Open the application in your browser:

```text
http://127.0.0.1:8000
```

The Incident Command Center UI allows users to submit incidents, ask follow-up questions, and retrieve troubleshooting guidance.

---

## 9. Upload New Operational Documents

CloudOps Sentinel includes a **Runbook Vault** that allows users to upload new operational documents through the UI.

Supported formats:

- PDF
- TXT
- Markdown
- DOCX

The **Add to Knowledge Base** action uses the same ingestion layer and embedding model as `data_ingestion.py`.

### Document Upload Pipeline

```text
User Upload
    |
    v
FastAPI /api/upload
    |
    v
Document Loader
    |
    v
Text Chunking
    |
    v
Gemini Embeddings
    |
    v
Existing Pinecone Index
    |
    v
Configured Namespace
    |
    v
Knowledge Base Updated
```

Newly uploaded documents become available for subsequent incident-related queries.

---

## 10. LangGraph SQLite Persistence Memory

CloudOps Sentinel uses LangGraph SQLite persistence to maintain conversation context across multiple user messages.

### How It Works

- Each browser incident session has a persistent `thread_id`.
- LangGraph stores conversation state in SQLite.
- The checkpoint database is located at:

```text
data/langgraph_memory.sqlite
```

- When a user asks a follow-up question, the application retrieves the previous state using the same `thread_id`.
- The contextualization node uses the previous incident conversation to convert the follow-up into a standalone operational question.
- The Self-RAG workflow then processes the contextualized question.

### Example

**Initial Question:**

```text
Why is the checkout API returning 502 errors after deployment?
```

**Follow-up Question:**

```text
What should I check next if that does not work?
```

The system uses the previous incident context to understand what "that" refers to and continues the troubleshooting process.

SQLite is used for local/demo persistence. For production deployment, PostgreSQL or another production-grade LangGraph checkpointer/database can be considered.

---

## 11. Example Usage Scenarios

### Demo A: Private Knowledge Base Retrieval

Ask:

```text
The checkout API is returning 502 errors after deployment. How should I investigate?
```

Expected behavior:

- Retrieves the checkout API runbook from Pinecone.
- Grades the retrieved documents.
- Generates a troubleshooting response grounded in private operational documentation.

### Demo B: Context-Aware Follow-up

After the first question, ask:

```text
What should I check next if that does not work?
```

Expected behavior:

- Reuses the same `thread_id`.
- Retrieves the previous incident context.
- Understands the follow-up question.
- Continues troubleshooting without requiring the user to repeat the original incident.

### Demo C: Upload New Operational Knowledge

- Open the Runbook Vault.
- Upload a new operational document.
- Click **Add to Knowledge Base**.
- Verify the ingestion status and chunk count.
- Ask a question related to the uploaded document.

Expected behavior:

- Processes the uploaded document.
- Generates Gemini embeddings.
- Stores vectors in the existing Pinecone index and namespace.
- Makes the new knowledge available for retrieval.

### Demo D: Internet Search Fallback

Ask a question about an issue that is not covered by the private runbooks.

Expected behavior:

- Searches the private Pinecone knowledge base first.
- Grades the retrieved evidence.
- Rewrites the query and retries private retrieval if needed.
- Uses Tavily only when private knowledge remains insufficient.
- Grades web evidence before generating the final response.

---

## 12. Important Configuration Rule

Both the application and ingestion script must use the same shared configuration from:

```text
src/vectorstore.py
```

Ensure that both use the same:

- Gemini embedding model
- Embedding output dimension
- Pinecone index name
- Pinecone namespace

**Important:** The Pinecone index dimension must match the output dimension of the Gemini embedding model. If you change the embedding model or its output dimension, create or configure a compatible Pinecone index before ingesting documents.

The application and ingestion pipeline must use the same embedding configuration to ensure that document retrieval works correctly.

---

## 13. Environment and Security

- Store API keys in `.env`.
- Do not expose Gemini, Pinecone, or Tavily API keys in frontend JavaScript.
- Keep `.env` out of version control.
- Use `.env.example` to document the required environment variables without including real credentials.
- Configure production secrets using a secure secrets-management solution.

---

## 14. Future Enhancements

- Integrate cloud monitoring platforms.
- Add automated incident detection.
- Integrate with service-management platforms.
- Add incident severity classification.
- Implement role-based access control.
- Introduce production-grade persistent storage.
- Add incident analytics and reporting dashboards.

---

**CloudOps Sentinel – Enterprise Incident Response Self-RAG Copilot**

_An intelligent incident response assistant that prioritizes private operational knowledge, evaluates evidence, and provides context-aware troubleshooting guidance._
