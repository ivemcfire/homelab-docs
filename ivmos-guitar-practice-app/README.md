# Guitar Practice App

A web application that helps you practice guitar by:
- Separating the guitar track from uploaded audio files
- Detecting the BPM and generating a matching metronome
- Allowing you to control the volume of the guitar and metronome independently

## Features

- **Audio Upload**: Upload MP3, WAV, OGG, FLAC, or M4A files
- **Guitar Separation**: Separates guitar frequencies from the rest of the track
- **BPM Detection**: Automatically detects the tempo of the song
- **Metronome Generation**: Creates a click track synchronized to the detected BPM
- **Volume Mixer**: Control volume of guitar, other instruments, and metronome separately
- **Quick Presets**: One-click presets for common practice scenarios

## Tech Stack

- **Frontend**: React + Vite + Tailwind CSS
- **Backend**: Python + FastAPI
- **Audio Processing**: librosa, numpy, scipy

## Setup

### Backend

1. Create a Python virtual environment:
```bash
cd backend
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

2. Install dependencies:
```bash
pip install -r requirements.txt
```

3. Run the backend server:
```bash
python main.py
```

The backend will start on http://localhost:8000

### Frontend

1. Install dependencies:
```bash
cd frontend
npm install
```

2. Run the development server:
```bash
npm run dev
```

The frontend will start on http://localhost:5173

## Usage

1. Open http://localhost:5173 in your browser
2. Upload an audio file (MP3, WAV, etc.)
3. Wait for processing (BPM detection, stem separation, metronome generation)
4. Use the volume sliders to adjust:
   - **Guitar**: Lower this to practice playing the guitar part yourself
   - **Other**: The rest of the instruments (bass, drums, vocals, etc.)
   - **Metronome**: Click track to keep time
5. Use quick presets for common scenarios

## Advanced: Better Stem Separation with Demucs

By default, the app uses a simple frequency-based separation which works well for most cases. For higher quality separation using AI (Demucs), set the environment variable before starting the backend:

```bash
USE_DEMUCS=true python main.py
```

Note: Demucs requires additional dependencies and processing time.

## API Endpoints

- `POST /api/upload` - Upload and process an audio file
- `GET /api/files/{file_id}` - Get information about a processed file
- `DELETE /api/files/{file_id}` - Delete a processed file
- `GET /api/health` - Health check
# ivmos-guitar-practice-app
