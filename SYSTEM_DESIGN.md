# SYSTEM_DESIGN — Project Aira

> *"Arsitektur menjelaskan apa yang dibangun. System Design menjelaskan bagaimana ia bekerja."*

---

## Daftar Isi

1. [Tujuan Dokumen](#tujuan-dokumen)
2. [Komponen Detail](#komponen-detail)
3. [Interface Contract](#interface-contract)
4. [Sequence Diagram](#sequence-diagram)
5. [State Machine](#state-machine)
6. [Error Handling Strategy](#error-handling-strategy)
7. [Configuration Design](#configuration-design)
8. [Dependency Injection](#dependency-injection)
9. [Startup & Shutdown Sequence](#startup--shutdown-sequence)
10. [Batasan Sistem](#batasan-sistem)

---

## Tujuan Dokumen

ARCHITECTURE.md menjelaskan *apa* yang dibangun dan *mengapa* strukturnya seperti itu.

SYSTEM_DESIGN.md menjelaskan *bagaimana* setiap komponen bekerja secara detail: kontrak interface yang tepat, urutan pemanggilan yang eksak, cara error ditangani, dan bagaimana seluruh sistem dirakit dari potongan-potongannya.

Dokumen ini adalah referensi utama saat implementasi. Setiap keputusan di sini sudah dipikirkan sebelum satu baris kode ditulis.

---

## Komponen Detail

### Orchestrator

Orchestrator adalah komponen paling sentral di sistem. Ia tidak memiliki logika bisnis — tugasnya murni mengatur urutan eksekusi.

**Tanggung jawab yang tepat:**
- Menerima audio dari VoiceInput
- Mendelegasikan transkripsi ke STT
- Mengambil context dari ContextManager
- Membangun pesan lengkap (MASTER_PROMPT + history + input)
- Mendelegasikan generation ke LLM
- Menyimpan giliran baru ke ContextManager
- Mendelegasikan output ke TTS

**Yang bukan tanggung jawab Orchestrator:**
- Memproses audio (milik STT)
- Menyimpan history (milik ContextManager)
- Mengetahui nama model atau provider (milik Provider)
- Memformat teks output (milik TTS)

**Ukuran yang wajar:** Orchestrator yang sehat berukuran ~50–80 baris. Jika mulai melebihi 100 baris, ada logika yang seharusnya ada di tempat lain.

---

### ContextManager

ContextManager menyimpan percakapan sebagai list of message dictionaries — format yang langsung kompatibel dengan OpenAI-style chat API yang digunakan oleh hampir semua LLM modern termasuk Ollama.

**Format internal:**

```python
# Internal storage — format OpenAI chat
messages: list[dict] = [
    {"role": "user",      "content": "halo Aira"},
    {"role": "assistant", "content": "halo Pasha, ada yang bisa saya bantu?"},
    {"role": "user",      "content": "aku lagi bingung soal ..."},
]
```

**MASTER_PROMPT tidak disimpan di sini.** MASTER_PROMPT diinjeksikan oleh Orchestrator setiap kali membangun message list untuk LLM call. ContextManager hanya menyimpan percakapan aktual.

**Window management:**

Untuk mencegah context window LLM overflow, ContextManager menerapkan sliding window — hanya N pesan terakhir yang dikirimkan ke LLM. Di v0.1, N = 20 (10 giliran percakapan).

```python
def get_messages(self, max_messages: int = 20) -> list[dict]:
    """Kembalikan max_messages pesan terakhir."""
    return self._messages[-max_messages:]
```

---

### VoiceInput

VoiceInput bertanggung jawab atas satu hal: menangkap audio dari mikrofon dan mengembalikan path ke file audio temporary.

**Mengapa file, bukan bytes langsung?**

Whisper bekerja paling baik dengan file audio. Mengirim bytes langsung membutuhkan in-memory file object yang lebih kompleks untuk di-debug. File temporary juga memudahkan debugging — jika transkripsi salah, file audio bisa diputar ulang untuk verifikasi.

**Format audio:**

```
Format  : WAV
Channels: 1 (mono)
Rate    : 16000 Hz
Depth   : 16-bit (paInt16)
```

16kHz mono adalah format optimal untuk Whisper. Stereo tidak memberikan manfaat untuk speech recognition dan hanya menggandakan ukuran file.

---

### STT (WhisperProvider)

STT menerima path file audio dan mengembalikan string teks hasil transkripsi.

**Model loading strategy:**

Model Whisper di-load satu kali saat startup (`__init__`), bukan per request. Ini adalah keputusan performa yang penting — loading model Whisper tiny membutuhkan ~2–3 detik. Jika di-load per request, setiap percakapan akan memiliki delay 2–3 detik hanya untuk loading.

**Language detection:**

Di v0.1, bahasa di-set eksplisit ke `"id"` (Indonesian) untuk menghindari language detection overhead dan meningkatkan akurasi. Jika Pasha berbicara dalam bahasa lain, transkripsi mungkin kurang akurat — ini adalah trade-off yang diterima di v0.1.

---

### LLMInterface & OllamaProvider

OllamaProvider berkomunikasi dengan Ollama melalui REST API di `localhost:11434`.

**Endpoint yang digunakan:**

```
POST http://localhost:11434/api/chat
```

**Request format:**

```json
{
    "model": "gemma3:1b",
    "messages": [
        {"role": "system",    "content": "...MASTER_PROMPT..."},
        {"role": "user",      "content": "halo"},
        {"role": "assistant", "content": "halo Pasha"},
        {"role": "user",      "content": "input terbaru"}
    ],
    "stream": false,
    "options": {
        "temperature": 0.7,
        "top_p": 0.9,
        "num_predict": 512
    }
}
```

**Mengapa `stream: false`?**

Di v0.1, TTS harus menunggu response lengkap sebelum bisa mulai berbicara. Streaming response tidak memberikan manfaat karena TTS tidak bisa berbicara kata per kata — ia membutuhkan kalimat yang lengkap. Streaming akan dipertimbangkan di versi mendatang jika ada kebutuhan untuk menampilkan teks response secara real-time.

---

## Interface Contract

Ini adalah kontrak yang harus dipenuhi oleh setiap implementasi provider. Kontrak ini tidak boleh berubah tanpa ADR baru.

### LLMInterface

```python
# interfaces/llm_interface.py
from typing import Protocol


class LLMInterface(Protocol):
    """
    Kontrak untuk semua LLM provider.

    Implementor harus memenuhi signature ini persis.
    mypy akan memverifikasi kepatuhan saat development.
    """

    def generate(
        self,
        messages: list[dict[str, str]],
    ) -> str:
        """
        Generate respons berdasarkan message history.

        Args:
            messages: List of message dicts dengan format:
                      [{"role": "system"|"user"|"assistant", "content": str}]
                      MASTER_PROMPT sudah termasuk sebagai pesan pertama
                      dengan role "system".

        Returns:
            String respons dari model. Tidak pernah None.
            Tidak mengandung leading/trailing whitespace.

        Raises:
            LLMConnectionError: Jika tidak bisa terhubung ke provider.
            LLMTimeoutError: Jika provider tidak merespons dalam batas waktu.
        """
        ...
```

### STTInterface

```python
# interfaces/stt_interface.py
from typing import Protocol


class STTInterface(Protocol):
    """Kontrak untuk semua Speech-to-Text provider."""

    def transcribe(self, audio_path: str) -> str:
        """
        Transkripsi file audio menjadi teks.

        Args:
            audio_path: Path absolut ke file audio WAV 16kHz mono.

        Returns:
            Teks hasil transkripsi. String kosong jika tidak ada suara.
            Tidak mengandung leading/trailing whitespace.

        Raises:
            STTFileNotFoundError: Jika file audio tidak ditemukan.
            STTTranscriptionError: Jika transkripsi gagal.
        """
        ...
```

### TTSInterface

```python
# interfaces/tts_interface.py
from typing import Protocol


class TTSInterface(Protocol):
    """Kontrak untuk semua Text-to-Speech provider."""

    def speak(self, text: str) -> None:
        """
        Ubah teks menjadi suara dan putar melalui speaker.

        Args:
            text: Teks yang akan diucapkan. Tidak boleh kosong.

        Returns:
            None. Method ini blocking — tidak return sampai
            audio selesai diputar.

        Raises:
            TTSSpeakError: Jika terjadi error saat synthesis atau playback.
        """
        ...
```

### ContextManager Interface

```python
# core/context_manager.py (interface implisit)

class ContextManager:
    """
    Mengelola conversation history dalam satu sesi.

    Public interface:
    - get_messages(max_messages) -> list[dict]
    - add_turn(user, assistant) -> None
    - clear() -> None
    - is_empty() -> bool
    """
```

---

## Sequence Diagram

### Happy Path — Percakapan Normal

```
Pasha       VoiceInput    Orchestrator   ContextMgr   STT          LLM          TTS
  │               │              │             │        │             │             │
  │ [bicara]      │              │             │        │             │             │
  ├──────────────►│              │             │        │             │             │
  │               │ capture()    │             │        │             │             │
  │               │──────────────────────────────────────────────────────────────► │
  │               │              │             │        │             │             │
  │               │ audio_path   │             │        │             │             │
  │               │─────────────►│             │        │             │             │
  │               │              │             │        │             │             │
  │               │              │ transcribe(audio_path)             │             │
  │               │              │─────────────────────►│             │             │
  │               │              │             │        │             │             │
  │               │              │ user_text   │        │             │             │
  │               │              │◄─────────────────────│             │             │
  │               │              │             │        │             │             │
  │               │              │ get_messages()        │             │             │
  │               │              │────────────►│         │             │             │
  │               │              │             │         │             │             │
  │               │              │ history     │         │             │             │
  │               │              │◄────────────│         │             │             │
  │               │              │             │         │             │             │
  │               │              │ [build messages:                    │             │
  │               │              │  MASTER_PROMPT +                   │             │
  │               │              │  history +                         │             │
  │               │              │  user_text]                        │             │
  │               │              │             │         │             │             │
  │               │              │ generate(messages)                 │             │
  │               │              │────────────────────────────────────►│             │
  │               │              │             │         │             │             │
  │               │              │ response_text         │             │             │
  │               │              │◄────────────────────────────────────│             │
  │               │              │             │         │             │             │
  │               │              │ add_turn(user_text, response_text) │             │
  │               │              │────────────►│         │             │             │
  │               │              │             │         │             │             │
  │               │              │ speak(response_text)               │             │
  │               │              │─────────────────────────────────────────────────►│
  │               │              │             │         │             │             │
  │ [mendengar]   │              │             │         │             │             │
  │◄──────────────────────────────────────────────────────────────────────────────── │
  │               │              │             │         │             │             │
```

### Error Path — Ollama Tidak Berjalan

```
Pasha       VoiceInput    Orchestrator   STT          LLM(Ollama)
  │               │              │        │             │
  │ [bicara]      │              │        │             │
  ├──────────────►│              │        │             │
  │               │ audio_path   │        │             │
  │               │─────────────►│        │             │
  │               │              │        │             │
  │               │              │ transcribe()         │
  │               │              │───────►│             │
  │               │              │ text   │             │
  │               │              │◄───────│             │
  │               │              │        │             │
  │               │              │ generate(messages)   │
  │               │              │─────────────────────►│
  │               │              │        │         [CONNECTION REFUSED]
  │               │              │        │             │
  │               │              │◄── LLMConnectionError│
  │               │              │        │             │
  │               │ [log error]  │        │             │
  │               │              │        │             │
  │ [Aira: "Maaf, │              │        │             │
  │  ada masalah  │              │        │             │
  │  koneksi"]    │              │        │             │
  │◄──────────────────────────────────────────────────── │
```

Ketika LLM tidak bisa dihubungi, Orchestrator menangkap exception dan merespons dengan pesan error yang natural — tidak membiarkan aplikasi crash.

---

## State Machine

Aira v0.1 memiliki state machine yang sederhana. Memahami state ini penting karena banyak bug yang terjadi karena transisi state yang tidak tepat.

```
                    ┌─────────────────────────────────────┐
                    │                                     │
                    ▼                                     │
            ┌──────────────┐                             │
            │  INITIALIZING │                             │
            │               │                             │
            │ Load Whisper  │                             │
            │ Check Ollama  │                             │
            │ Init TTS      │                             │
            └──────┬────────┘                             │
                   │ success                              │
                   ▼                                      │
            ┌──────────────┐     timeout/silence          │
            │   LISTENING   │◄─────────────────────────── │
            │               │                             │
            │ Mic aktif     │                             │
            │ Menunggu suara│                             │
            └──────┬────────┘                             │
                   │ voice detected                       │
                   ▼                                      │
            ┌──────────────┐                             │
            │  CAPTURING   │                             │
            │               │                             │
            │ Rekam audio   │                             │
            │ Sampai selesai│                             │
            └──────┬────────┘                             │
                   │ audio captured                       │
                   ▼                                      │
            ┌──────────────┐                             │
            │ TRANSCRIBING │                             │
            │               │                             │
            │ Whisper proses│                             │
            │ audio → text  │                             │
            └──────┬────────┘                             │
                   │ text ready                           │
                   ▼                                      │
            ┌──────────────┐                             │
            │  GENERATING  │                             │
            │               │                             │
            │ LLM proses   │                             │
            │ text → response                            │
            └──────┬────────┘                             │
                   │ response ready                       │
                   ▼                                      │
            ┌──────────────┐                             │
            │   SPEAKING   │                             │
            │               │                             │
            │ TTS ubah      │                             │
            │ response→suara│─────────────────────────────┘
            └──────────────┘
                   │
            [SHUTDOWN sinyal]
                   │
                   ▼
            ┌──────────────┐
            │ SHUTTING_DOWN│
            │               │
            │ Release mic   │
            │ Unload models │
            │ Close TTS     │
            └──────────────┘
```

**State yang tidak ada di v0.1:**

- `WAKE_WORD_DETECTING` — Aira selalu mendengarkan, tidak ada wake word di v0.1
- `INTERRUPTED` — Aira tidak bisa diinterupsi saat berbicara di v0.1
- `THINKING` (visible) — Tidak ada indikator visual bahwa Aira sedang memproses

---

## Error Handling Strategy

Setiap layer memiliki tanggung jawab error yang berbeda.

### Hierarki Exception

```python
# core/exceptions.py

class AiraError(Exception):
    """Base exception untuk semua error Aira."""
    pass

# LLM Errors
class LLMError(AiraError):
    """Base untuk semua LLM-related errors."""
    pass

class LLMConnectionError(LLMError):
    """Tidak bisa terhubung ke LLM provider (Ollama tidak berjalan, dll)."""
    pass

class LLMTimeoutError(LLMError):
    """LLM provider tidak merespons dalam batas waktu."""
    pass

class LLMResponseError(LLMError):
    """LLM memberikan respons yang tidak valid atau kosong."""
    pass

# STT Errors
class STTError(AiraError):
    """Base untuk semua STT-related errors."""
    pass

class STTTranscriptionError(STTError):
    """Transkripsi gagal."""
    pass

class STTFileNotFoundError(STTError):
    """File audio tidak ditemukan."""
    pass

# TTS Errors
class TTSError(AiraError):
    """Base untuk semua TTS-related errors."""
    pass

class TTSSpeakError(TTSError):
    """Gagal mensintesis atau memutar suara."""
    pass

# Audio Errors
class AudioCaptureError(AiraError):
    """Gagal menangkap audio dari mikrofon."""
    pass
```

### Siapa yang Menangani Apa?

```
Provider Layer    → raise exception spesifik (LLMConnectionError, dll)
                    Provider TIDAK menangani error — ia hanya melaporkan

Orchestrator      → catch exception dari provider
                    Log error
                    Respond dengan pesan fallback yang natural
                    Kembali ke state LISTENING

main.py           → catch exception yang tidak tertangani
                    Log critical error
                    Graceful shutdown
```

### Fallback Messages

Ketika terjadi error, Aira tetap merespons dengan suara — tidak diam. Respons disesuaikan dengan jenis error:

```python
# core/orchestrator.py
FALLBACK_MESSAGES = {
    "llm_connection": "Maaf Pasha, ada masalah koneksi. Coba lagi sebentar.",
    "llm_timeout":    "Aira butuh waktu lebih lama dari biasanya. Coba lagi?",
    "stt_error":      "Maaf, Aira tidak menangkap dengan jelas. Bisa diulang?",
    "tts_error":      "Ada masalah dengan suara. Cek output audio.",
    "unknown":        "Ada yang tidak beres. Aira akan coba lagi.",
}
```

---

## Configuration Design

Semua konfigurasi terpusat di satu tempat dan tidak ada magic number yang tersebar di kode.

```python
# config/settings.py
from dataclasses import dataclass, field
from pathlib import Path


@dataclass(frozen=True)
class AudioConfig:
    """Konfigurasi audio capture."""
    sample_rate: int = 16_000      # Hz — optimal untuk Whisper
    channels: int = 1              # Mono
    chunk_size: int = 1_024        # Buffer size per read
    record_seconds: int = 5        # Durasi recording per turn
    format: int = 8                # pyaudio.paInt16


@dataclass(frozen=True)
class WhisperConfig:
    """Konfigurasi Whisper STT."""
    model_name: str = "tiny"       # tiny|base|small|medium|large
    language: str = "id"           # Force Indonesian
    fp16: bool = False             # False untuk CPU (tidak ada GPU)


@dataclass(frozen=True)
class OllamaConfig:
    """Konfigurasi Ollama LLM provider."""
    base_url: str = "http://localhost:11434"
    model_name: str = "gemma3:1b"
    temperature: float = 0.7
    top_p: float = 0.9
    max_tokens: int = 512
    timeout_seconds: int = 60      # Timeout per request


@dataclass(frozen=True)
class ContextConfig:
    """Konfigurasi context management."""
    max_messages: int = 20         # Sliding window size


@dataclass(frozen=True)
class AiraConfig:
    """Root configuration — satu-satunya yang diinstansiasi di main.py."""
    audio: AudioConfig = field(default_factory=AudioConfig)
    whisper: WhisperConfig = field(default_factory=WhisperConfig)
    ollama: OllamaConfig = field(default_factory=OllamaConfig)
    context: ContextConfig = field(default_factory=ContextConfig)
    temp_dir: Path = Path("./temp")
    log_level: str = "INFO"


# Instance default yang digunakan seluruh sistem
DEFAULT_CONFIG = AiraConfig()
```

**Mengapa `frozen=True`?**

Konfigurasi adalah konstanta — tidak boleh berubah setelah startup. `frozen=True` membuat dataclass immutable, sehingga tidak ada kode yang bisa mengubah konfigurasi secara tidak sengaja saat runtime.

---

## Dependency Injection

Seluruh sistem dirakit di satu tempat: `main.py`. Tidak ada komponen yang membuat instance komponen lain secara internal.

```python
# main.py
import logging
from pathlib import Path

from aira.config.settings import AiraConfig, DEFAULT_CONFIG
from aira.core.context_manager import ContextManager
from aira.core.orchestrator import Orchestrator
from aira.personality.master_prompt import MASTER_PROMPT
from aira.pipeline.voice_input import VoiceInput
from aira.providers.llm.ollama_provider import OllamaProvider
from aira.providers.stt.whisper_provider import WhisperProvider
from aira.providers.tts.pyttsx3_provider import Pyttsx3Provider


def build_aira(config: AiraConfig = DEFAULT_CONFIG) -> Orchestrator:
    """
    Factory function — satu-satunya tempat dependency injection terjadi.

    Semua komponen dibuat di sini dan disuntikkan ke Orchestrator.
    Tidak ada komponen lain yang membuat instance komponen lain.
    """
    logging.basicConfig(level=config.log_level)
    logger = logging.getLogger(__name__)

    logger.info("Initializing Aira...")

    # Buat temp directory jika belum ada
    config.temp_dir.mkdir(exist_ok=True)

    # Instantiate providers
    logger.info("Loading Whisper model...")
    stt = WhisperProvider(config.whisper)

    logger.info("Initializing TTS...")
    tts = Pyttsx3Provider()

    logger.info("Connecting to Ollama...")
    llm = OllamaProvider(config.ollama)

    # Instantiate core components
    context = ContextManager(config.context)

    # Rakit semua komponen menjadi Orchestrator
    orchestrator = Orchestrator(
        stt=stt,
        llm=llm,
        tts=tts,
        context=context,
        master_prompt=MASTER_PROMPT,
        config=config,
    )

    logger.info("Aira is ready.")
    return orchestrator


def main() -> None:
    aira = build_aira()
    voice_input = VoiceInput(DEFAULT_CONFIG.audio)

    print("Aira aktif. Tekan Ctrl+C untuk keluar.\n")

    try:
        while True:
            audio_path = voice_input.capture()
            aira.process_turn(audio_path)
    except KeyboardInterrupt:
        print("\nAira: Sampai jumpa, Pasha.")


if __name__ == "__main__":
    main()
```

**Mengapa factory function `build_aira()`, bukan langsung di `main()`?**

Factory function yang terpisah memungkinkan testing — kita bisa memanggil `build_aira()` dengan konfigurasi berbeda dalam test tanpa harus menjalankan `main()`. Ini adalah pola yang sangat umum di production Python code.

---

## Startup & Shutdown Sequence

### Startup

```
1. Parse configuration (AiraConfig)
2. Setup logging
3. Create temp directory
4. Load Whisper model
   └── Jika gagal: exit dengan pesan yang jelas
5. Initialize pyttsx3
   └── Jika gagal: exit dengan pesan yang jelas
6. Verify Ollama connection
   └── GET http://localhost:11434/api/tags
   └── Jika gagal: tampilkan pesan "Pastikan Ollama berjalan"
       dan exit
7. Verify model tersedia di Ollama
   └── Jika model belum di-pull: tampilkan perintah pull
       dan exit
8. Initialize ContextManager
9. Build Orchestrator
10. Mulai main loop
```

**Mengapa verifikasi Ollama di startup, bukan saat request pertama?**

Fail fast — lebih baik gagal dengan pesan yang jelas di awal daripada berhasil startup tapi crash saat percakapan pertama. Pengguna tahu persis apa yang perlu diperbaiki.

### Shutdown (Ctrl+C)

```
1. KeyboardInterrupt tertangkap di main loop
2. Aira mengucapkan "Sampai jumpa, Pasha." (jika TTS masih aktif)
3. Release mikrofon (pyaudio stream ditutup)
4. Hapus file audio temporary
5. Exit
```

Whisper dan Ollama tidak perlu "di-shutdown" secara eksplisit — Whisper akan di-garbage collect oleh Python, dan Ollama adalah service terpisah yang terus berjalan.

---

## Batasan Sistem

Batasan ini adalah keputusan yang disengaja di v0.1, bukan kekurangan yang belum diperbaiki.

### Single-turn Processing
Aira memproses satu giliran dalam satu waktu. Saat Aira sedang memproses (transcribing atau generating), input baru diabaikan. Tidak ada queue, tidak ada interrupt.

**Mengapa:** Menghindari race condition dan kompleksitas concurrent state management di fase fondasi.

### Blocking TTS
TTS berjalan secara blocking — Aira tidak bisa mendengar atau memproses apapun saat sedang berbicara.

**Mengapa:** Simplicity. Non-blocking TTS membutuhkan threading yang menambah kompleksitas debugging secara signifikan.

### No Silence Detection
VoiceInput merekam selama durasi fixed (default 5 detik) — tidak ada voice activity detection (VAD) yang mendeteksi kapan pengguna selesai berbicara.

**Mengapa:** VAD menambah satu dependency lagi (webrtcvad atau silero-vad). Di v0.1, fixed duration adalah trade-off yang diterima. VAD masuk di iterasi berikutnya setelah pipeline dasar terbukti stabil.

### In-Memory Context Only
Conversation history hilang ketika aplikasi ditutup. Tidak ada persistent storage.

**Mengapa:** Long-term memory adalah fitur v0.3. Membangunnya di v0.1 berarti mendahulukan fitur sebelum fondasi terbukti stabil.

### No GUI
Aira v0.1 adalah aplikasi command-line. Tidak ada interface visual.

**Mengapa:** GUI menambah complexity yang tidak berkontribusi pada validasi arsitektur. Console output sudah cukup untuk v0.1.

---

*Dokumen ini diperbarui setiap kali ada perubahan signifikan pada desain sistem.*
*Setiap perubahan yang bertentangan dengan keputusan di sini harus disertai ADR baru.*

*Last updated: 2026 | Version: 1.0.0*
