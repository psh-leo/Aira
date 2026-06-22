# Aira — Desktop AI Companion

> *Bukan asisten. Bukan chatbot. Partner berpikir yang selalu hadir.*

---

Aira adalah **Desktop AI Companion** yang berjalan sepenuhnya lokal di komputermu — tanpa koneksi internet, tanpa API key, tanpa biaya per token.

Ia mendengar suaramu, berpikir, dan berbicara kembali. Tapi Aira bukan sekadar voice assistant. Ia adalah partner berpikir: hadir, aktif, dan jujur bahkan ketika kejujuran itu berarti mengoreksimu.

---

## Status Proyek

```
v0.1 — Fondasi         🔄 In Development
v0.2 — Identitas Suara 📋 Planned
v0.3 — Ingatan         📋 Planned
v0.4 — Kesadaran       📋 Planned
v0.5 — Tindakan        📋 Planned
v1.0 — Produk          📋 Planned
```

---

## Demo Singkat

```
Pasha: "Aira, aku lagi bingung harus pilih arsitektur monolith atau microservices."

Aira:  "Tergantung skalanya, Pasha. Untuk proyek yang baru mulai,
        monolith hampir selalu pilihan yang lebih tepat — lebih mudah
        di-debug, lebih cepat iterasi, dan kamu bisa refactor ke
        microservices nanti kalau memang perlu. Microservices bukan
        solusi untuk masalah yang belum ada."
```

---

## Fitur v0.1

- **Voice Input** — Bicara langsung ke mikrofon, Aira mendengar
- **Voice Output** — Aira merespons dengan suara, bukan teks
- **Natural Conversation** — Percakapan multi-giliran dengan context history
- **Consistent Personality** — Karakter Aira tidak berubah sepanjang sesi
- **100% Offline** — Tidak ada data yang dikirim ke server manapun
- **Provider-agnostic** — Ganti LLM hanya dengan mengubah satu baris konfigurasi

---

## Arsitektur Singkat

```
[Mic] → VoiceInput → STT (Whisper) → Orchestrator → LLM (Ollama)
                                           ↕
                                     ContextManager
                                           ↓
[Speaker] ← TTS (pyttsx3) ←────────── Response
```

Setiap komponen terisolasi dan dapat diganti secara independen. Lihat [ARCHITECTURE.md](./ARCHITECTURE.md) untuk detail lengkap.

---

## Prasyarat

Sebelum instalasi, pastikan hal berikut sudah terpasang:

| Prasyarat | Versi | Keterangan |
|---|---|---|
| Python | 3.11+ | [python.org](https://python.org) |
| Ollama | Latest | [ollama.com](https://ollama.com) |
| ffmpeg | Latest | Dibutuhkan oleh Whisper |
| Git | Latest | Untuk clone repository |

---

## Instalasi Cepat

```bash
# 1. Clone repository
git clone https://github.com/username/aira.git
cd aira

# 2. Buat virtual environment
python -m venv .venv
.venv\Scripts\activate  # Windows

# 3. Install PyTorch CPU (penting: lakukan ini sebelum install Whisper)
pip install torch --index-url https://download.pytorch.org/whl/cpu

# 4. Install dependencies
pip install -r requirements.txt

# 5. Pull model LLM
ollama pull gemma3:1b

# 6. Jalankan Aira
python main.py
```

Panduan instalasi lengkap tersedia di [DEVELOPMENT_GUIDE.md](./docs/DEVELOPMENT_GUIDE.md).

---

## Dokumentasi

| Dokumen | Deskripsi |
|---|---|
| [VISION](./VISION.md) | Mengapa Aira ada |
| [ARCHITECTURE](./ARCHITECTURE.md) | Bagaimana sistem dirancang |
| [SYSTEM_DESIGN](./docs/SYSTEM_DESIGN.md) | Detail teknis dan interface contract |
| [TECH_STACK](./docs/TECH_STACK.md) | Teknologi yang digunakan |
| [ROADMAP](./ROADMAP.md) | Peta perkembangan Aira |
| [ADR](./docs/adr/ADR.md) | Architecture Decision Records |
| [MASTER_PROMPT](./docs/MASTER_PROMPT.md) | Definisi personality Aira |
| [DEVELOPMENT_GUIDE](./docs/DEVELOPMENT_GUIDE.md) | Panduan setup dan pengembangan |
| [CONTRIBUTING](./CONTRIBUTING.md) | Cara berkontribusi |
| [CHANGELOG](./CHANGELOG.md) | Riwayat perubahan |

---

## Tech Stack

| Komponen | Teknologi |
|---|---|
| Language | Python 3.11+ |
| STT | OpenAI Whisper tiny (lokal) |
| LLM | Ollama + Gemma 3 1B (lokal) |
| TTS | pyttsx3 → Kokoro (v0.2) |
| Type Checking | mypy |
| Linting | ruff |
| Testing | pytest |

---

## Filosofi Proyek

Aira dibangun dengan satu prinsip utama:

> *Fondasi yang baik selalu lebih berharga dari fitur yang cepat.*

Setiap keputusan teknis didokumentasikan dan dapat dijelaskan. Tidak ada magic, tidak ada shortcut yang menyembunyikan kompleksitas. Jika kamu membaca kode Aira dan bertanya "mengapa ini seperti ini?" — jawabannya ada di dokumentasi.

---

## Lisensi

MIT License — lihat [LICENSE](./LICENSE) untuk detail.

---

## Author

Dibuat oleh **Pasha** sebagai proyek pembelajaran software engineering dan AI/ML yang serius — dibangun untuk bertahan bertahun-tahun, bukan selesai dalam seminggu.
