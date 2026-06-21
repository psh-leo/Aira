# ADR — Architecture Decision Records
# Project Aira

> *"Keputusan yang tidak didokumentasikan adalah keputusan yang akan diulang."*

ADR adalah catatan permanen tentang keputusan arsitektur yang signifikan: konteks yang memaksanya, pilihan yang tersedia, dan alasan di balik keputusan yang diambil.

Setiap ADR tidak pernah dihapus. Jika sebuah keputusan berubah, ADR lama diberi status **Superseded** dan ADR baru ditambahkan.

---

## Daftar ADR

| ID | Judul | Status | Versi |
|---|---|---|---|
| [ADR-001](#adr-001) | Python sebagai bahasa utama | Accepted | v0.1 |
| [ADR-002](#adr-002) | Protocol sebagai mekanisme abstraction layer | Accepted | v0.1 |
| [ADR-003](#adr-003) | Orchestrator sebagai satu titik kontrol pipeline | Accepted | v0.1 |
| [ADR-004](#adr-004) | ContextManager sebagai komponen independen | Accepted | v0.1 |
| [ADR-005](#adr-005) | pyttsx3 sebagai TTS placeholder di v0.1 | Accepted | v0.1 |
| [ADR-006](#adr-006) | Whisper tiny sebagai STT di v0.1 | Accepted | v0.1 |
| [ADR-007](#adr-007) | Ollama + Gemma 3 1B sebagai LLM di v0.1 | Accepted | v0.1 |

---

## ADR-001

### Python sebagai Bahasa Utama

**Status:** Accepted
**Tanggal:** 2025
**Versi:** v0.1

---

#### Konteks

Aira membutuhkan bahasa pemrograman yang dapat mengintegrasikan tiga domain sekaligus: pemrosesan audio (speech-to-text, text-to-speech), large language model inference, dan antarmuka sistem operasi (kontrol window, file system, automation). Di masa depan, Aira juga akan membutuhkan kemampuan computer vision dan agent tooling.

Keputusan ini juga harus mempertimbangkan konteks pengembang: mahasiswa semester 6 dengan background pemrograman dasar yang sedang belajar software engineering dan AI/ML secara paralel.

#### Keputusan

Python digunakan sebagai bahasa utama untuk seluruh sistem Aira.

#### Alasan

Python adalah lingua franca ekosistem AI/ML. Hampir semua library yang dibutuhkan Aira — Whisper, Ollama client, TTS engines, computer vision, LangChain, hingga agent frameworks — tersedia secara native di Python dengan dokumentasi yang matang.

Lebih dari itu, Python memiliki kurva belajar yang rendah untuk konsep-konsep yang akan menjadi fokus pembelajaran: clean architecture, abstraction, testing, dan design pattern. Ketika Pasha nanti mempelajari konsep seperti Protocol atau dependency injection, Python mengimplementasikannya dengan cara yang eksplisit dan mudah dibaca — berbeda dengan bahasa seperti Java yang menyembunyikan kompleksitas di balik boilerplate.

Selain itu, Python mendukung gradual typing melalui `typing` module. Ini artinya kita bisa menulis kode yang terasa dinamis di permukaan tapi memiliki kontrak yang jelas di level interface — persis seperti yang dibutuhkan untuk abstraction layer LLM.

#### Konsekuensi

*Positif:*
- Ekosistem AI/ML terlengkap — semua komponen Aira tersedia sebagai library Python yang matang.
- Kurva belajar yang manageable — Pasha bisa fokus pada konsep arsitektur, bukan sintaks bahasa.
- Komunitas dan dokumentasi yang sangat luas.
- Gradual typing memungkinkan penulisan kode yang bersih sekaligus memiliki kontrak yang terdefinisi.

*Negatif:*
- Python lebih lambat dari compiled language (Go, Rust, C++). Untuk Aira dengan RAM 4GB, ini berarti kita harus lebih berhati-hati dalam manajemen memory dan menghindari loading semua model sekaligus.
- Global Interpreter Lock (GIL) membatasi true parallelism — ini akan menjadi pertimbangan ketika implementasi concurrent voice processing di versi mendatang.

#### Alternatif yang Dipertimbangkan

**Go** — Performa lebih baik dan memory footprint lebih kecil. Ditolak karena ekosistem AI/ML-nya belum matang; hampir semua library AI yang dibutuhkan Aira tidak tersedia secara native di Go.

**JavaScript/Node.js** — Ditolak karena meskipun ekosistemnya luas, tooling untuk AI inference dan audio processing jauh lebih terbatas dibanding Python.

**Rust** — Performa terbaik, memory usage terkecil. Ditolak karena kurva belajarnya sangat curam dan tidak sesuai dengan tujuan pembelajaran proyek ini.

---

## ADR-002

### Protocol sebagai Mekanisme Abstraction Layer

**Status:** Accepted
**Tanggal:** 2025
**Versi:** v0.1

---

#### Konteks

Salah satu prinsip utama Aira adalah tidak bergantung pada satu LLM provider. Ini berarti sistem membutuhkan mekanisme abstraksi yang memungkinkan pergantian provider (Ollama, OpenAI, Gemini, dan lainnya) tanpa mengubah kode di lapisan aplikasi.

Python menyediakan dua mekanisme utama untuk ini: Abstract Base Class (ABC) dan Protocol (structural subtyping). Keputusan ini menentukan mana yang digunakan dan mengapa.

#### Keputusan

Python `Protocol` (dari modul `typing`) digunakan sebagai mekanisme abstraction layer untuk LLM, STT, dan TTS interfaces — bukan Abstract Base Class (ABC).

#### Alasan

Protocol menggunakan structural subtyping, bukan nominal subtyping. Ini artinya sebuah class dianggap memenuhi kontrak Protocol cukup dengan memiliki method yang sesuai — tanpa perlu mewarisi atau mendaftarkan diri secara eksplisit.

Implikasi praktisnya sangat signifikan: jika di masa depan ada library pihak ketiga (misalnya SDK resmi dari provider LLM baru) yang sudah memiliki method `generate()` dengan signature yang kompatibel, library itu bisa langsung digunakan sebagai provider tanpa perlu membuat wrapper class. Dengan ABC, kita terpaksa membuat wrapper hanya untuk memenuhi syarat inheritance.

Protocol juga lebih idiomatik di Python modern (3.8+) dan bekerja lebih baik dengan static type checker seperti `mypy` — yang akan kita gunakan untuk memastikan kontrak interface terpenuhi saat development, bukan saat runtime.

```python
# Dengan Protocol — tidak ada inheritance, tidak ada boilerplate
from typing import Protocol

class LLMInterface(Protocol):
    def generate(self, prompt: str, context: list[dict]) -> str:
        ...

# OllamaProvider tidak perlu tahu tentang LLMInterface
# cukup implementasikan method yang sesuai
class OllamaProvider:
    def generate(self, prompt: str, context: list[dict]) -> str:
        # implementasi spesifik Ollama
        ...

# mypy akan memverifikasi bahwa OllamaProvider memenuhi LLMInterface
# error terdeteksi saat development, bukan saat runtime
```

#### Konsekuensi

*Positif:*
- Provider baru bisa ditambahkan tanpa mengubah Interface definition.
- Library pihak ketiga bisa digunakan langsung tanpa wrapper jika signature-nya kompatibel.
- Lebih idiomatik di Python modern dan bekerja lebih baik dengan tooling.
- Error ketidaksesuaian kontrak terdeteksi di development time oleh type checker.

*Negatif:*
- Protocol tidak enforce implementation secara runtime — jika developer lupa mengimplementasikan method, error baru muncul saat method itu dipanggil, bukan saat class di-instantiate.
- Kurang familiar bagi developer yang terbiasa dengan pola OOP tradisional (Java, C#).

*Mitigasi negatif:*
- Gunakan `mypy` dalam development workflow untuk mendeteksi pelanggaran kontrak lebih awal.
- Tambahkan unit test yang memverifikasi setiap provider dapat di-instantiate dan method-nya dapat dipanggil.

#### Alternatif yang Dipertimbangkan

**Abstract Base Class (ABC)** — Lebih familiar, enforce inheritance secara eksplisit, error muncul saat instantiation jika ada method abstrak yang belum diimplementasikan. Ditolak karena memaksa inheritance dan membuat integrasi library pihak ketiga lebih sulit.

**Duck typing murni (tanpa Protocol)** — Tidak ada kontrak formal sama sekali, bergantung sepenuhnya pada konvensi. Ditolak karena tidak memberikan keamanan di level type checking dan membuat kode lebih sulit dipahami dan di-maintain.

---

## ADR-003

### Orchestrator sebagai Satu Titik Kontrol Pipeline

**Status:** Accepted
**Tanggal:** 2025
**Versi:** v0.1

---

#### Konteks

Pipeline Aira terdiri dari beberapa komponen yang harus bekerja secara berurutan: VoiceInput → STT → LLM → TTS → Output. Ada dua pendekatan untuk mengatur urutan ini: setiap komponen memanggil komponen berikutnya secara langsung (chaining), atau ada satu komponen pusat yang mengatur seluruh alur (orchestration).

#### Keputusan

Satu komponen `Orchestrator` bertanggung jawab penuh atas urutan eksekusi pipeline. Tidak ada komponen lain yang memanggil komponen lain secara langsung.

#### Alasan

Dengan pendekatan chaining, setiap komponen harus tahu komponen apa yang ada di "sebelahnya". `VoiceInput` harus tahu tentang `STT`. `STT` harus tahu tentang `LLM`. Ini menciptakan coupling yang dalam — mengubah urutan pipeline berarti mengubah kode di setiap komponen yang terlibat.

Dengan Orchestrator, hanya satu tempat yang perlu diubah ketika alur berubah. Komponen-komponen lain tetap tidak berubah karena mereka tidak tahu siapa yang memanggil mereka. Ini juga membuat setiap komponen jauh lebih mudah ditest secara independen — tidak perlu mensimulasikan seluruh pipeline untuk mengtest satu komponen.

Pola ini dikenal sebagai Mediator Pattern — sebuah pola klasik yang dirancang persis untuk masalah ini: mengurangi ketergantungan antar komponen dengan memperkenalkan satu titik komunikasi sentral.

#### Konsekuensi

*Positif:*
- Alur pipeline mudah dipahami — cukup baca `orchestrator.py` untuk mengerti seluruh sistem.
- Setiap komponen bisa ditest secara independen tanpa mock yang kompleks.
- Menambah atau mengubah langkah pipeline hanya membutuhkan modifikasi di satu file.
- Logging dan error handling dapat dipusatkan di Orchestrator.

*Negatif:*
- Orchestrator berpotensi menjadi "God Object" jika tidak dijaga — semakin banyak logika yang ditaruh di dalamnya.
- Satu titik kegagalan: jika Orchestrator memiliki bug, seluruh pipeline terpengaruh.

*Mitigasi negatif:*
- Orchestrator hanya boleh mengatur *urutan* eksekusi, tidak pernah memproses konten.
- Semua logika bisnis tetap di komponen masing-masing.
- Jika Orchestrator mulai tumbuh lebih dari ~100 baris, itu sinyal bahwa ada logika yang harus dipindah ke komponen lain.

#### Alternatif yang Dipertimbangkan

**Direct Chaining** — Setiap komponen memanggil komponen berikutnya. Ditolak karena menciptakan tight coupling dan membuat testing sangat sulit.

**Event-driven Pipeline** — Komponen berkomunikasi melalui event bus, tidak ada yang memanggil siapapun secara langsung. Ditolak untuk v0.1 karena menambah kompleksitas yang tidak diperlukan di fase fondasi. Akan dipertimbangkan ulang di v0.5+ ketika sistem mulai memerlukan asynchronous processing.

---

## ADR-004

### ContextManager sebagai Komponen Independen

**Status:** Accepted
**Tanggal:** 2025
**Versi:** v0.1

---

#### Konteks

LLM adalah model stateless — ia tidak ingat percakapan sebelumnya kecuali jika history percakapan dikirimkan sebagai bagian dari setiap request. Aira membutuhkan mekanisme untuk menyimpan dan menyediakan conversation history ini.

Pertanyaannya: di mana history ini disimpan? Di dalam Orchestrator? Di dalam LLM Provider? Atau sebagai komponen terpisah?

#### Keputusan

Conversation history dikelola oleh `ContextManager` — sebuah komponen independen yang terpisah dari Orchestrator maupun LLM Provider.

#### Alasan

History percakapan adalah *state* — ia hidup lebih lama dari satu request dan memiliki lifecycle-nya sendiri. Menaruh state di dalam Orchestrator melanggar Single Responsibility Principle: Orchestrator seharusnya hanya mengatur alur, bukan menyimpan state. Menaruh state di dalam LLM Provider berarti state hilang ketika provider diganti.

`ContextManager` yang independen memiliki satu tanggung jawab yang jelas: mengelola percakapan. Ini membuatnya bisa berevolusi secara independen — di v0.1 ia menyimpan history di memory, di v0.3 ia bisa upgrade ke persistent storage (database, file) tanpa mengubah cara Orchestrator atau LLM Provider berinteraksi dengannya.

Antarmuka `ContextManager` sengaja dibuat sesederhana mungkin:

```python
class ContextManager:
    def get_messages(self) -> list[dict]:
        """Kembalikan seluruh history untuk dikirim ke LLM."""
        ...

    def add_turn(self, user: str, assistant: str) -> None:
        """Simpan satu giliran percakapan."""
        ...

    def clear(self) -> None:
        """Reset history — mulai sesi baru."""
        ...
```

Antarmuka yang sederhana ini tidak akan berubah meskipun implementasi di dalamnya berubah total di v0.3.

#### Konsekuensi

*Positif:*
- History percakapan tetap ada meskipun LLM Provider diganti.
- `ContextManager` bisa ditest secara independen tanpa membutuhkan LLM atau Orchestrator.
- Evolusi dari in-memory ke persistent storage tidak mempengaruhi komponen lain.
- Mudah menambahkan fitur seperti windowing (membatasi jumlah pesan yang dikirim ke LLM untuk menghemat token).

*Negatif:*
- Menambah satu komponen dan satu abstraksi yang perlu dipahami.
- Di v0.1 yang sederhana, ini mungkin terasa over-engineered.

*Mengapa tetap dipilih meskipun terasa over-engineered:*

Biaya memisahkan `ContextManager` sekarang adalah nol — hanya menambah satu file kecil. Biaya menggabungkannya ke dalam Orchestrator dan kemudian harus memisahkannya lagi di v0.3 ketika long-term memory dibutuhkan jauh lebih besar. Keputusan yang murah sekarang mencegah refactoring yang mahal nanti.

#### Alternatif yang Dipertimbangkan

**History di dalam Orchestrator** — Lebih sederhana di v0.1. Ditolak karena melanggar Single Responsibility dan akan mempersulit upgrade ke persistent storage di v0.3.

**History di dalam LLM Provider** — Paling sederhana, tidak perlu komponen tambahan. Ditolak karena history akan hilang setiap kali provider diganti — bertentangan dengan prinsip provider-agnostic architecture.

---

## ADR-005

### pyttsx3 sebagai TTS Placeholder di v0.1

**Status:** Accepted — akan di-supersede oleh ADR-008 di v0.2
**Tanggal:** 2025
**Versi:** v0.1

---

#### Konteks

Aira membutuhkan kemampuan text-to-speech untuk berbicara dengan pengguna. Kualitas suara adalah bagian penting dari personality Aira — suara yang robotic akan merusak pengalaman percakapan. Namun di v0.1, prioritas utama adalah membangun fondasi arsitektur yang benar, bukan mengoptimalkan pengalaman.

Dua kandidat yang dipertimbangkan untuk TTS: `pyttsx3` (offline, ringan, suara robotic) dan Kokoro TTS (offline, model neural, suara natural, ~500MB-1GB RAM).

#### Keputusan

`pyttsx3` digunakan sebagai TTS di v0.1 sebagai placeholder sementara yang akan digantikan oleh Kokoro TTS di v0.2.

#### Alasan

Di v0.1, semua komponen berjalan bersamaan di RAM 4GB:

```
Whisper tiny    ≈ 400MB
Ollama Gemma 3  ≈ 800MB–1GB
OS + Python     ≈ 1.5GB
────────────────────────
Tersisa         ≈ 1.1GB–1.3GB
```

Menambahkan Kokoro TTS (~500MB–1GB) di atas ini membawa total penggunaan RAM ke batas yang sangat mepet — bahkan berpotensi melebihi kapasitas. Di fase fondasi, debugging masalah infrastruktur (RAM overflow, model loading error) akan mengalihkan fokus dari hal yang lebih penting: memverifikasi bahwa arsitektur pipeline sudah benar.

`pyttsx3` menggunakan speech engine yang sudah ada di OS (SAPI5 di Windows), sehingga tidak membutuhkan tambahan RAM yang signifikan. Ini memungkinkan v0.1 untuk memverifikasi seluruh pipeline end-to-end tanpa risiko keterbatasan hardware.

Keputusan ini direncanakan dan bukan kompromis permanen. Abstraction layer TTS sudah dirancang sejak awal untuk menerima Kokoro di v0.2 tanpa perubahan di kode di luar `providers/tts/`.

#### Konsekuensi

*Positif:*
- Tidak menambah beban RAM yang signifikan di v0.1.
- Setup sangat sederhana: `pip install pyttsx3`, langsung berjalan.
- Tidak ada model yang perlu di-download.
- Pipeline bisa ditest end-to-end tanpa risiko RAM overflow.

*Negatif:*
- Suara robotic — tidak mencerminkan personality Aira yang natural dan tidak terdengar seperti robot.
- Kualitas suara sangat bergantung pada voice pack yang terinstall di OS.

*Mitigasi:*
- `pyttsx3` hanya ada di `providers/tts/pyttsx3_provider.py` — terisolasi sepenuhnya.
- Upgrade ke Kokoro di v0.2 hanya membutuhkan penambahan `providers/tts/kokoro_provider.py` dan perubahan satu baris di `main.py`.

#### Alternatif yang Dipertimbangkan

**Kokoro TTS** — Kualitas suara terbaik untuk kasus ini, offline, open-source. Ditolak untuk v0.1 karena RAM footprint terlalu besar untuk dijalankan bersamaan dengan Whisper dan Ollama di RAM 4GB pada fase fondasi.

**ElevenLabs API** — Kualitas suara terbaik secara keseluruhan. Ditolak karena berbayar, membutuhkan internet, dan bertentangan dengan prinsip tidak bergantung pada satu provider eksternal.

**gTTS (Google TTS)** — Gratis, kualitas cukup baik. Ditolak karena membutuhkan koneksi internet setiap kali bicara — tidak sesuai dengan prinsip offline-first Aira.

---

## ADR-006

### Whisper Tiny sebagai STT di v0.1

**Status:** Accepted
**Tanggal:** 2025
**Versi:** v0.1

---

#### Konteks

Aira membutuhkan kemampuan speech-to-text yang akurat untuk Bahasa Indonesia — yang digunakan dalam percakapan sehari-hari antara Pasha dan Aira. Sistem harus berjalan offline untuk memastikan privasi dan tidak bergantung pada koneksi internet.

#### Keputusan

OpenAI Whisper model `tiny` digunakan sebagai STT di v0.1, berjalan secara lokal menggunakan library `openai-whisper`.

#### Alasan

Whisper adalah satu-satunya model STT yang secara konsisten memberikan akurasi tinggi untuk Bahasa Indonesia tanpa membutuhkan koneksi internet. Ini bukan kebetulan — Whisper dilatih pada dataset multilingual yang sangat besar dan secara eksplisit mendukung Bahasa Indonesia sebagai salah satu bahasa target.

Model `tiny` dipilih (bukan `base` atau `small`) karena trade-off RAM vs akurasi yang paling sesuai untuk spesifikasi perangkat:

| Model | RAM | Akurasi (relatif) |
|---|---|---|
| tiny | ~390MB | Cukup baik untuk percakapan |
| base | ~740MB | Lebih baik |
| small | ~1.5GB | Signifikan lebih baik |

Pada RAM 4GB dengan Ollama sudah menggunakan ~1GB, `tiny` adalah pilihan yang paling konservatif sambil tetap memberikan akurasi yang acceptable untuk percakapan normal. Jika di masa depan Aira berjalan di perangkat dengan RAM lebih besar, upgrade ke `base` atau `small` tidak memerlukan perubahan arsitektur — hanya perubahan konfigurasi.

#### Konsekuensi

*Positif:*
- Offline sepenuhnya — tidak ada data percakapan yang dikirim ke server manapun.
- Akurasi Bahasa Indonesia yang baik, termasuk code-switching (campuran Indonesia-Inggris) yang umum digunakan Pasha.
- Model bisa di-upgrade (tiny → base → small) hanya dengan mengubah satu nilai di konfigurasi.
- Satu library (`openai-whisper`) menangani seluruh pipeline audio processing.

*Negatif:*
- ~390MB RAM — terbesar kedua setelah Ollama.
- Latensi: model `tiny` di CPU membutuhkan waktu untuk transkripsi, terutama untuk audio yang lebih panjang.
- Tidak ada streaming transcription di v0.1 — Aira harus selesai mendengar sebelum mulai memproses.

#### Alternatif yang Dipertimbangkan

**SpeechRecognition + Google** — Ringan, tidak perlu model lokal. Ditolak karena membutuhkan internet dan mengirim audio ke server Google — bertentangan dengan privasi dan prinsip offline-first.

**Vosk** — Model offline yang lebih ringan dari Whisper. Ditolak karena akurasi Bahasa Indonesia-nya jauh lebih rendah dari Whisper, terutama untuk percakapan natural.

**Azure Speech Services / AWS Transcribe** — Akurasi sangat tinggi. Ditolak karena berbayar, membutuhkan internet, dan menciptakan ketergantungan pada cloud provider.

---

## ADR-007

### Ollama + Gemma 3 1B sebagai LLM di v0.1

**Status:** Accepted
**Tanggal:** 2025
**Versi:** v0.1

---

#### Konteks

Aira membutuhkan large language model untuk menghasilkan respons percakapan yang natural, konsisten dengan personality, dan relevan dengan konteks. Pemilihan LLM di v0.1 harus mempertimbangkan tiga constraint utama: RAM 4GB, CPU tanpa GPU dedicated, dan kebutuhan untuk berjalan offline.

#### Keputusan

Ollama digunakan sebagai runtime untuk menjalankan model Gemma 3 1B secara lokal. Ollama diakses melalui REST API yang berjalan di localhost.

#### Alasan

**Mengapa Ollama?**

Ollama adalah cara paling mudah untuk menjalankan model bahasa secara lokal di Windows. Ia menangani semua kompleksitas: download model, quantization, memory management, dan menyediakan REST API yang konsisten. Tanpa Ollama, kita harus mengelola semua ini secara manual menggunakan library seperti `llama-cpp-python` — yang memiliki dependency C++ yang kompleks dan sering bermasalah di Windows.

Ollama juga selaras dengan abstraction layer yang sudah diputuskan di ADR-002: `OllamaProvider` hanya perlu memanggil REST API di `localhost:11434`, tanpa perlu mengimpor SDK yang berat.

**Mengapa Gemma 3 1B?**

Gemma 3 1B adalah model terkecil dari keluarga Gemma 3 yang tetap memiliki kemampuan percakapan yang memadai. Dengan quantization Q4 (default Ollama), model ini membutuhkan sekitar 800MB–1GB RAM — menyisakan ruang yang cukup untuk Whisper dan sistem operasi.

Pertimbangan penting: Gemma 3 adalah model multilingual yang mendukung Bahasa Indonesia dengan cukup baik. Ini bukan model yang dioptimalkan untuk Bahasa Indonesia, tapi kemampuannya cukup untuk percakapan natural dalam konteks Aira v0.1.

**Tentang kualitas respons:**

Model 1B memiliki keterbatasan dibanding model yang lebih besar (7B, 13B, dan seterusnya). Respons mungkin kadang kurang koheren untuk pertanyaan yang kompleks. Namun di v0.1, kualitas respons bukan prioritas utama — validasi arsitektur pipeline adalah prioritasnya. Upgrade model bisa dilakukan di v0.2 atau ketika Pasha memiliki perangkat dengan RAM lebih besar, tanpa mengubah arsitektur.

#### Konsekuensi

*Positif:*
- Setup sangat mudah: install Ollama → `ollama pull gemma3:1b` → selesai.
- Berjalan sepenuhnya offline — tidak ada API key, tidak ada biaya per token.
- REST API Ollama yang konsisten memudahkan implementasi `OllamaProvider`.
- Mudah upgrade model (1B → 3B → 7B) hanya dengan mengubah konfigurasi.
- Mudah ganti provider (Ollama → OpenAI) karena abstraction layer sudah ada.

*Negatif:*
- Kualitas respons model 1B terbatas untuk pertanyaan kompleks.
- Latensi lebih tinggi dibanding API cloud (tapi tidak membutuhkan internet).
- Ollama harus berjalan sebagai background service sebelum Aira dijalankan.

#### Alternatif yang Dipertimbangkan

**OpenAI GPT-4o mini** — Kualitas respons jauh lebih baik, latensi rendah. Ditolak karena berbayar per token, membutuhkan internet, dan menciptakan ketergantungan pada satu provider eksternal.

**Google Gemini Flash** — Serupa dengan OpenAI dalam hal trade-off. Ditolak dengan alasan yang sama.

**llama-cpp-python langsung (tanpa Ollama)** — Lebih kontrol, tidak butuh Ollama sebagai service terpisah. Ditolak karena kompleksitas setup di Windows sangat tinggi (dependency C++, kompilasi) dan tidak sebanding dengan manfaat di fase fondasi.

**Mistral 7B via Ollama** — Kualitas lebih baik dari Gemma 3 1B. Ditolak karena 7B quantized membutuhkan ~4-5GB RAM — melebihi kapasitas perangkat dengan margin yang aman.

---

*Dokumen ini diperbarui setiap kali ada keputusan arsitektur baru.*
*ADR tidak pernah dihapus — hanya status-nya yang bisa berubah.*

*Last updated: 2025 | v0.1*
