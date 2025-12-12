# Complete Beginner-Friendly Guide to RAG with Frontend (100% FREE)

## Table of Contents
1. [What is RAG? (Simple Explanation)](#what-is-rag-simple-explanation)
2. [Architecture Overview](#architecture-overview)
3. [Free Tools You'll Need](#free-tools-youll-need)
4. [Step-by-Step Methodology](#step-by-step-methodology)
5. [Phase 1: Setup Local LLM](#phase-1-setup-local-llm)
6. [Phase 2: Build Backend (RAG Pipeline)](#phase-2-build-backend-rag-pipeline)
7. [Phase 3: Create Frontend](#phase-3-create-frontend)
8. [Phase 4: Connect Everything](#phase-4-connect-everything)
9. [Testing & Troubleshooting](#testing--troubleshooting)
10. [Next Steps & Learning Resources](#next-steps--learning-resources)

---

## What is RAG? (Simple Explanation)

**RAG = Retrieval-Augmented Generation**

### Simple Analogy
Imagine you have a problem to solve:
- **Without RAG**: You ask ChatGPT a question → ChatGPT answers based on its training data only (might be outdated or wrong)
- **With RAG**: You give ChatGPT your documents first → ChatGPT reads them → ChatGPT answers based on YOUR documents + its knowledge

### Why RAG?
- ✅ Uses your private data (documents, PDFs, text files)
- ✅ Answers are accurate and based on YOUR information
- ✅ Completely FREE with open-source tools
- ✅ Works offline (no API calls needed)
- ✅ Privacy-friendly (data stays on your computer)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│              YOUR RAG SYSTEM                    │
├─────────────────────────────────────────────────┤
│                                                 │
│  FRONTEND (React/Vue)                          │
│  ├─ User Interface                             │
│  └─ Chat Input & Display                       │
│          ↓                                      │
│  BACKEND (Python + Flask/FastAPI)              │
│  ├─ Document Processing                        │
│  ├─ Vector Embeddings                          │
│  ├─ Similarity Search                          │
│  └─ LLM Prompt Assembly                        │
│          ↓                                      │
│  LOCAL LLM (Ollama/Llama.cpp)                  │
│  └─ Text Generation                            │
│          ↓                                      │
│  VECTOR DATABASE (Chroma/Qdrant)               │
│  └─ Document Storage & Retrieval               │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## Free Tools You'll Need

| Component | Free Tool | Why? |
|-----------|-----------|------|
| **Local LLM** | Ollama | Easy 1-click setup, runs on CPU |
| **Backend** | Python + Flask | Simple, beginner-friendly |
| **Vector DB** | Chroma | Lightweight, embedded, no setup |
| **Embeddings** | HuggingFace (Sentence-Transformers) | Free, open-source |
| **Frontend** | HTML + JavaScript (or React) | Simple & works everywhere |
| **File Format** | PDF/TXT | Easy to process |

**Total Cost: $0** ✅

---

## Step-by-Step Methodology

### Overview of 4 Phases

1. **Phase 1**: Download & run local LLM
2. **Phase 2**: Build backend (Python) with RAG pipeline
3. **Phase 3**: Create simple frontend (HTML + JavaScript)
4. **Phase 4**: Connect backend ↔ frontend

**Time Estimate**: 2-4 hours for first working version

---

## Phase 1: Setup Local LLM

### Step 1.1: Download Ollama

**What is Ollama?**
- Runs LLMs locally on your computer
- No GPU needed (works on CPU)
- Easiest setup for beginners

**Download:**
1. Go to [ollama.ai](https://ollama.ai)
2. Download for your OS (Windows/Mac/Linux)
3. Install it

### Step 1.2: Run a Local Model

Open terminal/command prompt and run:

```bash
ollama pull mistral
```

This downloads Mistral model (~4GB, might take 5-10 minutes)

### Step 1.3: Test the LLM

```bash
ollama run mistral
```

Try typing a question:
```
>>> Hello, how are you?
```

If it responds, ✅ you're good! Press Ctrl+C to exit.

### Step 1.4: Run Ollama in Server Mode

Keep Ollama running in background:

```bash
ollama serve
```

You'll see:
```
Listening on 127.0.0.1:11434
```

**Leave this terminal open** - your LLM is now accessible!

---

## Phase 2: Build Backend (RAG Pipeline)

### Step 2.1: Setup Python Project

Create a new folder:

```bash
mkdir my-rag-app
cd my-rag-app
python -m venv venv
source venv/bin/activate    # Mac/Linux
# or
venv\Scripts\activate       # Windows
```

### Step 2.2: Install Required Libraries

Create `requirements.txt`:

```
flask==2.3.3
flask-cors==4.0.0
requests==2.31.0
chromadb==0.4.5
sentence-transformers==2.2.2
PyPDF2==3.0.1
python-dotenv==1.0.0
```

Install them:

```bash
pip install -r requirements.txt
```

### Step 2.3: Create Document Processor

Create `document_processor.py`:

```python
import os
import PyPDF2
from typing import List

def extract_text_from_pdf(pdf_path: str) -> str:
    """Extract text from PDF file"""
    text = ""
    with open(pdf_path, 'rb') as file:
        pdf_reader = PyPDF2.PdfReader(file)
        for page in pdf_reader.pages:
            text += page.extract_text()
    return text

def extract_text_from_txt(txt_path: str) -> str:
    """Extract text from TXT file"""
    with open(txt_path, 'r', encoding='utf-8') as file:
        return file.read()

def chunk_text(text: str, chunk_size: int = 500, overlap: int = 100) -> List[str]:
    """
    Split text into overlapping chunks
    
    Example:
    - chunk_size=500 means each chunk is 500 characters
    - overlap=100 means 100 characters are repeated between chunks
    """
    chunks = []
    start = 0
    
    while start < len(text):
        end = start + chunk_size
        chunk = text[start:end]
        chunks.append(chunk.strip())
        start = end - overlap
    
    return [c for c in chunks if len(c) > 50]  # Remove very small chunks

def process_documents(doc_folder: str = "documents") -> str:
    """
    Process all documents in a folder
    
    Folder structure:
    documents/
    ├── file1.pdf
    ├── file2.txt
    └── file3.pdf
    """
    all_text = ""
    
    # Create folder if it doesn't exist
    os.makedirs(doc_folder, exist_ok=True)
    
    for filename in os.listdir(doc_folder):
        filepath = os.path.join(doc_folder, filename)
        
        if filename.endswith('.pdf'):
            print(f"Processing {filename}...")
            text = extract_text_from_pdf(filepath)
            all_text += "\n" + text
        
        elif filename.endswith('.txt'):
            print(f"Processing {filename}...")
            text = extract_text_from_txt(filepath)
            all_text += "\n" + text
    
    return all_text

if __name__ == "__main__":
    # Test the processor
    text = process_documents()
    chunks = chunk_text(text)
    print(f"Created {len(chunks)} chunks from documents")
```

### Step 2.4: Create Vector Database Handler

Create `vector_db.py`:

```python
import chromadb
from sentence_transformers import SentenceTransformer
from typing import List, Tuple

class VectorDatabase:
    def __init__(self, db_name: str = "rag_db"):
        """Initialize Chroma vector database"""
        # Initialize embedding model
        self.embedding_model = SentenceTransformer('all-MiniLM-L6-v2')
        
        # Initialize Chroma client
        self.client = chromadb.Client()
        
        # Create or get collection
        self.collection = self.client.get_or_create_collection(name=db_name)
        
        print(f"Vector DB initialized: {db_name}")
    
    def add_documents(self, chunks: List[str], metadata: dict = None):
        """
        Add document chunks to vector database
        
        Args:
            chunks: List of text chunks
            metadata: Optional metadata dict
        """
        if not chunks:
            print("No chunks to add")
            return
        
        # Generate embeddings for all chunks
        print(f"Generating embeddings for {len(chunks)} chunks...")
        embeddings = self.embedding_model.encode(chunks).tolist()
        
        # Add to Chroma
        self.collection.add(
            embeddings=embeddings,
            documents=chunks,
            ids=[f"chunk_{i}" for i in range(len(chunks))],
            metadatas=[metadata or {} for _ in chunks]
        )
        
        print(f"✅ Added {len(chunks)} chunks to vector DB")
    
    def search(self, query: str, num_results: int = 3) -> List[Tuple[str, float]]:
        """
        Search for similar documents
        
        Args:
            query: User's question
            num_results: Number of results to return
        
        Returns:
            List of (chunk_text, similarity_score)
        """
        # Embed the query
        query_embedding = self.embedding_model.encode([query]).tolist()
        
        # Search in Chroma
        results = self.collection.query(
            query_embeddings=query_embedding,
            n_results=num_results
        )
        
        # Format results
        documents = results['documents'][0]
        distances = results['distances'][0]
        
        # Convert distance to similarity (lower distance = higher similarity)
        similarities = [1 - d for d in distances]
        
        return list(zip(documents, similarities))
    
    def clear(self):
        """Clear all documents from database"""
        self.client.delete_collection(name=self.collection.name)
        print("Vector DB cleared")

if __name__ == "__main__":
    # Test vector database
    db = VectorDatabase()
    
    test_chunks = [
        "Python is a programming language",
        "Machine learning is a subset of AI",
        "RAG combines retrieval and generation"
    ]
    
    db.add_documents(test_chunks)
    results = db.search("What is AI?", num_results=2)
    
    for chunk, score in results:
        print(f"Score: {score:.2f} | {chunk[:50]}...")
```

### Step 2.5: Create LLM Handler

Create `llm_handler.py`:

```python
import requests
import json
from typing import List, Tuple

class OllamaLLM:
    def __init__(self, model: str = "mistral", base_url: str = "http://localhost:11434"):
        """
        Initialize Ollama LLM handler
        
        Args:
            model: Model name (e.g., "mistral", "neural-chat")
            base_url: Ollama server URL
        """
        self.model = model
        self.base_url = base_url
        self.api_endpoint = f"{base_url}/api/generate"
        
        # Test connection
        self.test_connection()
    
    def test_connection(self):
        """Check if Ollama server is running"""
        try:
            response = requests.get(f"{self.base_url}/api/tags", timeout=5)
            print(f"✅ Connected to Ollama at {self.base_url}")
        except requests.exceptions.ConnectionError:
            print(f"❌ ERROR: Cannot connect to Ollama at {self.base_url}")
            print("Make sure to run: ollama serve")
            raise
    
    def generate_response(
        self,
        prompt: str,
        context: List[str] = None,
        temperature: float = 0.7
    ) -> str:
        """
        Generate response from LLM with optional context
        
        Args:
            prompt: User's question
            context: List of relevant document chunks
            temperature: Creativity (0=deterministic, 1=creative)
        
        Returns:
            Generated response
        """
        # Build context string
        context_str = ""
        if context:
            context_str = "Based on the following documents:\n\n"
            for i, doc in enumerate(context, 1):
                context_str += f"[Document {i}]\n{doc}\n\n"
        
        # Build final prompt
        final_prompt = f"""{context_str}Question: {prompt}

Answer based on the documents above. If you can't find the answer, say "I don't know based on the provided documents."
"""
        
        try:
            # Send request to Ollama
            response = requests.post(
                self.api_endpoint,
                json={
                    "model": self.model,
                    "prompt": final_prompt,
                    "stream": False,
                    "temperature": temperature
                },
                timeout=120
            )
            
            response.raise_for_status()
            
            # Extract response
            result = response.json()
            return result['response'].strip()
        
        except requests.exceptions.Timeout:
            return "Error: Request timed out. The model might be processing a large prompt."
        except requests.exceptions.RequestException as e:
            return f"Error communicating with Ollama: {str(e)}"
    
    def get_available_models(self) -> List[str]:
        """Get list of available models"""
        try:
            response = requests.get(f"{self.base_url}/api/tags")
            data = response.json()
            return [m['name'] for m in data.get('models', [])]
        except:
            return []

if __name__ == "__main__":
    # Test LLM
    llm = OllamaLLM()
    
    response = llm.generate_response("What is machine learning?")
    print(f"Response:\n{response}")
```

### Step 2.6: Create RAG Pipeline (Main Logic)

Create `rag_pipeline.py`:

```python
from document_processor import process_documents, chunk_text
from vector_db import VectorDatabase
from llm_handler import OllamaLLM
from typing import List, Dict

class RAGPipeline:
    def __init__(self, model: str = "mistral"):
        """Initialize RAG pipeline"""
        self.vector_db = VectorDatabase()
        self.llm = OllamaLLM(model=model)
        self.initialized = False
    
    def initialize_with_documents(self, doc_folder: str = "documents"):
        """
        Initialize RAG with documents from a folder
        
        Steps:
        1. Extract text from PDFs/TXTs
        2. Split into chunks
        3. Create embeddings
        4. Store in vector database
        """
        print("\n📚 Processing documents...")
        
        # Extract text from all documents
        all_text = process_documents(doc_folder)
        
        if not all_text:
            print("⚠️  No documents found in 'documents' folder")
            return False
        
        # Split into chunks
        chunks = chunk_text(all_text, chunk_size=500, overlap=100)
        
        # Add to vector database
        self.vector_db.add_documents(chunks)
        
        self.initialized = True
        print("✅ RAG Pipeline ready!")
        return True
    
    def answer_question(self, question: str, num_context: int = 3) -> Dict:
        """
        Answer a question using RAG
        
        Returns:
            {
                'question': user's question,
                'answer': LLM's answer,
                'sources': list of relevant documents
            }
        """
        if not self.initialized:
            return {
                'question': question,
                'answer': "Please initialize RAG with documents first",
                'sources': []
            }
        
        # Step 1: Search for relevant documents
        print(f"\n🔍 Searching for relevant documents...")
        search_results = self.vector_db.search(question, num_results=num_context)
        
        context = [doc for doc, score in search_results]
        
        # Step 2: Generate answer using LLM with context
        print(f"🤖 Generating response...")
        answer = self.llm.generate_response(question, context=context)
        
        return {
            'question': question,
            'answer': answer,
            'sources': context
        }

if __name__ == "__main__":
    # Test RAG pipeline
    rag = RAGPipeline()
    
    # Initialize with documents
    rag.initialize_with_documents("documents")
    
    # Ask a question
    result = rag.answer_question("What is the main topic of these documents?")
    
    print("\n" + "="*50)
    print(f"Q: {result['question']}")
    print(f"A: {result['answer']}")
    print("\nSources:")
    for i, source in enumerate(result['sources'], 1):
        print(f"  {i}. {source[:100]}...")
```

### Step 2.7: Create Flask API

Create `app.py`:

```python
from flask import Flask, request, jsonify
from flask_cors import CORS
from rag_pipeline import RAGPipeline
import os

app = Flask(__name__)
CORS(app)  # Allow frontend to call backend

# Initialize RAG pipeline
rag = RAGPipeline(model="mistral")

@app.route('/api/initialize', methods=['POST'])
def initialize():
    """
    Initialize RAG with documents
    
    Request body: { "doc_folder": "documents" }
    """
    try:
        doc_folder = request.json.get('doc_folder', 'documents')
        
        success = rag.initialize_with_documents(doc_folder)
        
        return jsonify({
            'success': success,
            'message': 'RAG initialized successfully' if success else 'No documents found'
        })
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500

@app.route('/api/ask', methods=['POST'])
def ask_question():
    """
    Ask a question to the RAG system
    
    Request body: { "question": "Your question here" }
    """
    try:
        data = request.json
        question = data.get('question', '')
        
        if not question:
            return jsonify({'error': 'Question is required'}), 400
        
        # Get answer
        result = rag.answer_question(question)
        
        return jsonify(result)
    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/api/health', methods=['GET'])
def health():
    """Check if backend is running"""
    return jsonify({'status': 'ok', 'initialized': rag.initialized})

if __name__ == '__main__':
    print("\n" + "="*50)
    print("RAG Backend Starting...")
    print("="*50)
    print("1. Make sure 'ollama serve' is running in another terminal")
    print("2. Place your PDFs/TXTs in 'documents' folder")
    print("3. Backend running at http://localhost:5000")
    print("="*50 + "\n")
    
    # Create documents folder if it doesn't exist
    os.makedirs('documents', exist_ok=True)
    
    app.run(debug=True, port=5000)
```

---

## Phase 3: Create Frontend

Create `index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RAG Chatbot</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
            padding: 20px;
        }

        .container {
            width: 100%;
            max-width: 800px;
            background: white;
            border-radius: 12px;
            box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
            display: flex;
            flex-direction: column;
            height: 90vh;
        }

        .header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 20px;
            border-radius: 12px 12px 0 0;
        }

        .header h1 {
            font-size: 24px;
            margin-bottom: 5px;
        }

        .header p {
            font-size: 13px;
            opacity: 0.9;
        }

        .status {
            display: inline-block;
            padding: 4px 12px;
            border-radius: 20px;
            font-size: 12px;
            margin-top: 10px;
        }

        .status.ok {
            background: #10b981;
            color: white;
        }

        .status.error {
            background: #ef4444;
            color: white;
        }

        .chat-container {
            flex: 1;
            overflow-y: auto;
            padding: 20px;
            background: #f9fafb;
        }

        .message {
            margin-bottom: 15px;
            display: flex;
            animation: slideIn 0.3s ease;
        }

        @keyframes slideIn {
            from {
                opacity: 0;
                transform: translateY(10px);
            }
            to {
                opacity: 1;
                transform: translateY(0);
            }
        }

        .message.user {
            justify-content: flex-end;
        }

        .message-content {
            max-width: 70%;
            padding: 12px 16px;
            border-radius: 12px;
            line-height: 1.5;
            font-size: 14px;
        }

        .message.user .message-content {
            background: #667eea;
            color: white;
            border-radius: 12px 0 12px 12px;
        }

        .message.bot .message-content {
            background: white;
            color: #333;
            border: 1px solid #e5e7eb;
            border-radius: 0 12px 12px 12px;
        }

        .sources {
            background: #eff6ff;
            border-left: 3px solid #667eea;
            padding: 10px 12px;
            margin-top: 8px;
            border-radius: 4px;
            font-size: 12px;
            color: #1e40af;
        }

        .sources strong {
            display: block;
            margin-bottom: 4px;
        }

        .loading {
            display: flex;
            align-items: center;
            gap: 8px;
            font-size: 14px;
            color: #666;
        }

        .loading span {
            width: 8px;
            height: 8px;
            background: #667eea;
            border-radius: 50%;
            animation: bounce 1.4s infinite;
        }

        .loading span:nth-child(2) {
            animation-delay: 0.2s;
        }

        .loading span:nth-child(3) {
            animation-delay: 0.4s;
        }

        @keyframes bounce {
            0%, 80%, 100% {
                transform: scale(1);
                opacity: 0.5;
            }
            40% {
                transform: scale(1);
                opacity: 1;
            }
        }

        .input-area {
            padding: 20px;
            border-top: 1px solid #e5e7eb;
            display: flex;
            gap: 10px;
        }

        .input-group {
            flex: 1;
            display: flex;
            gap: 10px;
        }

        input[type="text"] {
            flex: 1;
            padding: 12px 16px;
            border: 1px solid #e5e7eb;
            border-radius: 8px;
            font-size: 14px;
            font-family: inherit;
        }

        input[type="text"]:focus {
            outline: none;
            border-color: #667eea;
            box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
        }

        button {
            padding: 12px 24px;
            background: #667eea;
            color: white;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-weight: 500;
            transition: background 0.3s;
        }

        button:hover {
            background: #5568d3;
        }

        button:disabled {
            background: #cbd5e1;
            cursor: not-allowed;
        }

        .init-section {
            padding: 20px;
            text-align: center;
            color: #666;
        }

        .init-section button {
            background: #10b981;
            margin-top: 15px;
        }

        .init-section button:hover {
            background: #059669;
        }

        .error-message {
            color: #ef4444;
            padding: 12px 16px;
            background: #fee2e2;
            border-radius: 8px;
            margin-bottom: 10px;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🤖 RAG Chatbot</h1>
            <p>Ask questions about your documents</p>
            <div class="status ok" id="status">● Checking connection...</div>
        </div>

        <div class="chat-container" id="chatContainer">
            <div class="init-section">
                <p>👋 Welcome! Click the button below to initialize RAG with your documents.</p>
                <p style="font-size: 12px; margin-top: 10px; color: #999;">
                    Make sure you have placed PDF or TXT files in the 'documents' folder
                </p>
                <button id="initBtn">Initialize RAG</button>
            </div>
        </div>

        <div class="input-area" id="inputArea" style="display: none;">
            <div class="input-group">
                <input
                    type="text"
                    id="questionInput"
                    placeholder="Ask a question about your documents..."
                    autocomplete="off"
                />
                <button id="sendBtn">Send</button>
            </div>
        </div>
    </div>

    <script>
        const API_URL = 'http://localhost:5000/api';
        const chatContainer = document.getElementById('chatContainer');
        const questionInput = document.getElementById('questionInput');
        const sendBtn = document.getElementById('sendBtn');
        const initBtn = document.getElementById('initBtn');
        const inputArea = document.getElementById('inputArea');
        const statusEl = document.getElementById('status');

        let ragInitialized = false;

        // Check backend connection on load
        window.addEventListener('load', checkConnection);

        async function checkConnection() {
            try {
                const response = await fetch(`${API_URL}/health`);
                if (response.ok) {
                    const data = await response.json();
                    updateStatus('ok', 'Connected to backend');
                    ragInitialized = data.initialized;
                } else {
                    updateStatus('error', 'Backend error');
                }
            } catch (error) {
                updateStatus('error', 'Backend not running');
            }
        }

        function updateStatus(status, message) {
            statusEl.className = `status ${status}`;
            statusEl.textContent = `● ${message}`;
        }

        // Initialize RAG
        initBtn.addEventListener('click', async () => {
            initBtn.disabled = true;
            initBtn.textContent = 'Initializing...';

            try {
                const response = await fetch(`${API_URL}/initialize`, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ doc_folder: 'documents' })
                });

                const data = await response.json();

                if (data.success) {
                    ragInitialized = true;
                    chatContainer.innerHTML = '';
                    inputArea.style.display = 'flex';
                    updateStatus('ok', 'RAG Initialized - Ready to answer questions');
                    addBotMessage('✅ RAG initialized! I\'m now ready to answer questions about your documents. Go ahead and ask me anything!');
                } else {
                    addBotMessage('❌ Initialization failed: ' + data.message);
                    updateStatus('error', 'No documents found');
                }
            } catch (error) {
                addBotMessage('❌ Error: ' + error.message);
                updateStatus('error', 'Connection failed');
            }

            initBtn.disabled = false;
            initBtn.textContent = 'Initialize RAG';
        });

        // Send question
        sendBtn.addEventListener('click', askQuestion);
        questionInput.addEventListener('keypress', (e) => {
            if (e.key === 'Enter' && !sendBtn.disabled) {
                askQuestion();
            }
        });

        async function askQuestion() {
            const question = questionInput.value.trim();
            if (!question) return;

            // Add user message
            addUserMessage(question);
            questionInput.value = '';
            sendBtn.disabled = true;

            // Show loading
            addBotMessage('<div class="loading"><span></span><span></span><span></span></div>');

            try {
                const response = await fetch(`${API_URL}/ask`, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ question })
                });

                const data = await response.json();

                // Remove loading message
                chatContainer.removeChild(chatContainer.lastChild);

                if (data.error) {
                    addBotMessage('❌ Error: ' + data.error);
                } else {
                    addBotMessage(data.answer, data.sources);
                }
            } catch (error) {
                chatContainer.removeChild(chatContainer.lastChild);
                addBotMessage('❌ Error: ' + error.message);
            }

            sendBtn.disabled = false;
            questionInput.focus();
        }

        function addUserMessage(text) {
            const div = document.createElement('div');
            div.className = 'message user';
            div.innerHTML = `<div class="message-content">${escapeHtml(text)}</div>`;
            chatContainer.appendChild(div);
            chatContainer.scrollTop = chatContainer.scrollHeight;
        }

        function addBotMessage(text, sources = []) {
            const div = document.createElement('div');
            div.className = 'message bot';
            let html = `<div class="message-content">${text}`;

            if (sources && sources.length > 0) {
                html += '<div class="sources"><strong>📚 Sources:</strong>';
                sources.forEach((source, i) => {
                    const preview = source.substring(0, 100) + (source.length > 100 ? '...' : '');
                    html += `<div>${i + 1}. ${escapeHtml(preview)}</div>`;
                });
                html += '</div>';
            }

            html += '</div>';
            div.innerHTML = html;
            chatContainer.appendChild(div);
            chatContainer.scrollTop = chatContainer.scrollHeight;
        }

        function escapeHtml(text) {
            const map = {
                '&': '&amp;',
                '<': '&lt;',
                '>': '&gt;',
                '"': '&quot;',
                "'": '&#039;'
            };
            return text.replace(/[&<>"']/g, m => map[m]);
        }
    </script>
</body>
</html>
```

---

## Phase 4: Connect Everything

### Step 4.1: Create Project Structure

Your folder should look like:

```
my-rag-app/
├── app.py
├── rag_pipeline.py
├── llm_handler.py
├── vector_db.py
├── document_processor.py
├── requirements.txt
├── index.html
├── documents/
│   ├── file1.pdf
│   ├── file2.txt
│   └── ...
└── venv/
```

### Step 4.2: Setup Documents Folder

1. Create a `documents` folder in your project
2. Add your PDFs or TXT files there
3. Make sure Ollama can process them (PDFs must be readable)

### Step 4.3: Run Everything

**Terminal 1 - Run Ollama:**
```bash
ollama serve
```

**Terminal 2 - Run Backend:**
```bash
cd my-rag-app
source venv/bin/activate  # Mac/Linux
# or
venv\Scripts\activate     # Windows

python app.py
```

**Terminal 3 - Run Frontend:**
```bash
cd my-rag-app
python -m http.server 8000
```

### Step 4.4: Access Your RAG System

Open browser and go to:
```
http://localhost:8000
```

Click "Initialize RAG" button, then start asking questions!

---

## Testing & Troubleshooting

### Problem: "Cannot connect to Ollama"

**Solution:**
```bash
# Make sure Ollama is running
ollama serve

# Check if it's accessible
curl http://localhost:11434/api/tags
```

### Problem: "No documents found"

**Solution:**
- Check if `documents` folder exists in your project
- Make sure PDFs are readable (not encrypted)
- Try with a simple `.txt` file first

### Problem: Very slow responses

**Solution:**
- Smaller model: `ollama pull neural-chat` (faster)
- Reduce chunk size in `document_processor.py`
- Use fewer context chunks in `rag_pipeline.py`

### Problem: Frontend won't connect to backend

**Solution:**
```bash
# Check if backend is running on port 5000
netstat -an | grep 5000

# Restart Flask
python app.py
```

---

## Next Steps & Learning Resources

### Improvements to Make

1. **Better UI**: Add markdown rendering, code highlighting
2. **Document Management**: Upload files directly from UI
3. **Chat Memory**: Keep conversation history
4. **Multiple Models**: Allow switching between Mistral, Neural Chat, etc.
5. **Fine-tuning**: Train the model on your specific data

### Learning Path

1. **Week 1**: Understand RAG basics (this guide)
2. **Week 2**: Experiment with different models (Llama2, Neural Chat)
3. **Week 3**: Add features (file upload, chat memory)
4. **Week 4**: Deploy locally or to cloud
5. **Month 2**: Learn fine-tuning and advanced retrieval techniques

### Free Resources

- **Ollama Docs**: [ollama.ai/docs](https://ollama.ai/docs)
- **LangChain Tutorials**: [python.langchain.com](https://python.langchain.com)
- **HuggingFace Embedding**: [huggingface.co/sentence-transformers](https://huggingface.co/sentence-transformers)
- **Chroma DB Docs**: [docs.trychroma.com](https://docs.trychroma.com)
- **YouTube**: Search "RAG tutorial 2024"

### Advanced Topics (After Mastering Basics)

- **LangChain**: Higher-level RAG framework
- **LlamaIndex**: Specialized for documents
- **Fine-tuning**: Train models on your data
- **Semantic Caching**: Reduce API calls
- **Multi-agent Systems**: Multiple specialized agents

---

## Summary

You now have a **complete RAG system** that:

✅ **Costs $0** (all free tools)
✅ **Works offline** (no API keys needed)
✅ **Respects privacy** (data stays on your computer)
✅ **Is beginner-friendly** (well-commented code)
✅ **Is extensible** (easy to add features)

**Total setup time**: 2-4 hours for first working version

**Next goal**: Add more documents and test with different questions!

---

## Quick Command Reference

```bash
# Start Ollama
ollama serve

# Start Backend
python app.py

# Start Frontend
python -m http.server 8000

# Download a new model
ollama pull llama2

# List available models
ollama list

# Run model in interactive mode
ollama run mistral
```

---

Good luck! You've got this! 🚀
