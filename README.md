# 📸 Facial Recognition Attendance System

A real-time attendance marking system using facial recognition via webcam. When a known face is detected, the person's name and timestamp are automatically logged to a CSV file — no manual input required.

---

## 📌 Overview

The system loads reference images of known individuals from a folder, encodes their facial features, and continuously compares them against live webcam frames. On a successful match, it draws a bounding box with the person's name on screen and marks their attendance in a CSV file (once per session).

---

## ✨ Features

- **Real-time face detection** — processes live webcam feed frame by frame
- **Face recognition** — matches detected faces against a known image database
- **Auto attendance logging** — writes name + timestamp to `Attendance.csv` on first match
- **Duplicate prevention** — each person is logged only once per session
- **Visual feedback** — green bounding box and name label rendered on the live feed
- **Lightweight processing** — frames downscaled to 25% before recognition for speed

---

## 🗂️ Project Structure

```
FacialRecognitionAttendance/
├── main.py                  # Core application
├── Attendance.csv           # Auto-generated attendance log
└── ImagesAttendance/        # Reference images (one per person)
    ├── John.jpg
    ├── Jane.png
    └── ...
```

> Image filename = person's name (e.g. `Praveen.jpg` → logged as `PRAVEEN`)

---

## ⚙️ Tech Stack

| Library | Purpose |
|---|---|
| `face_recognition` | Face encoding and matching (dlib-based) |
| `OpenCV (cv2)` | Webcam capture and frame rendering |
| `numpy` | Face distance calculation |
| `os` | Image directory loading |
| `datetime` | Timestamp generation |

---

## 🚀 Getting Started

### Prerequisites

- Python 3.7+
- Webcam
- CMake and dlib (required by `face_recognition`)

### Install Dependencies

```bash
pip install face_recognition opencv-python numpy
```

> On Windows, install CMake and Visual Studio Build Tools before `face_recognition`.
> On Linux: `sudo apt install cmake libboost-all-dev`

### Set Up Reference Images

1. Create a folder for known faces:
```
ImagesAttendance/
├── Alice.jpg
├── Bob.png
```
2. Use clear, front-facing photos — one image per person.
3. The filename (without extension) is used as the name in the attendance log.

### Configure Paths

In `main.py`, update the two hardcoded paths:

```python
# Line 5 — folder of known faces
path = r'path/to/ImagesAttendance'

# Inside markAttendance() — CSV output file
with open(r'path/to/Attendance.csv', 'r+') as f:
```

### Run

```bash
python main.py
```

Press **`Q`** to quit the webcam window.

---

## 📋 Attendance CSV Format

```
Name,Time
ALICE,09:15:32
BOB,09:17:45
```

Each person is logged once per run, even if they appear multiple times in the session.

---

## 🔄 How It Works

```
Load images from ImagesAttendance/
        │
        ▼
Encode facial features for each image
        │
        ▼
Open webcam → capture frame
        │
        ▼
Resize frame to 25% (speed optimisation)
        │
        ▼
Detect face locations in frame
        │
        ▼
Encode detected faces
        │
        ▼
Compare with known encodings (face distance)
        │
  Match found?
  ├── Yes → Draw box + name label
  │         Mark attendance in CSV (if not already logged)
  └── No  → Continue
        │
        ▼
Loop until 'Q' pressed
```

---

## ⚠️ Notes

- Ensure good lighting for accurate recognition.
- One clear, front-facing photo per person is sufficient for encoding.
- The CSV file must exist before running — create it manually if needed:
  ```
  Name,Time
  ```
- Recognition accuracy depends on `face_recognition` tolerance (default threshold used).

---

## 🔮 Future Enhancements

- [ ] GUI for managing known faces and viewing attendance
- [ ] Date column in CSV for multi-day tracking
- [ ] Database backend (MySQL/SQLite) instead of CSV
- [ ] Unknown face alerting
- [ ] Multiple encodings per person for improved accuracy
- [ ] Export attendance report as PDF

---

## 📄 License

This project is intended for academic and educational purposes.
