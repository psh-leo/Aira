# TECH_STACK — Project Aira

> *"Setiap teknologi di sini dipilih karena alasan yang terdokumentasi, bukan karena populer."*

---

## Daftar Isi

1. [Gambaran Umum](#gambaran-umum)
2. [Runtime & Bahasa](#runtime--bahasa)
3. [Core Pipeline](#core-pipeline)
4. [LLM Layer](#llm-layer)
5. [Audio Layer](#audio-layer)
6. [Development Tools](#development-tools)
7. [Testing Stack](#testing-stack)
8. [Dependency Map](#dependency-map)
9. [Versi & Kompatibilitas](#versi--kompatibilitas)
10. [Upgrade Path](#upgrade-path)

---

## Gambaran Umum

Aira v0.1 dibangun di atas tiga prinsip pemilihan teknologi:

**Offline-first** — Semua komponen inti berjalan lokal tanpa koneksi internet. Tidak ada API key yang dibutuhkan untuk menjalankan Aira.

**RAM-conscious** — Setiap teknologi dipilih dengan mempertimbangkan total memory footprint di perangkat dengan RAM 4GB.

**Replaceable** — Tidak ada teknologi yang tidak bisa diganti. Abstraction layer memastikan setiap komponen bisa diswap tanpa merombak sistem.

---

## Runtime & Bahasa

### Python 3.11+

**Peran:** Bahasa utama seluruh sistem Aira.

**Versi minimum:** 3.11
**Versi yang direkomendasikan:** 3.11.x (LTS, stabil)

**Mengapa 3.11, bukan versi lebih baru?**

Python 3.11 adalah titik keseimbangan terbaik saat ini: performa signifikan lebih baik dari 3.10 (up to 60% lebih cepat di beberapa benchmark), ekosistem library AI/ML sudah mature dan stabil, serta dukungan jangka panjang yang terjamin. Python 3.12+ masih dalam adopsi awal dan beberapa library AI (terutama yang bergantung pada C extension) belum sepenuhnya kompatibel.

**Fitur Python yang digunakan secara aktif:**

```python
# typing.Protocol — abstraction layer tanpa inheritance
from typing import Protocol

# dataclasses — data container yang bersih
from dataclasses import dataclass

# pathlib — file system yang idiomatik
from pathlib import Path

# logging — structured logging bawaan stdlib
import logging

# abc — hanya untuk dokumentasi, bukan enforcement
# (enforcement dilakukan oleh mypy)
```

**Yang tidak digunakan di v0.1:**

- `asyncio` — pipeline v0.1 berjalan secara sekuensial. Concurrent processing masuk di versi mendatang.
- `multiprocessing` — belum diperlukan sampai ada kebutuhan parallel model loading.

---

## Core Pipeline

### Komponen inti tidak memiliki dependency eksternal

Ini adalah keputusan arsitektur yang sangat penting. Semua file di dalam `core/` dan `interfaces/` hanya boleh mengimpor dari Python standard library. Tidak ada `import ollama`, `import whisper`, atau library eksternal apapun di sana.

Dependency eksternal hanya boleh ada di `providers/`.

---

## LLM Layer

### Ollama

**Peran:** Runtime untuk menjalankan LLM secara lokal.
**Versi:** Latest stable (Ollama tidak menggunakan semantic versioning yang ketat; selalu gunakan versi terbaru)
**Website:** https://ollama.com
**Tipe:** External service (berjalan sebagai background process, bukan library Python)

**Cara Aira berinteraksi dengan Ollama:**

Ollama berjalan sebagai service di `localhost:11434`. Aira tidak mengimpor Ollama sebagai library — ia berkomunikasi melalui REST API menggunakan `requests` atau `httpx`.

```python
# providers/llm/ollama_provider.py
import httpx

class OllamaProvider:
    BASE_URL = "http://localhost:11434"

    def generate(self, prompt: str, context: list[dict]) -> str:
        response = httpx.post(
            f"{self.BASE_URL}/api/chat",
            json={
                "model": self.model_name,
                "messages": context,
                "stream": False,
            }
        )
        return response.json()["message"]["content"]
```

**Mengapa tidak menggunakan library `ollama` Python?**

Library `ollama` Python adalah wrapper tipis di atas REST API yang sama. Menggunakannya menambahkan satu dependency yang tidak memberikan nilai signifikan — dan membuat kita bergantung pada versi library yang mungkin tidak selalu sinkron dengan Ollama server. Langsung menggunakan `httpx` lebih eksplisit, lebih mudah di-debug, dan tidak menambah dependency.

---

### Gemma 3 1B (via Ollama)

**Peran:** Model LLM default di v0.1.
**Model string:** `gemma3:1b`
**RAM footprint:** ~800MB–1GB (Q4 quantization, default Ollama)
**Provider:** Google DeepMind (open weights)

**Karakteristik yang relevan untuk Aira:**

- Multilingual: mendukung Bahasa Indonesia dengan cukup baik
- Context window: 8192 token (cukup untuk percakapan beberapa giliran)
- Instruction-following: cukup baik untuk mengikuti MASTER_PROMPT
- Kecepatan di CPU: lambat (~10–20 token/detik di i3), tapi acceptable untuk percakapan

**Konfigurasi yang direkomendasikan:**

```python
# config/settings.py
LLM_CONFIG = {
    "model": "gemma3:1b",
    "temperature": 0.7,      # Kreativitas yang seimbang
    "top_p": 0.9,
    "max_tokens": 512,        # Respons yang tidak terlalu panjang untuk percakapan
    "num_ctx": 4096,          # Context window yang aktif digunakan
}
```

**Upgrade path:**

Ketika perangkat memiliki RAM lebih besar atau performa yang lebih baik dibutuhkan, model bisa diupgrade hanya dengan mengubah nilai `model` di `settings.py`:

```
gemma3:1b    →  gemma3:4b   (butuh ~2.5GB RAM)
gemma3:4b    →  gemma3:12b  (butuh ~7GB RAM)
gemma3:1b    →  llama3.2:3b (alternatif dari Meta)
```

Tidak ada perubahan kode selain konfigurasi.

---

### httpx

**Peran:** HTTP client untuk berkomunikasi dengan Ollama REST API.
**Versi:** `>=0.25.0`
**Mengapa bukan `requests`?**

`httpx` adalah successor modern dari `requests` dengan API yang hampir identik. Keunggulan utamanya adalah dukungan native untuk async HTTP — yang akan dibutuhkan di versi mendatang ketika pipeline bergerak ke concurrent processing. Menggunakan `httpx` dari awal menghindari migration dari `requests` ke `httpx` di masa depan.

```
pip install httpx
```

---

## Audio Layer

### Speech-to-Text: openai-whisper

**Peran:** Mengubah audio dari mikrofon menjadi teks (Speech-to-Text).
**Versi:** `>=20231117`
**Model yang digunakan:** `tiny` (multilingual)
**RAM footprint:** ~390MB saat model ter-load
**Berjalan:** Sepenuhnya lokal, offline, tidak perlu API key

**Instalasi:**

```bash
pip install openai-whisper
# Whisper juga membutuhkan ffmpeg
# Windows: winget install ffmpeg
# atau download dari https://ffmpeg.org
```

**Dependensi Whisper:**

```
openai-whisper
    ├── torch          (PyTorch — terbesar, ~2GB saat install, ~500MB saat runtime)
    ├── numpy
    ├── tqdm
    ├── more-itertools
    ├── tiktoken
    └── ffmpeg-python  (Python binding untuk ffmpeg)
```

**Catatan penting tentang PyTorch:**

Whisper bergantung pada PyTorch. Di perangkat tanpa GPU (seperti setup Pasha saat ini), pastikan menginstall versi CPU-only untuk menghemat disk space:

```bash
pip install torch --index-url https://download.pytorch.org/whl/cpu
pip install openai-whisper
```

Jika install PyTorch versi GPU secara tidak sengaja, ukurannya bisa mencapai 2GB+ dan menginstall CUDA dependency yang tidak diperlukan.

**Cara penggunaan di Aira:**

```python
# providers/stt/whisper_provider.py
import whisper

class WhisperProvider:
    def __init__(self, model_name: str = "tiny"):
        # Model di-load sekali saat startup, bukan per request
        self.model = whisper.load_model(model_name)

    def transcribe(self, audio_path: str) -> str:
        result = self.model.transcribe(audio_path, language="id")
        return result["text"].strip()
```

**Mengapa model di-load sekali saat startup?**

Loading model Whisper membutuhkan waktu dan RAM. Jika model di-load setiap kali ada input suara, latensi akan sangat tinggi dan RAM akan terus dialokasikan dan di-dealokasikan. Load sekali di awal, simpan di memori selama aplikasi berjalan.

---

### Audio Capture: PyAudio

**Peran:** Menangkap audio stream dari mikrofon.
**Versi:** `>=0.2.13`

**Instalasi di Windows:**

```bash
pip install pyaudio
```

Jika instalasi gagal karena missing C dependency (PortAudio), gunakan wheel yang sudah di-compile:

```bash
pip install pipwin
pipwin install pyaudio
```

**Cara penggunaan di Aira:**

```python
# pipeline/voice_input.py
import pyaudio
import wave

class VoiceInput:
    CHUNK = 1024
    FORMAT = pyaudio.paInt16
    CHANNELS = 1
    RATE = 16000  # 16kHz — optimal untuk Whisper

    def capture(self, duration_seconds: int = 5) -> str:
        """Rekam audio dan simpan ke file temporary, kembalikan path-nya."""
        ...
```

**Mengapa sample rate 16000 Hz?**

Whisper dilatih dengan audio 16kHz. Input audio dengan sample rate berbeda akan di-resample secara internal oleh Whisper, yang menambah overhead komputasi yang tidak perlu. Menggunakan 16kHz dari awal menghindari resampling dan memberikan hasil transkripsi yang optimal.

---

### Text-to-Speech: pyttsx3 (v0.1 Placeholder)

**Peran:** Mengubah teks menjadi suara (Text-to-Speech). Placeholder untuk v0.1.
**Versi:** `>=2.90`
**Status:** Akan digantikan Kokoro TTS di v0.2 (lihat ADR-005)

**Instalasi:**

```bash
pip install pyttsx3
```

`pyttsx3` menggunakan SAPI5 di Windows — speech engine yang sudah built-in di sistem operasi. Tidak ada model tambahan yang perlu di-download.

**Konfigurasi dasar:**

```python
# providers/tts/pyttsx3_provider.py
import pyttsx3

class Pyttsx3Provider:
    def __init__(self):
        self.engine = pyttsx3.init()
        self.engine.setProperty("rate", 175)    # Kecepatan bicara (kata/menit)
        self.engine.setProperty("volume", 0.9)  # Volume 0.0–1.0

    def speak(self, text: str) -> None:
        self.engine.say(text)
        self.engine.runAndWait()
```

---

### Text-to-Speech: Kokoro TTS (v0.2 — Planned)

**Peran:** Menggantikan pyttsx3 sebagai TTS provider di v0.2.
**Repository:** https://github.com/hexgrad/kokoro
**RAM footprint:** ~500MB–1GB
**Status:** Planned — belum diimplementasi

**Mengapa Kokoro?**

Kokoro adalah model TTS neural open-source dengan kualitas suara yang sangat natural untuk ukurannya. Berjalan sepenuhnya lokal, tidak membutuhkan API key, dan mendukung berbagai voice style. Ini adalah pilihan yang paling sesuai dengan prinsip offline-first Aira sambil memberikan kualitas suara yang mencerminkan personality Aira.

**Catatan implementasi untuk v0.2:**

Pergantian dari pyttsx3 ke Kokoro hanya membutuhkan:
1. Membuat `providers/tts/kokoro_provider.py`
2. Mengubah satu baris di `main.py`

Tidak ada perubahan di `core/`, `interfaces/`, atau komponen lain.

---

## Development Tools

### Package Management: pip + venv

**Peran:** Mengelola Python dependencies dan virtual environment.

**Mengapa bukan Poetry atau Conda?**

Poetry dan Conda adalah tools yang bagus, tapi menambahkan complexity untuk proyek yang baru dimulai. `pip` + `venv` adalah standar Python yang paling portable, paling banyak didukung tutorial, dan paling mudah di-debug. Ketika proyek berkembang dan membutuhkan dependency resolution yang lebih canggih, migrasi ke Poetry bisa dilakukan.

**Setup environment:**

```bash
# Buat virtual environment
python -m venv .venv

# Aktivasi (Windows)
.venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

**Struktur requirements:**

```
requirements.txt          # Production dependencies
requirements-dev.txt      # Development + testing dependencies
```

```
# requirements.txt
httpx>=0.25.0
openai-whisper>=20231117
pyaudio>=0.2.13
pyttsx3>=2.90
torch --index-url https://download.pytorch.org/whl/cpu
```

```
# requirements-dev.txt
-r requirements.txt
pytest>=7.4.0
pytest-cov>=4.1.0
mypy>=1.5.0
ruff>=0.1.0
```

---

### Type Checking: mypy

**Peran:** Static type checker untuk memverifikasi kontrak Protocol terpenuhi.
**Versi:** `>=1.5.0`

`mypy` adalah komponen penting dalam arsitektur Aira karena kita menggunakan `Protocol` sebagai mekanisme abstraction. `mypy` yang memverifikasi bahwa setiap Provider benar-benar memenuhi Interface yang didefinisikan — error terdeteksi saat development, bukan saat runtime.

```bash
# Jalankan type checking
mypy aira/ --strict
```

**Konfigurasi (`mypy.ini`):**

```ini
[mypy]
python_version = 3.11
strict = true
ignore_missing_imports = true
```

---

### Linting & Formatting: Ruff

**Peran:** Linter dan formatter untuk menjaga konsistensi kode.
**Versi:** `>=0.1.0`
**Menggantikan:** `flake8` + `black` + `isort` dalam satu tool

`ruff` adalah linter Python modern yang sangat cepat (ditulis dalam Rust) dan menggantikan beberapa tool sekaligus. Menggunakannya dari awal memastikan seluruh kodebase memiliki gaya yang konsisten.

```bash
# Check
ruff check aira/

# Format
ruff format aira/
```

**Konfigurasi (`ruff.toml`):**

```toml
[tool.ruff]
line-length = 88
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP"]
```

---

## Testing Stack

### pytest

**Peran:** Framework untuk unit test dan integration test.
**Versi:** `>=7.4.0`

```bash
# Jalankan semua test
pytest

# Dengan coverage report
pytest --cov=aira --cov-report=term-missing
```

### pytest-cov

**Peran:** Plugin pytest untuk mengukur code coverage.
**Versi:** `>=4.1.0`

**Target coverage v0.1:** Minimum 70% untuk `core/` dan `interfaces/`. Provider boleh lebih rendah karena bergantung pada service eksternal (Ollama, hardware).

---

## Dependency Map

Visualisasi dependency antar layer — panah menunjukkan arah import:

```
main.py
    │
    ├──► core/orchestrator.py
    │         │
    │         ├──► interfaces/llm_interface.py      (Protocol)
    │         ├──► interfaces/stt_interface.py      (Protocol)
    │         ├──► interfaces/tts_interface.py      (Protocol)
    │         └──► core/context_manager.py
    │
    ├──► providers/llm/ollama_provider.py
    │         │
    │         └──► httpx                            (external)
    │
    ├──► providers/stt/whisper_provider.py
    │         │
    │         └──► openai-whisper                   (external)
    │              └──► torch                       (external)
    │
    ├──► providers/tts/pyttsx3_provider.py
    │         │
    │         └──► pyttsx3                          (external)
    │
    ├──► pipeline/voice_input.py
    │         │
    │         └──► pyaudio                          (external)
    │
    └──► personality/master_prompt.py               (no external deps)

ATURAN:
✅ core/       → interfaces/     (diizinkan)
✅ providers/  → interfaces/     (diizinkan)
✅ providers/  → external libs   (diizinkan)
✅ main.py     → providers/      (diizinkan)
✅ main.py     → core/           (diizinkan)
❌ core/       → providers/      (DILARANG)
❌ interfaces/ → external libs   (DILARANG)
❌ core/       → external libs   (DILARANG)
```

---

## Versi & Kompatibilitas

### Compatibility Matrix v0.1

| Komponen | Versi | Python | OS |
|---|---|---|---|
| Python | 3.11+ | — | Windows 11 |
| httpx | >=0.25.0 | 3.8+ | All |
| openai-whisper | >=20231117 | 3.8+ | All |
| PyTorch (CPU) | >=2.0.0 | 3.8+ | All |
| pyaudio | >=0.2.13 | 3.x | All* |
| pyttsx3 | >=2.90 | 3.x | Windows/Mac/Linux |
| mypy | >=1.5.0 | 3.8+ | All |
| pytest | >=7.4.0 | 3.8+ | All |
| ruff | >=0.1.0 | — | All |
| Ollama | Latest | — | Windows/Mac/Linux |

*pyaudio di Linux membutuhkan `portaudio19-dev` via apt.

### Estimasi RAM saat runtime v0.1

```
Komponen               Estimasi RAM
─────────────────────────────────────
OS + background        ~1.2GB
Python runtime         ~50MB
Whisper tiny (loaded)  ~390MB
Ollama (Gemma 3 1B)    ~800MB–1GB
pyttsx3                ~5MB
─────────────────────────────────────
Total estimasi         ~2.4GB–2.6GB
Tersisa (dari 4GB)     ~1.4GB–1.6GB
```

Angka-angka di atas adalah estimasi. RAM usage aktual dapat bervariasi tergantung kondisi sistem. Tersisa ~1.4GB memberikan buffer yang cukup untuk operasi normal.

**Catatan:** Whisper dan Ollama tidak perlu berjalan bersamaan setiap saat. Whisper hanya aktif saat transkripsi berlangsung. Jika RAM terlalu ketat, optimasi bisa dilakukan dengan unloading Whisper model setelah transkripsi selesai dan sebelum Ollama mulai generate.

---

## Upgrade Path

Dokumen ini akan diperbarui setiap kali ada perubahan teknologi. Berikut rencana upgrade yang sudah diantisipasi:

| Versi | Komponen | Dari | Ke | Dampak |
|---|---|---|---|---|
| v0.2 | TTS | pyttsx3 | Kokoro TTS | Hanya `providers/tts/` |
| v0.3 | Storage | in-memory | SQLite/JSON | Hanya `core/context_manager.py` |
| v0.4 | LLM | Gemma 3 1B | Multimodal model | Hanya `config/settings.py` |
| v1.x | HTTP client | httpx sync | httpx async | `providers/llm/` + `core/` |

Perhatikan bahwa setiap upgrade hanya mempengaruhi satu atau dua file — ini adalah bukti bahwa abstraction layer yang dibangun di v0.1 bekerja sesuai rencana.

---

*Dokumen ini diperbarui setiap kali ada perubahan versi dependency atau penambahan teknologi baru.*
*Setiap penambahan teknologi harus disertai ADR baru jika keputusannya signifikan.*

*Last updated: 2026 | Version: 1.0.0*
