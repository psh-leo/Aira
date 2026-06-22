# DEVELOPMENT_GUIDE — Project Aira

> *Dokumen ini adalah panduan lengkap untuk menyiapkan environment dan mulai mengembangkan Aira dari nol.*

---

## Daftar Isi

1. [Prasyarat](#prasyarat)
2. [Setup Environment](#setup-environment)
3. [Instalasi Dependencies](#instalasi-dependencies)
4. [Setup Ollama](#setup-ollama)
5. [Verifikasi Instalasi](#verifikasi-instalasi)
6. [Menjalankan Aira](#menjalankan-aira)
7. [Struktur Project](#struktur-project)
8. [Workflow Pengembangan](#workflow-pengembangan)
9. [Troubleshooting](#troubleshooting)

---

## Prasyarat

Sebelum memulai, pastikan semua prasyarat berikut sudah terpasang.

### Python 3.11+

```bash
# Cek versi Python
python --version
# Output yang diharapkan: Python 3.11.x atau lebih baru
```

Jika Python belum terinstall atau versinya di bawah 3.11, download dari [python.org](https://python.org). Pastikan centang **"Add Python to PATH"** saat instalasi di Windows.

### Git

```bash
git --version
# Output: git version 2.x.x
```

Download dari [git-scm.com](https://git-scm.com) jika belum ada.

### ffmpeg

Whisper membutuhkan ffmpeg untuk memproses audio.

**Windows:**
```bash
# Opsi 1: Menggunakan winget (Windows Package Manager)
winget install ffmpeg

# Opsi 2: Download manual dari https://ffmpeg.org/download.html
# Extract dan tambahkan folder bin/ ke PATH
```

**Verifikasi:**
```bash
ffmpeg -version
# Harus menampilkan versi ffmpeg
```

### Ollama

Download dan install dari [ollama.com](https://ollama.com).

Setelah install, Ollama berjalan sebagai background service secara otomatis. Verifikasi:

```bash
ollama --version
# Output: ollama version x.x.x
```

---

## Setup Environment

### Clone Repository

```bash
git clone https://github.com/username/aira.git
cd aira
```

### Buat Virtual Environment

Virtual environment memastikan dependencies Aira tidak bertabrakan dengan package Python lain di sistemmu.

```bash
# Buat virtual environment
python -m venv .venv

# Aktivasi di Windows
.venv\Scripts\activate

# Setelah aktivasi, prompt akan berubah:
# (.venv) C:\Users\...\aira>
```

**Penting:** Selalu aktifkan virtual environment sebelum menjalankan perintah apapun yang berkaitan dengan Aira. Jika terminal ditutup dan dibuka lagi, jalankan perintah aktivasi ulang.

---

## Instalasi Dependencies

### Langkah 1: Install PyTorch CPU

**Ini harus dilakukan sebelum install Whisper.** Jika dilewati, pip akan menginstall PyTorch versi GPU yang ukurannya 2GB+ dan tidak diperlukan.

```bash
pip install torch --index-url https://download.pytorch.org/whl/cpu
```

Proses ini bisa memakan waktu beberapa menit tergantung kecepatan internet.

**Verifikasi:**
```bash
python -c "import torch; print(torch.__version__); print('CUDA:', torch.cuda.is_available())"
# CUDA harus False — kita tidak menggunakan GPU
```

### Langkah 2: Install Dependencies Utama

```bash
pip install -r requirements.txt
```

### Langkah 3: Install Development Dependencies

```bash
pip install -r requirements-dev.txt
```

### Estimasi Waktu dan Ukuran

| Package | Ukuran Download | Catatan |
|---|---|---|
| torch (CPU) | ~200MB | Sudah terinstall di langkah 1 |
| openai-whisper | ~10MB | Model di-download saat pertama kali digunakan |
| Whisper model `tiny` | ~74MB | Di-download otomatis saat pertama kali |
| pyaudio | ~1MB | Mungkin butuh PortAudio |
| pyttsx3 | ~1MB | Menggunakan SAPI5 bawaan Windows |
| httpx | ~1MB | |

**Total estimasi:** ~300–400MB untuk download awal.

### Jika pyaudio Gagal Install

```bash
# Coba dengan pipwin
pip install pipwin
pipwin install pyaudio
```

Jika masih gagal, pastikan Microsoft C++ Build Tools terinstall dari [visualstudio.microsoft.com](https://visualstudio.microsoft.com/visual-cpp-build-tools/).

---

## Setup Ollama

### Pull Model

```bash
# Pull Gemma 3 1B — model default Aira v0.1
ollama pull gemma3:1b
```

Ukuran model: ~800MB. Proses ini hanya dilakukan sekali.

**Verifikasi model tersedia:**
```bash
ollama list
# Harus menampilkan: gemma3:1b
```

### Verifikasi Ollama API

```bash
# Test bahwa Ollama API berjalan
curl http://localhost:11434/api/tags
# atau menggunakan PowerShell:
Invoke-WebRequest -Uri http://localhost:11434/api/tags
```

Jika mendapat respons JSON, Ollama berjalan dengan benar.

### Menjalankan Ollama Secara Manual

Jika Ollama tidak berjalan otomatis:

```bash
ollama serve
```

Biarkan terminal ini terbuka, buka terminal baru untuk menjalankan Aira.

---

## Verifikasi Instalasi

Jalankan script verifikasi berikut untuk memastikan semua komponen siap:

```bash
python -c "
import sys
print(f'Python: {sys.version}')

import torch
print(f'PyTorch: {torch.__version__} (CUDA: {torch.cuda.is_available()})')

import whisper
print(f'Whisper: OK')

import pyaudio
p = pyaudio.PyAudio()
print(f'PyAudio: OK ({p.get_device_count()} audio device(s))')
p.terminate()

import pyttsx3
engine = pyttsx3.init()
print(f'pyttsx3: OK')

import httpx
r = httpx.get('http://localhost:11434/api/tags')
print(f'Ollama: OK (status {r.status_code})')

print('')
print('Semua komponen siap.')
"
```

Output yang diharapkan:
```
Python: 3.11.x ...
PyTorch: 2.x.x+cpu (CUDA: False)
Whisper: OK
PyAudio: OK (1 audio device(s))
pyttsx3: OK
Ollama: OK (status 200)

Semua komponen siap.
```

---

## Menjalankan Aira

```bash
# Pastikan virtual environment aktif
# Pastikan Ollama berjalan

python main.py
```

Output startup yang diharapkan:
```
INFO: Initializing Aira...
INFO: Loading Whisper model...
INFO: Initializing TTS...
INFO: Connecting to Ollama...
INFO: Aira is ready.

Aira aktif. Tekan Ctrl+C untuk keluar.
```

**Untuk keluar:** Tekan `Ctrl+C`.

---

## Struktur Project

```
aira/
│
├── main.py                          # Entry point & dependency injection
│
├── aira/
│   ├── core/
│   │   ├── orchestrator.py          # Pengatur alur pipeline
│   │   ├── context_manager.py       # Conversation history
│   │   └── exceptions.py            # Custom exception hierarchy
│   │
│   ├── interfaces/
│   │   ├── llm_interface.py         # Protocol: LLM contract
│   │   ├── stt_interface.py         # Protocol: STT contract
│   │   └── tts_interface.py         # Protocol: TTS contract
│   │
│   ├── providers/
│   │   ├── llm/
│   │   │   └── ollama_provider.py   # Implementasi: Ollama
│   │   ├── stt/
│   │   │   └── whisper_provider.py  # Implementasi: Whisper
│   │   └── tts/
│   │       └── pyttsx3_provider.py  # Implementasi: pyttsx3
│   │
│   ├── pipeline/
│   │   └── voice_input.py           # Audio capture dari mikrofon
│   │
│   ├── personality/
│   │   └── master_prompt.py         # MASTER_PROMPT — identitas Aira
│   │
│   └── config/
│       └── settings.py              # Konfigurasi terpusat
│
├── tests/
│   ├── test_orchestrator.py
│   ├── test_context_manager.py
│   └── providers/
│       └── test_ollama_provider.py
│
├── docs/
│   ├── SYSTEM_DESIGN.md
│   ├── TECH_STACK.md
│   ├── MASTER_PROMPT.md
│   ├── DEVELOPMENT_GUIDE.md
│   └── adr/
│       └── ADR.md
│
├── temp/                            # File audio temporary (di-gitignore)
│
├── requirements.txt
├── requirements-dev.txt
├── mypy.ini
├── ruff.toml
├── .gitignore
│
├── README.md
├── VISION.md
├── ROADMAP.md
├── ARCHITECTURE.md
├── CONTRIBUTING.md
├── CHANGELOG.md
└── LICENSE
```

---

## Workflow Pengembangan

### Sebelum Mulai Coding

Setiap fitur baru mengikuti workflow yang sama, tanpa pengecualian:

```
1. Baca dokumen yang relevan (ARCHITECTURE, SYSTEM_DESIGN, ADR)
2. Pastikan kamu mengerti mengapa struktur yang ada seperti itu
3. Identifikasi komponen mana yang perlu diubah atau ditambah
4. Verifikasi bahwa perubahan tidak melanggar aturan di ARCHITECTURE
5. Baru mulai menulis kode
```

### Menjalankan Type Checker

```bash
mypy aira/ --strict
```

Jalankan ini sebelum commit. Jika ada error mypy, perbaiki dulu.

### Menjalankan Linter

```bash
# Check
ruff check aira/

# Auto-fix yang bisa diperbaiki otomatis
ruff check aira/ --fix

# Format
ruff format aira/
```

### Menjalankan Tests

```bash
# Semua test
pytest

# Dengan coverage
pytest --cov=aira --cov-report=term-missing

# Satu file spesifik
pytest tests/test_context_manager.py -v
```

### Sebelum Commit

```bash
# Checklist sebelum commit:
ruff check aira/          # Tidak ada linting error
ruff format aira/         # Kode sudah diformat
mypy aira/ --strict       # Tidak ada type error
pytest                    # Semua test pass
```

---

## Troubleshooting

### Ollama tidak bisa dihubungi

**Gejala:** `LLMConnectionError` atau `Connection refused` saat startup.

**Solusi:**
```bash
# Cek apakah Ollama berjalan
ollama list

# Jika tidak berjalan, jalankan manual
ollama serve
```

### pyaudio tidak mendeteksi mikrofon

**Gejala:** `AudioCaptureError` atau jumlah audio device = 0.

**Solusi:**
- Pastikan mikrofon terhubung dan diizinkan di Privacy Settings Windows
- Cek di Settings → Privacy & Security → Microphone
- Pastikan aplikasi Python diizinkan mengakses mikrofon

### Whisper model belum ter-download

**Gejala:** Download terjadi saat pertama kali `main.py` dijalankan, menyebabkan delay lama.

**Solusi:** Ini adalah perilaku normal saat pertama kali. Model ~74MB akan di-download dan di-cache. Startup berikutnya akan normal.

### pyttsx3 tidak mengeluarkan suara

**Gejala:** Tidak ada suara tapi tidak ada error.

**Solusi:**
- Cek volume sistem Windows
- Pastikan output audio device terpilih dengan benar di Sound Settings
- Test dengan: `python -c "import pyttsx3; e=pyttsx3.init(); e.say('test'); e.runAndWait()"`

### Import error setelah install

**Gejala:** `ModuleNotFoundError` meskipun sudah install.

**Solusi:**
```bash
# Pastikan virtual environment aktif (cek ada (.venv) di prompt)
.venv\Scripts\activate

# Verifikasi package terinstall di venv yang benar
pip list | findstr whisper
```

### mypy error pada Protocol

**Gejala:** mypy melaporkan bahwa Provider tidak memenuhi Interface.

**Solusi:** Pastikan signature method di Provider **persis** sama dengan yang didefinisikan di Interface — nama parameter, tipe, dan return type harus identik.

---

*Dokumen ini diperbarui setiap kali ada perubahan setup atau dependency.*

*Last updated: 2026 | Version: 1.0.0*
