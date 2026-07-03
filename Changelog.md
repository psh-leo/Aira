# CHANGELOG — Project Aira

Semua perubahan signifikan pada proyek ini didokumentasikan di sini.

Format mengikuti [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning mengikuti [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

### In Progress
- Implementasi core pipeline v0.1
- Setup project structure
- Unit tests untuk semua komponen

---

## [0.1.0] — TBD

### Added
- Voice Input via PyAudio (16kHz mono WAV capture)
- Speech-to-Text menggunakan Whisper tiny (lokal, offline)
- LLM conversation menggunakan Ollama + Gemma 3 1B (lokal, offline)
- Text-to-Speech menggunakan pyttsx3 (placeholder)
- Orchestrator sebagai satu titik kontrol pipeline
- ContextManager untuk conversation history (in-memory, per sesi)
- Abstraction layer (Protocol) untuk LLM, STT, dan TTS
- MASTER_PROMPT — definisi personality permanen Aira
- Konfigurasi terpusat via AiraConfig dataclass
- Custom exception hierarchy (AiraError dan turunannya)
- Dependency injection pattern via `build_aira()` factory function
- Dokumentasi lengkap: VISION, ARCHITECTURE, ADR, SYSTEM_DESIGN,
  TECH_STACK, ROADMAP, MASTER_PROMPT, README, DEVELOPMENT_GUIDE,
  CONTRIBUTING, CHANGELOG, LICENSE

### Technical Decisions
- Lihat [ADR.md](./docs/adr/ADR.md) untuk semua keputusan arsitektur v0.1

---

*Format entri CHANGELOG:*

*`### Added` — fitur baru*
*`### Changed` — perubahan pada fitur yang sudah ada*
*`### Deprecated` — fitur yang akan dihapus di versi mendatang*
*`### Removed` — fitur yang dihapus*
*`### Fixed` — bug fix*
*`### Security` — perbaikan keamanan*

*Last updated: 2025 | Version: 1.0.0*
