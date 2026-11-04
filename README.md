# Face Attendance Management System

An automated, privacy-first, biometric classroom attendance platform built with **Flask**, **InsightFace (SCRFD + ArcFace)**, **SQLite (WAL mode)**, **Fernet Biometric Encryption**, and **Google Sheets API**.

Designed for real-world educational institutions, the system solves the challenge of capturing attendance in large, high-density lecture halls from a single panoramic or wide-angle photograph—eliminating proxy attendance, manual roll calls, and identity fraud.

---

## 🌟 Key Features

### 1. Robust Biometric Enrollment (Zero Image Retention)
- **Multi-Angle Photo & Video Sampling**: Enrolls students using either 3–5 multi-angle photos or a 5-second video recording sampled at optimal frame intervals.
- **Biometric Quality Filters**: Analyzes image sharpness (Laplacian variance) and 3D head pose angles (yaw, pitch, roll) to reject blurred or heavily occluded samples.
- **Privacy by Design**: Raw face images and video files are processed strictly in memory and wiped immediately. Only 512-dimensional feature embeddings are retained.
- **Military-Grade Encryption**: Face feature vectors are serialized and encrypted using Fernet (AES-128-CBC + HMAC-SHA256) before storing in the database.

### 2. High-Density Classroom Recognition
- **Quadrant Tiling Engine**: Solves the small-face detection problem in high-resolution classroom images (4K / ultra-wide) by dividing the image into overlapping sub-regions alongside full-image scanning.
- **Non-Maximum Suppression (NMS)**: Merges overlapping detections across tiles using IoU thresholding to eliminate duplicate bounding boxes.
- **Sub-Second Matching**: Computes cosine similarity against enrolled student embeddings with a tuned threshold ($0.50$).

### 3. Teacher-in-the-Loop Review Workflow
- **Review Dashboard**: Automatically groups students into **Present** (high confidence) and **Absent** (unmatched).
- **Manual Status Override**: Teachers can inspect cropped face thumbnails, review detections, toggle attendance status, or add manual notes before finalizing.
- **Historical Records**: Full attendance session history with aggregated metrics (total enrolled, present count, attendance rate).

### 4. Automated Google Sheets Synchronization
- **Live Cloud Sync**: Seamlessly exports verified attendance sessions to institutional Google Sheets using Service Account authentication (`gspread`).
- **Dynamic Date Columns**: Detects existing dates or automatically generates new `DD-MM-YYYY` columns.
- **Non-Downgrade Safety Rule**: Prevents network retries or resubmissions from overwriting an existing **P** (Present) status with an **A** (Absent).

### 5. Security & Production Hardening
- **CSRF Protection**: Integrated across all state-changing endpoints with Flask-WTF.
- **Role-Based Access**: Session-based teacher authentication with Werkzeug password hashing.
- **Database Concurrency**: Configured SQLite with Write-Ahead Logging (WAL) and 30-second busy timeout to avoid locked database errors during external inspection.
- **Dockerized Ready**: Optimized multi-stage Docker build with OpenCV runtime libraries and Gunicorn WSGI.

---

## 🏗️ System Architecture

```mermaid
flowchart TD
    subgraph Enrollment ["1. Biometric Enrollment Pipeline"]
        A[Student Video / Multi-Angle Photos] --> B[SCRFD Face Detection & Quality Filter]
        B --> C[ArcFace 512-D Embedding Extraction]
        C --> D[L2 Normalization & Fernet Encryption]
        D --> E[(SQLite Database: Encrypted Embeddings)]
        A -.->|Discard Raw Media| Trash[Memory Flush / Privacy Zero-Retention]
    end

    subgraph Classroom ["2. Attendance Capture & Inference"]
        F[Classroom Group Image] --> G[Quadrant Tiler & Full-Scale Scan]
        G --> H[SCRFD Multi-Face Detection]
        H --> I[NMS Deduplication IoU 0.40]
        I --> J[ArcFace Feature Extraction]
        J --> K[Cosine Similarity Engine vs Decrypted Embeddings]
        E --> K
        K --> L[Attendance Session Draft Present / Absent]
    end

    subgraph Review ["3. Teacher Review & Cloud Sync"]
        L --> M[Teacher Web Dashboard]
        M -->|Confirm / Manual Overrides| N[Finalized Attendance Session]
        N --> O[(Local SQLite Database)]
        N --> P[Google Sheets Sync Engine API]
        P --> Q[Institutional Google Sheet]
    end
```

---

## 🛠️ Technology Stack

| Domain | Technology / Library | Purpose |
| :--- | :--- | :--- |
| **Backend Framework** | Flask 3.x, Flask-Login, Flask-WTF | Web server, session management, CSRF security |
| **Database & ORM** | SQLite (WAL mode), SQLAlchemy | Local relational database with concurrent read support |
| **Computer Vision** | OpenCV (`opencv-python-headless`) | Frame decoding, resizing, tiling, Laplacian blur detection |
| **Deep Learning** | InsightFace (`buffalo_l`), ONNX Runtime | SCRFD face detection & ArcFace 512-D embedding extraction |
| **Biometric Security** | Cryptography (`Fernet`) | Symmetrical AES-128-CBC encryption of face embeddings |
| **Cloud Integration** | `gspread`, `google-auth` | Automated synchronization with Google Sheets |
| **Containerization** | Docker, Docker Compose, Gunicorn | Production deployment and system dependency packaging |

---

## 📁 Repository Structure

```text
face-attendance/
├── app/
│   ├── attendance/           # Attendance capture, session review, and history routes
│   │   ├── routes.py
│   │   └── __init__.py
│   ├── auth/                 # Teacher authentication & session handling
│   │   ├── routes.py
│   │   └── __init__.py
│   ├── face_engine/          # Computer vision and deep learning core
│   │   ├── classroom.py      # Quadrant tiling, NMS deduplication, group recognition
│   │   ├── engine.py         # SCRFD detection, ArcFace embeddings, Fernet encryption
│   │   └── __init__.py
│   ├── google_sheets/        # Google Sheets API synchronization client
│   │   ├── sync.py           # Auto-headers, batch cell updates, non-downgrade logic
│   │   └── __init__.py
│   ├── students/             # Student roster management & biometric enrollment
│   │   ├── routes.py
│   │   └── __init__.py
│   ├── templates/            # Jinja2 HTML responsive UI templates
│   ├── extensions.py         # Shared SQLAlchemy, LoginManager, CSRF instances
│   ├── models.py             # User, Student, FaceEmbedding, AttendanceSession models
│   └── __init__.py           # Application factory with SQLite WAL configuration
├── instance/                 # Local database storage (attendance.db)
├── models/                   # Local InsightFace model checkpoints (buffalo_l)
├── config.py                 # Central configuration loader with python-dotenv
├── run.py                    # Application entrypoint & CLI commands
├── run_all_tests.py          # Master test suite runner
├── Dockerfile                # Docker container definition
├── docker-compose.yml        # Docker service orchestration
└── requirements.txt          # Python dependencies
```

---

## 🚀 Quick Start Guide

### Prerequisites
- Python 3.10+ (Python 3.12 recommended)
- `ffmpeg` or system OpenCV dependencies (for Linux: `libgl1`, `libglib2.0-0`, `libgomp1`)
- Git

### 1. Clone & Set Up Virtual Environment

```bash
git clone https://github.com/Praveen-ASML24026/Face-Recognition-Attendance.git
cd Face-Recognition-Attendance

# Create virtual environment
python -m venv venv

# Activate virtual environment
# Windows:
.\venv\Scripts\activate
# Linux/macOS:
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

### 2. Configure Environment Variables

Create a `.env` file in the project root (refer to `.env.example`):

```ini
FLASK_APP=run.py
FLASK_ENV=development
SECRET_KEY=generate-a-strong-secret-key-here
ENCRYPTION_KEY=generate-fernet-key-here
DATABASE_URL=sqlite:///instance/attendance.db

# Optional Google Sheets Integration
GOOGLE_SERVICE_ACCOUNT_JSON=instance/credentials.json
GOOGLE_SHEET_ID=your-google-sheet-id-here
```

To generate a valid `ENCRYPTION_KEY`:
```bash
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

### 3. Initialize Teacher Account & Run Server

```bash
# Create initial admin/teacher credentials
flask --app run.py create-teacher

# Run the local development server
python run.py
```
Open [http://127.0.0.1:5000](http://127.0.0.1:5000) in your browser.

---

## 🐳 Docker Deployment

Run the complete application in a self-contained container:

```bash
# Build and run with Docker Compose
docker-compose up --build -d

# View container logs
docker-compose logs -f
```
The application will be accessible at [http://localhost:5000](http://localhost:5000).

---

## 🧪 Testing & Verification

The project includes an end-to-end master test suite verifying all 8 operational phases:

```bash
python run_all_tests.py
```

### Verification Coverage:
1. **Phase 1: Foundation & Auth** — User models, password hashing, student directory CRUD.
2. **Phase 2: Face Engine** — SCRFD face detection, ArcFace 512-D vectors, cosine similarity, sharpness/pose evaluation.
3. **Phase 3: Biometric Enrollment** — Photo/video sampling, Fernet encryption integrity, zero image leakage.
4. **Phase 4: Classroom Tiling** — High-resolution quadrant tiling, IoU overlap calculation, NMS deduplication.
5. **Phase 5 & 6: Attendance Workflow** — Review interface, manual override state persistence, session history.
6. **Phase 7: Google Sheets API** — Header formatting, non-downgrade logic, batch updates.
7. **Phase 8: Security** — CSRF verification, unauthorized route protection, ciphertext encryption guarantees.

---

## 🔒 Privacy & Compliance

- **No Raw Biometric Storage**: Student face photos and videos are processed exclusively in volatile memory and destroyed once embeddings are extracted.
- **Encrypted Feature Vectors**: Mathematical face representations are stored as AES-128 encrypted blobs.
- **Audit Logging**: Every attendance session logs the teacher identity, timestamp, detection confidence scores, and manual review history.
