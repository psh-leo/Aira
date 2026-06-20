# ARCHITECTURE — Project Aira

> *"Arsitektur yang baik bukan yang paling canggih, melainkan yang paling mudah diubah."*

---

## Daftar Isi

1. [Prinsip Arsitektur](#prinsip-arsitektur)
2. [Gambaran Besar Sistem](#gambaran-besar-sistem)
3. [Lapisan Arsitektur](#lapisan-arsitektur)
4. [Komponen Utama](#komponen-utama)
5. [Alur Data](#alur-data)
6. [Abstraction Layer](#abstraction-layer)
7. [Folder Structure](#folder-structure)
8. [Keputusan Arsitektur](#keputusan-arsitektur)
9. [Batasan v0.1](#batasan-v01)
10. [Evolusi Arsitektur](#evolusi-arsitektur)

---

## Prinsip Arsitektur

Seluruh keputusan arsitektur di proyek ini berpijak pada lima prinsip berikut. Urutan ini bukan kebetulan — ketika ada konflik antara dua prinsip, prinsip yang lebih atas selalu menang.

### 1. Separation of Concerns (SoC)

Setiap komponen hanya boleh tahu satu hal dan melakukannya dengan baik.

Konsekuensinya: `VoiceInput` tidak boleh tahu bahwa ada Whisper di baliknya. `OllamaProvider` tidak boleh tahu bahwa ada `ContextManager` yang mengelola history. Ketidaktahuan ini bukan kelemahan — ini adalah kekuatan arsitektur.

### 2. Dependency Inversion

Komponen tingkat tinggi tidak boleh bergantung pada komponen tingkat rendah. Keduanya harus bergantung pada abstraksi.

Dalam konteks Aira: `Orchestrator` tidak boleh memanggil `OllamaProvider` secara langsung. `Orchestrator` hanya boleh berbicara dengan `LLMInterface` — sebuah kontrak abstrak. Siapa yang memenuhi kontrak itu (Ollama, OpenAI, Gemini) adalah urusan lapisan di bawahnya.

### 3. Single Responsibility

Satu kelas, satu alasan untuk berubah.

Jika `ContextManager` perlu diubah karena ada perubahan cara menyimpan history, perubahan itu tidak boleh mempengaruhi `Orchestrator` atau `LLMInterface`. Jika `pyttsx3` diganti dengan `Kokoro`, hanya `TTSProvider` yang perlu diubah.

### 4. Open/Closed

Komponen terbuka untuk ekstensi, tertutup untuk modifikasi.

Menambahkan provider LLM baru harus dilakukan dengan *menambahkan* file baru, bukan dengan *mengubah* kode yang sudah ada. Ini yang membuat sistem bisa berkembang tanpa takut merusak yang sudah berfungsi.

### 5. Explicitness over Cleverness

Kode yang mudah dibaca lebih berharga daripada kode yang pintar.

Di Aira, tidak ada magic, tidak ada metaprogramming yang tidak perlu, tidak ada shortcut yang menghemat 5 baris tapi membutuhkan 30 menit untuk dipahami. Setiap keputusan harus bisa dijelaskan kepada orang lain.

---

## Gambaran Besar Sistem

Aira v0.1 adalah sistem pipeline satu arah dengan satu titik kontrol sentral.

```
┌─────────────────────────────────────────────────────────────┐
│                        EXTERNAL LAYER                        │
│                                                              │
│         [Microphone]                    [Speaker]            │
└──────────────┬──────────────────────────────┬───────────────┘
               │ audio stream                 │ speech output
               ▼                              ▲
┌─────────────────────────────────────────────────────────────┐
│                       PIPELINE LAYER                         │
│                                                              │
│      [Voice Input]                  [TTS Output]             │
│      Capture & buffer               Synthesize speech        │
└──────────────┬──────────────────────────────┬───────────────┘
               │ raw audio                    │ response text
               ▼                              │
┌─────────────────────────────────────────────────────────────┐
│                    ORCHESTRATION LAYER                        │
│                                                              │
│                      [Orchestrator]                          │
│              Satu-satunya yang tahu seluruh alur             │
│                                                              │
│         ┌────────────┐         ┌────────────────┐           │
│         │   STT      │         │  LLM Interface │           │
│         │  (Whisper) │         │  (abstraction) │           │
│         └────────────┘         └───────┬────────┘           │
│                                        │                     │
│         ┌────────────────────┐         │                     │
│         │  Context Manager   │         │                     │
│         │  (history & state) │         │                     │
│         └────────────────────┘         │                     │
└────────────────────────────────────────┼─────────────────────┘
                                         │ provider call
                                         ▼
┌─────────────────────────────────────────────────────────────┐
│                      PROVIDER LAYER                          │
│                                                              │
│     [OllamaProvider]   [OpenAIProvider]*  [GeminiProvider]* │
│      Gemma 3 1B          (future)           (future)         │
│                                                              │
│     * Belum diimplementasi di v0.1                          │
└─────────────────────────────────────────────────────────────┘
```

Empat lapisan ini memiliki satu aturan yang tidak boleh dilanggar: **lapisan atas tidak pernah mengimpor lapisan bawah secara langsung.** Komunikasi antar lapisan selalu melalui abstraksi.

---

## Lapisan Arsitektur

### External Layer

Dunia luar yang tidak bisa dikontrol: mikrofon fisik dan speaker fisik. Lapisan ini tidak memiliki logika — ia hanya sumber dan tujuan sinyal.

### Pipeline Layer

Komponen yang berhadapan langsung dengan External Layer. Tugasnya satu: menjadi jembatan antara dunia luar dan sistem internal.

- `VoiceInput` — menangkap audio dari mikrofon dan meneruskannya ke sistem.
- `TTSOutput` — menerima teks dari sistem dan meneruskannya ke speaker.

Penting: kedua komponen ini **tidak memproses** konten. Mereka hanya memindahkan data.

### Orchestration Layer

Otak sistem. `Orchestrator` adalah satu-satunya komponen yang tahu urutan kerja: *"Terima audio → transkripsi → ambil context → generate response → speak."*

Komponen lain di lapisan ini:
- `STT` — mengubah audio menjadi teks (menggunakan Whisper, tapi `Orchestrator` tidak perlu tahu ini).
- `LLMInterface` — kontrak abstrak untuk semua interaksi dengan language model.
- `ContextManager` — menyimpan dan mengelola riwayat percakapan.

### Provider Layer

Implementasi konkret dari `LLMInterface`. Di v0.1, hanya ada `OllamaProvider`. Di masa depan, lapisan ini bisa berkembang tanpa mengubah satu baris pun kode di lapisan Orchestration.

---

## Komponen Utama

### Orchestrator

**Tanggung jawab:** Mengatur urutan eksekusi seluruh pipeline.

**Yang boleh dilakukan:**
- Memanggil `STT.transcribe(audio)`
- Memanggil `ContextManager.get_context()`
- Memanggil `LLMInterface.generate(prompt, context)`
- Memanggil `TTSOutput.speak(text)`

**Yang tidak boleh dilakukan:**
- Mengetahui bahwa STT menggunakan Whisper
- Mengetahui bahwa LLM menggunakan Ollama
- Menyimpan conversation history sendiri
- Memproses audio atau text secara langsung

```python
# Apa yang Orchestrator lihat — tidak ada nama provider, tidak ada library
class Orchestrator:
    def __init__(
        self,
        stt: STTInterface,
        llm: LLMInterface,
        tts: TTSInterface,
        context: ContextManager,
    ):
        ...

    def run_turn(self, audio: bytes) -> None:
        text = self.stt.transcribe(audio)
        context = self.context.get_messages()
        response = self.llm.generate(text, context)
        self.context.add_turn(user=text, assistant=response)
        self.tts.speak(response)
```

### ContextManager

**Tanggung jawab:** Menyimpan dan menyediakan conversation history untuk setiap LLM call.

**Mengapa dipisahkan dari Orchestrator?**

Karena history adalah *state yang persisten dalam satu sesi*, sedangkan Orchestrator adalah *flow controller yang stateless*. Jika history ada di dalam Orchestrator, maka mengganti Orchestrator berarti kehilangan history. Memisahkan keduanya membuat masing-masing bisa berubah secara independen.

Di v0.1, `ContextManager` hanya menyimpan history di memory (in-process). Di v0.3, implementasinya akan berubah untuk mendukung long-term memory — tapi antarmukanya (`get_messages()`, `add_turn()`) tidak akan berubah.

### LLMInterface

**Tanggung jawab:** Mendefinisikan kontrak yang harus dipenuhi oleh semua LLM provider.

Menggunakan `Protocol` (bukan `ABC`) karena lebih idiomatik di Python modern dan tidak memaksa inheritance — sehingga library pihak ketiga pun bisa digunakan sebagai provider tanpa perlu dimodifikasi.

```python
from typing import Protocol

class LLMInterface(Protocol):
    def generate(self, prompt: str, context: list[dict]) -> str:
        ...
```

Kontrak ini sangat sederhana. `Orchestrator` tidak perlu tahu lebih dari ini.

### OllamaProvider

**Tanggung jawab:** Mengimplementasikan `LLMInterface` menggunakan Ollama sebagai backend.

Ini satu-satunya tempat di seluruh sistem yang boleh mengimpor library Ollama. Jika Ollama berganti versi API atau digantikan teknologi lain, perubahan hanya terjadi di sini.

### STT (Speech-to-Text)

**Tanggung jawab:** Mengubah audio bytes menjadi string teks.

Di v0.1 menggunakan Whisper `tiny`. Seperti LLM, STT juga menggunakan abstraction layer sehingga bisa diganti di masa depan.

### TTSOutput (Text-to-Speech)

**Tanggung jawab:** Mengubah string teks menjadi output suara.

Di v0.1 menggunakan `pyttsx3` sebagai placeholder. Di v0.2 akan diganti dengan Kokoro TTS. Abstraction layer memastikan penggantian ini tidak mempengaruhi `Orchestrator`.

---

## Alur Data

### Alur Normal (Happy Path)

```
1. Mikrofon menghasilkan audio stream
2. VoiceInput menangkap dan mem-buffer audio
3. VoiceInput mengirim audio bytes ke Orchestrator

4. Orchestrator memanggil STT.transcribe(audio)
5. STT (Whisper) mengonversi audio → teks
6. STT mengembalikan string teks ke Orchestrator

7. Orchestrator memanggil ContextManager.get_messages()
8. ContextManager mengembalikan conversation history

9. Orchestrator membangun prompt lengkap:
   [MASTER_PROMPT] + [conversation history] + [user input]

10. Orchestrator memanggil LLMInterface.generate(prompt, context)
11. LLMInterface meneruskan ke OllamaProvider
12. OllamaProvider mengirim request ke Ollama (lokal)
13. Ollama (Gemma 3 1B) menghasilkan response teks
14. Response teks dikembalikan ke Orchestrator

15. Orchestrator memanggil ContextManager.add_turn(user, assistant)
16. ContextManager menyimpan giliran percakapan terbaru

17. Orchestrator memanggil TTSOutput.speak(response)
18. TTSOutput (pyttsx3) mengonversi teks → suara
19. Speaker mengeluarkan suara
```

### Tentang MASTER_PROMPT

MASTER_PROMPT adalah sistem prompt permanen yang mendefinisikan personality Aira. Ia selalu ada di posisi pertama dari setiap LLM call — sebelum context history, sebelum user input.

MASTER_PROMPT tidak disimpan di `ContextManager` karena ia bukan bagian dari percakapan. Ia adalah *identitas* yang menetap, bukan *konten* yang berubah.

```python
# Struktur pesan yang dikirim ke LLM
[
    {"role": "system",    "content": MASTER_PROMPT},       # Selalu ada
    {"role": "user",      "content": "pesan pertama..."},  # History
    {"role": "assistant", "content": "balasan pertama..."},
    {"role": "user",      "content": "pesan terbaru"},     # Input saat ini
]
```

---

## Abstraction Layer

Ini adalah salah satu keputusan terpenting di seluruh arsitektur Aira.

### Mengapa Abstraction Layer?

Tanpa abstraction layer, kode aplikasi menjadi:

```python
# Tightly coupled — JANGAN dilakukan
import ollama

response = ollama.chat(
    model="gemma3:1b",
    messages=context
)
```

Kode di atas berarti seluruh sistem mengetahui bahwa LLM yang digunakan adalah Ollama dengan model Gemma. Jika besok kamu ingin mencoba OpenAI, kamu harus mencari dan mengubah kode ini di setiap tempat ia muncul.

Dengan abstraction layer:

```python
# Loosely coupled — yang benar
response = self.llm.generate(prompt, context)
```

`self.llm` bisa berisi apapun — selama ia memenuhi kontrak `LLMInterface`. Pergantian provider hanya membutuhkan satu perubahan di *dependency injection*, bukan di logika aplikasi.

### Bagaimana Provider Dipilih?

Di v0.1, pemilihan provider dilakukan satu kali di titik masuk aplikasi (`main.py`):

```python
# main.py — satu-satunya tempat yang tahu provider yang digunakan
from aira.providers.llm.ollama_provider import OllamaProvider
from aira.core.orchestrator import Orchestrator

llm = OllamaProvider(model="gemma3:1b")
orchestrator = Orchestrator(llm=llm, ...)
```

Seluruh kode di bawah `main.py` tidak pernah menyebut nama provider. Mereka hanya berbicara dengan antarmuka.

---

## Folder Structure

```
aira/
│
├── main.py                     # Entry point, dependency injection
│
├── core/                       # Inti sistem — tidak bergantung pada library eksternal
│   ├── orchestrator.py         # Pengatur alur pipeline
│   ├── context_manager.py      # Pengelola conversation history
│   └── session.py              # State satu sesi percakapan
│
├── interfaces/                 # Kontrak abstrak (Protocol definitions)
│   ├── llm_interface.py        # Kontrak untuk semua LLM provider
│   ├── stt_interface.py        # Kontrak untuk semua STT provider
│   └── tts_interface.py        # Kontrak untuk semua TTS provider
│
├── providers/                  # Implementasi konkret dari interfaces
│   ├── llm/
│   │   ├── ollama_provider.py  # Implementasi: Ollama + Gemma 3 1B
│   │   ├── openai_provider.py  # (future) Implementasi: OpenAI
│   │   └── gemini_provider.py  # (future) Implementasi: Gemini
│   ├── stt/
│   │   └── whisper_provider.py # Implementasi: Whisper tiny
│   └── tts/
│       ├── pyttsx3_provider.py # Implementasi v0.1: pyttsx3
│       └── kokoro_provider.py  # (future) Implementasi v0.2: Kokoro
│
├── pipeline/                   # Komponen pipeline (jembatan ke dunia luar)
│   ├── voice_input.py          # Capture audio dari mikrofon
│   └── tts_output.py           # Output suara ke speaker
│
├── personality/                # Definisi karakter Aira
│   └── master_prompt.py        # MASTER_PROMPT — identitas permanen Aira
│
├── config/                     # Konfigurasi sistem
│   └── settings.py             # Environment variables, model names, parameters
│
└── tests/                      # Unit & integration tests
    ├── test_orchestrator.py
    ├── test_context_manager.py
    └── test_providers/
        └── test_ollama_provider.py
```

### Mengapa Struktur Ini?

**`core/` tidak mengimpor dari `providers/`**
Ini adalah aturan paling penting. `orchestrator.py` tidak boleh memiliki `import ollama` atau `import openai`. Core hanya mengimpor dari `interfaces/`. Ini yang memastikan core bisa ditest tanpa perlu Ollama berjalan.

**`interfaces/` tidak mengimpor apapun selain stdlib**
Protocol definitions harus bersih dari dependency eksternal. Mereka adalah kontrak, bukan implementasi.

**`providers/` mengimpor library eksternal**
Ini satu-satunya tempat yang boleh mengimpor `ollama`, `openai`, `whisper`, dan sebagainya. Ketika library ini berubah versi atau API-nya, hanya folder ini yang perlu diupdate.

**`personality/` terpisah dari `core/`**
MASTER_PROMPT bukan logika bisnis — ia adalah identitas. Memisahkannya membuatnya mudah ditemukan, mudah diupdate, dan tidak tersembunyi di dalam orchestrator.

---

## Keputusan Arsitektur

Setiap keputusan arsitektur yang signifikan didokumentasikan secara terpisah dalam format ADR (Architecture Decision Record). Berikut ringkasannya:

| ID | Keputusan | Status |
|---|---|---|
| ADR-001 | Menggunakan Python sebagai bahasa utama | Accepted |
| ADR-002 | Menggunakan Protocol untuk abstraction layer LLM | Accepted |
| ADR-003 | Orchestrator sebagai satu titik kontrol pipeline | Accepted |
| ADR-004 | ContextManager sebagai komponen terpisah | Accepted |
| ADR-005 | pyttsx3 sebagai TTS placeholder di v0.1 | Accepted |
| ADR-006 | Whisper tiny sebagai STT di v0.1 | Accepted |
| ADR-007 | Ollama + Gemma 3 1B sebagai LLM di v0.1 | Accepted |

Dokumen ADR lengkap tersedia di `docs/adr/`.

---

## Batasan v0.1

Arsitektur ini dirancang dengan sadar untuk v0.1. Ada batasan yang bukan kekurangan, melainkan keputusan yang disengaja:

**Tidak ada konkurensi**
Voice Input dan TTS Output tidak berjalan paralel. Sistem berjalan secara sekuensial: dengar → proses → bicara. Ini menyederhanakan debugging dan mengurangi risiko race condition di fase fondasi.

**Context hanya in-memory**
Conversation history hilang ketika aplikasi ditutup. Long-term memory direncanakan di v0.3 setelah fondasi terbukti stabil.

**Tidak ada error recovery otomatis**
Jika Ollama tidak merespons atau Whisper gagal, sistem akan raise exception. Error handling yang lebih sophisticated akan ditambahkan setelah happy path berjalan sempurna.

**Satu sesi aktif**
v0.1 tidak mendukung multiple concurrent sessions. Aira berbicara dengan satu pengguna dalam satu waktu.

---

## Evolusi Arsitektur

Arsitektur ini dirancang untuk tumbuh. Berikut peta evolusinya:

```
v0.1  Fondasi
      └── Pipeline linear: Voice → STT → LLM → TTS → Speaker
          Orchestrator, ContextManager, abstraction layer

v0.2  Identitas Suara
      └── Ganti TTSProvider: pyttsx3 → Kokoro TTS
          Zero perubahan di core/, hanya tambah providers/tts/kokoro_provider.py

v0.3  Ingatan
      └── Upgrade ContextManager: in-memory → persistent storage
          Interface tidak berubah, implementasi dalam berubah

v0.4  Kesadaran Konteks
      └── Tambah Vision Pipeline: Screenshot → VisionInterface → Multimodal LLM
          Orchestrator diperluas, bukan dirombak

v0.5+ Agen
      └── Tambah AgentLayer di atas Orchestrator
          Orchestrator tetap ada sebagai execution engine
          AgentLayer menambahkan planning, tool use, dan reasoning loop
```

Perhatikan polanya: setiap versi *menambahkan* komponen baru atau *mengganti implementasi* di dalam Provider Layer — tidak pernah merombak Core atau Interface Layer. Inilah yang dimaksud dengan arsitektur yang tumbuh secara organik.

---

## Prinsip yang Tidak Boleh Dilanggar

Terlepas dari versi mana yang sedang dikembangkan:

1. **`core/` tidak pernah mengimpor dari `providers/`**
2. **`interfaces/` tidak pernah mengimpor library eksternal**
3. **`Orchestrator` tidak pernah menyebut nama provider secara eksplisit**
4. **MASTER_PROMPT tidak pernah dimodifikasi saat runtime**
5. **Setiap keputusan arsitektur yang baru harus didokumentasikan sebagai ADR**

---

*Dokumen ini adalah living document. Diagram dan struktur folder akan diperbarui setiap kali ada keputusan arsitektur yang signifikan.*

*Last updated: 2025 | Version: 1.0.0*
