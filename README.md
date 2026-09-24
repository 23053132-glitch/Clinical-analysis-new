# AI Clinical Document Reviewer

A web app that takes clinical documentation — typed text, an image, or a PDF — and produces a structured clinical review. The application extracts and organizes relevant clinical information such as symptoms, diagnoses, medications, vitals, allergies, concerns, missing information, and possible inconsistencies.

Completed analyses are stored in MongoDB and can be reopened later from the history section.

> **All clinical data used in this project is synthetic. This application is intended for development and demonstration purposes only and is not a diagnostic tool.**

* Live app: `<ADD FRONTEND URL>`
* API: `<ADD BACKEND URL>` — health check: `/api/health`
* Note: the backend runs on a free hosting tier and may take up to a minute to wake up on the first request.

---

## Features

* Three input types:

  * Pasted clinical text
  * Image upload (`PNG`, `JPG`, `JPEG`, `WebP`)
  * PDF upload
* Typed PDFs are parsed directly using `pdf-parse`
* Scanned PDFs and images can be processed using a multimodal Ollama model
* Structured clinical report generation
* Short report summary followed by detailed clinical sections
* Extraction of:

  * Symptoms
  * Diagnoses
  * Medications
  * Vitals
  * Allergies
  * Concerns
  * Missing information
  * Possible inconsistencies
  * Items requiring further review
* Per-item confidence scores
* Verbatim source quote for extracted information
* Quote-grounding validation for documents with available text
* Automatic flagging and confidence reduction for unmatched quotes
* Detection of potentially inconsistent information
* Clear handling of:

  * Unreadable documents
  * Empty submissions
  * Unsupported files
  * Non-clinical documents
  * Processing failures
* Analysis history with:

  * Date
  * Status
  * Summary
  * Complete report details
* Loading and error states throughout the application
* Persistent storage using MongoDB Atlas
* Synthetic clinical data only

---

## Tech Stack

| Layer            | Technology            |
| ---------------- | --------------------- |
| Frontend         | React 18, Vite        |
| Frontend Hosting | Vercel                |
| Backend          | Node.js, Express      |
| File Uploads     | Multer                |
| Validation       | Zod                   |
| Database         | MongoDB Atlas         |
| ODM              | Mongoose              |
| PDF Processing   | pdf-parse             |
| AI               | Ollama                |
| AI Integration   | Ollama JavaScript SDK |
| AI Output        | Structured JSON       |
| Backend Hosting  | Render                |

---

## Architecture

```mermaid
flowchart LR
  U[User] --> FE[React Frontend<br/>Vercel]

  FE -- "REST + multipart/form-data" --> API[Express API<br/>Render]

  subgraph Backend
    API --> V[Validation<br/>size, type, empty input]

    V --> EX[Document Processing]

    EX --> TXT[Text Extraction]
    EX --> PDF[PDF Processing<br/>pdf-parse]
    EX --> IMG[Image / Scanned Document]

    TXT --> AI[AI Analysis<br/>Ollama]
    PDF --> AI
    IMG --> AI

    AI --> ZV[Zod Validation]
    ZV --> GR[Quote Grounding Check]

    GR --> DB[(MongoDB Atlas)]
  end

  AI <--> OL[Ollama<br/>Configured Model]

  DB --> API
  API --> FE
```

### Processing Flow

The application follows this general pipeline:

```text
User Input
    ↓
Request Validation
    ↓
File / Text Extraction
    ↓
PDF / Image Processing
    ↓
Ollama AI Analysis
    ↓
Structured JSON Response
    ↓
Zod Schema Validation
    ↓
Quote Grounding Check
    ↓
MongoDB Persistence
    ↓
Response to Frontend
    ↓
Clinical Report
```

If an error occurs after an analysis record has been created, the analysis is stored with a `failed` status so that the failure can still appear in the history.

---

# AI Processing

The application uses **Ollama** for clinical document analysis instead of Google Gemini.

The backend sends the extracted clinical content to the configured Ollama model and requests a structured JSON response matching the application's clinical report schema.

The generated response is then validated using Zod before it is stored.

### AI pipeline

```text
Document
   ↓
Text / Image extraction
   ↓
Ollama
   ↓
Structured JSON
   ↓
Zod validation
   ↓
Grounding verification
   ↓
MongoDB
```

The Ollama model can be configured through the backend environment variables.

---

## API

| Method | Path                | Description                                     |
| ------ | ------------------- | ----------------------------------------------- |
| `POST` | `/api/analyses`     | Submit clinical text or a document for analysis |
| `GET`  | `/api/analyses`     | Retrieve analysis history                       |
| `GET`  | `/api/analyses/:id` | Retrieve one complete analysis                  |
| `GET`  | `/api/health`       | Check API and database health                   |

### POST `/api/analyses`

The endpoint accepts `multipart/form-data`.

Supported input:

```text
text
```

or:

```text
file
```

The file can be a supported image or PDF.

### Successful response

```json
{
  "success": true,
  "data": {}
}
```

### Error response

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Error description"
  }
}
```

---

# Database

The application uses **MongoDB Atlas** with **Mongoose**.

The database name is specified directly in the MongoDB connection URI.

For example:

```env
MONGODB_URI="mongodb://username:password@host1:27017,host2:27017,host3:27017/clinicalreviewer?ssl=true&replicaSet=...&authSource=admin"
```

The database is:

```text
clinicalreviewer
```

Mongoose automatically creates the required collection when an analysis is stored.

No manual collection setup is required.

---

# Configuration

## Backend

Create:

```text
backend/.env
```

based on:

```text
backend/.env.example
```

Environment variables:

| Variable         | Purpose                                                             |
| ---------------- | ------------------------------------------------------------------- |
| `MONGODB_URI`    | MongoDB Atlas connection string including the database name         |
| `OLLAMA_API_KEY` | API key used to authenticate with the Ollama service, when required |
| `OLLAMA_MODEL`   | Ollama model used for document analysis                             |
| `CLIENT_URL`     | Allowed frontend origin(s), comma-separated                         |
| `PORT`           | Backend port, defaults to `5000`                                    |

Example:

```env
MONGODB_URI="mongodb://username:password@host1:27017,host2:27017,host3:27017/clinicalreviewer?ssl=true&replicaSet=...&authSource=admin"

OLLAMA_API_KEY="your-api-key"

OLLAMA_MODEL="your-model"

CLIENT_URL="http://localhost:5173"

PORT=5000
```

> Never commit `.env` files or API keys to GitHub.

---

## Frontend

Create:

```text
frontend/.env
```

Example:

```env
VITE_API_URL=http://localhost:5000
```

For production:

```env
VITE_API_URL=<BACKEND URL>
```

---

# Project Structure

```text
backend/
├── src/
│   ├── server.js
│   │   ├── Express application setup
│   │   ├── CORS
│   │   ├── routes
│   │   └── database connection
│   │
│   ├── config.js
│   │   └── Environment configuration
│   │
│   ├── routes/
│   │   └── analyses.js
│   │       ├── POST /api/analyses
│   │       ├── GET /api/analyses
│   │       └── GET /api/analyses/:id
│   │
│   ├── services/
│   │   ├── extract.js
│   │   │   ├── File-type detection
│   │   │   ├── PDF text extraction
│   │   │   └── Scan detection
│   │   │
│   │   ├── ollama.js
│   │   │   ├── Ollama client
│   │   │   ├── Prompt construction
│   │   │   └── AI response handling
│   │   │
│   │   ├── schema.js
│   │   │   └── Zod report schema
│   │   │
│   │   ├── ground.js
│   │   │   └── Quote-grounding validation
│   │   │
│   │   └── selftest.js
│   │       └── Offline schema / grounding checks
│   │
│   ├── models/
│   │   └── Analysis.js
│   │       └── Mongoose analysis model
│   │
│   └── middleware/
│       ├── upload.js
│       └── error.js
│
└── package.json


frontend/
├── src/
│   ├── App.jsx
│   ├── api.js
│   ├── sample.js
│   ├── styles.css
│   │
│   └── components/
│       ├── NewAnalysis.jsx
│       ├── History.jsx
│       └── ReportView.jsx
│
└── package.json
```

---

# Running Locally

## 1. Clone the repository

```bash
git clone <YOUR_REPOSITORY_URL>
cd <PROJECT_DIRECTORY>
```

---

## 2. Start the backend

```bash
cd backend
```

Install dependencies:

```bash
npm install
```

Create your environment file:

```bash
cp .env.example .env
```

Fill in:

```env
MONGODB_URI=...
OLLAMA_API_KEY=...
OLLAMA_MODEL=...
CLIENT_URL=http://localhost:5173
PORT=5000
```

Start the development server:

```bash
npm run dev
```

Backend:

```text
http://localhost:5000
```

Health check:

```text
http://localhost:5000/api/health
```

---

## 3. Start the frontend

Open another terminal:

```bash
cd frontend
```

Install dependencies:

```bash
npm install
```

Create the environment file:

```bash
cp .env.example .env
```

Set:

```env
VITE_API_URL=http://localhost:5000
```

Start the frontend:

```bash
npm run dev
```

Frontend:

```text
http://localhost:5173
```

---

# MongoDB Setup

1. Create a MongoDB Atlas cluster.
2. Create a database user.
3. Configure network access.
4. Copy the connection string.
5. Add the database name to the connection string.
6. Put the connection string in `MONGODB_URI`.

Example:

```text
mongodb://username:password@host1:27017,host2:27017,host3:27017/clinicalreviewer?... 
```

The application will create the required collection automatically when the first analysis is saved.

---

# Ollama Setup

The application uses Ollama as its AI provider.

Configure the Ollama credentials and model in the backend `.env` file:

```env
OLLAMA_API_KEY=your-api-key
OLLAMA_MODEL=your-model
```

The backend is responsible for:

* Sending document content to Ollama
* Providing the clinical-review prompt
* Requesting structured JSON output
* Handling AI errors
* Validating the generated response
* Retrying invalid structured responses when configured
* Passing the validated result to the grounding layer

The exact model can be changed without changing the frontend.

---

# Quote Grounding

A major part of the application is verifying that extracted information is supported by the original document.

For text-based documents, each extracted item can contain a source quote.

The grounding process checks whether that quote actually exists in the extracted document text.

Conceptually:

```text
AI extracted fact
       ↓
Source quote
       ↓
Search original document text
       ↓
   ┌───┴────┐
   ↓        ↓
 Found    Not Found
   ↓        ↓
 Valid    Flagged
          + confidence reduced
```

This helps reduce unsupported information being presented as if it were explicitly contained in the document.

For images and scanned PDFs without a usable text layer, quote grounding is limited because there may be no extracted source text to compare against.

---

# Error Handling

The application handles errors at multiple stages:

```text
Upload
  ↓
Validation
  ↓
Extraction
  ↓
AI Processing
  ↓
JSON Validation
  ↓
Grounding
  ↓
Database
```

Examples include:

* Unsupported file type
* File too large
* Empty text
* Invalid PDF
* Unreadable document
* Non-clinical document
* AI/API failure
* Invalid AI JSON
* Database failure
* Network errors

Failures are returned using a consistent API structure:

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message"
  }
}
```

---

# Deployment

## Backend — Render

Deploy the backend using Render.

Recommended configuration:

```text
Root Directory: backend
Build Command: npm install
Start Command: npm start
```

Configure the required environment variables:

```env
MONGODB_URI=...
OLLAMA_API_KEY=...
OLLAMA_MODEL=...
CLIENT_URL=<VERCEL FRONTEND URL>
PORT=...
```

The backend exposes:

```text
/api/health
```

for health monitoring.

> The backend uses a free hosting tier, so the first request after inactivity may take some time while the service wakes up.

---

## Frontend — Vercel

Deploy the `frontend` directory to Vercel.

Set:

```env
VITE_API_URL=<RENDER BACKEND URL>
```

The frontend communicates with the deployed Express API using this URL.

---

# Limitations

* Processing is synchronous, so very large documents may hit request or hosting timeouts.
* Free hosting may introduce cold-start delays.
* AI-generated clinical extraction can contain errors.
* Grounding validation is strongest when a usable text layer is available.
* For images and scanned PDFs, source-quote verification may be limited.
* Handwriting recognition depends on image quality and the capabilities of the configured multimodal model.
* No authentication is currently implemented.
* Anyone with access to the application can potentially view stored reports.
* **Only synthetic clinical data should be used with this demonstration application.**
* No rate limiting is currently implemented.
* Heavy usage may exhaust available AI/API resources.
* The system has not been formally evaluated against a large clinical benchmark.
* Results have been manually tested using synthetic clinical documents.
* This application is a **clinical document review aid**, not a diagnostic system.
* AI-generated information should not be treated as medical advice or as a substitute for qualified clinical judgment.

---

# Security & Privacy

This project is intended for demonstration and development using synthetic clinical information.

Do **not** upload:

* Real patient records
* Personally identifiable health information
* Hospital records
* Medical reports containing real patient information
* Any other sensitive clinical data

API keys and database credentials should always be stored in environment variables and should never be committed to the repository.

---

# Testing

The backend includes an offline test for the schema and grounding logic.

Run:

```bash
cd backend
npm run test:schema
```

This allows the schema and grounding functionality to be tested without making an AI request.

---

# Future Improvements

Potential future improvements include:

* User authentication
* Per-user report history
* Role-based access control
* Rate limiting
* Background document processing
* Larger-document support
* More robust OCR
* Additional document formats
* Formal evaluation datasets
* Improved medical terminology extraction
* Audit logging
* More detailed confidence calibration
* Streaming AI responses
* Improved monitoring and observability

---

## Disclaimer

This project is an experimental software application for clinical-document analysis.

All clinical examples and test documents used during development are synthetic.
