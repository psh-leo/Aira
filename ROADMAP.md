# ROADMAP — Project Aira

> *"Aira tidak dibangun dalam semalam. Ia tumbuh melalui kepercayaan yang dibangun satu versi dalam satu waktu."*

---

## Filosofi Versioning

Setiap versi Aira menjawab satu pertanyaan fundamental:

> *"Apa yang bisa dipercayakan kepada Aira setelah versi ini?"*

Versi bukan tentang fitur yang ditambahkan. Versi tentang **kepercayaan yang dibangun**. Aira tidak mendapatkan kemampuan baru sebelum kemampuan sebelumnya terbukti stabil, konsisten, dan dapat diandalkan.

Ini juga berarti: **tidak ada fitur yang dibangun di atas fondasi yang rapuh.** Jika v0.1 belum stabil, v0.2 tidak dimulai.

---

## Konvensi Versi

```
v0.x  →  Fase fondasi dan pembentukan karakter
v1.x  →  Fase produk yang dapat digunakan sehari-hari
v2.x  →  Fase agen yang dapat bertindak atas nama Pasha
```

Setiap minor version (`0.1`, `0.2`, dst) adalah milestone yang dapat di-demo dan diverifikasi secara independen. Setiap patch version (`0.1.1`, `0.1.2`, dst) adalah perbaikan dalam milestone yang sama.

---

## Peta Perjalanan

```
                        AIRA ROADMAP
                        ============

v0.1  ──────────────────────────────────  FONDASI
      Voice Input + STT + LLM + TTS
      Pipeline berjalan end-to-end
      Personality konsisten
      Arsitektur abstraction layer siap

      ↓ (fondasi terbukti stabil)

v0.2  ──────────────────────────────────  IDENTITAS SUARA
      Kokoro TTS menggantikan pyttsx3
      Aira punya suara yang benar-benar miliknya

      ↓ (suara konsisten, personality terasa nyata)

v0.3  ──────────────────────────────────  INGATAN
      Long-term memory: Aira mulai mengenal Pasha
      Persistent conversation history
      Context yang bertahan antar sesi

      ↓ (Aira ingat siapa Pasha)

v0.4  ──────────────────────────────────  KESADARAN KONTEKS
      Vision: Aira bisa melihat layar
      Screenshot → multimodal LLM
      Aira memahami apa yang sedang Pasha kerjakan

      ↓ (Aira memahami konteks visual)

v0.5  ──────────────────────────────────  TINDAKAN DASAR
      Tool use: Aira bisa membuka aplikasi
      File system access
      System commands sederhana

      ↓ (Aira bisa bertindak, bukan hanya berbicara)

v1.0  ──────────────────────────────────  PRODUK
      Integrasi semua komponen v0.x
      Stability & performance tuning
      Instalasi yang mudah
      Aira siap digunakan sehari-hari

      ↓ (Aira adalah daily companion yang dapat diandalkan)

v2.0  ──────────────────────────────────  AGEN
      Planning & reasoning loop
      Multi-step task execution
      Plugin & skill system
      Aira dapat menyelesaikan tugas kompleks secara otonom
```

---

## Detail Per Versi

---

### v0.1 — Fondasi

**Status:** 🔄 In Development
**Prioritas:** Highest

#### Pertanyaan yang dijawab versi ini
*"Bisakah Aira mendengar, berpikir, dan berbicara?"*

#### Scope

Membangun pipeline percakapan end-to-end yang berfungsi dengan benar secara arsitektur. Kualitas respons dan kualitas suara bukan prioritas di versi ini — yang paling penting adalah setiap komponen bekerja sesuai perannya dan abstraction layer sudah siap untuk pertumbuhan.

#### Fitur

- Voice Input via mikrofon
- Speech-to-Text menggunakan Whisper tiny (lokal, offline)
- Percakapan natural dengan Gemma 3 1B via Ollama
- Text-to-Speech menggunakan pyttsx3 (placeholder)
- Personality Aira yang konsisten melalui MASTER_PROMPT
- Conversation history dalam satu sesi (in-memory)
- Abstraction layer untuk LLM, STT, dan TTS

#### Kriteria Selesai (Definition of Done)

- [ ] Pasha bisa berbicara dengan Aira menggunakan suara
- [ ] Aira merespons dalam Bahasa Indonesia yang natural
- [ ] Personality konsisten di seluruh percakapan
- [ ] Mengganti LLM provider hanya membutuhkan perubahan di `main.py`
- [ ] Semua komponen memiliki unit test dasar
- [ ] Tidak ada circular import antar modul

#### Yang Tidak Ada di Versi Ini (By Design)

- Suara Aira yang natural (ditunda ke v0.2)
- Memory antar sesi (ditunda ke v0.3)
- Vision, automation, agent (ditunda ke v0.4+)

#### Stack

| Komponen | Teknologi |
|---|---|
| STT | Whisper tiny |
| LLM | Ollama + Gemma 3 1B |
| TTS | pyttsx3 (placeholder) |
| Runtime | Python 3.11+ |

---

### v0.2 — Identitas Suara

**Status:** 📋 Planned
**Prioritas:** High

#### Pertanyaan yang dijawab versi ini
*"Apakah Aira terdengar seperti dirinya sendiri?"*

#### Scope

Mengganti TTS placeholder (pyttsx3) dengan Kokoro TTS — model neural TTS open-source yang berjalan lokal dan menghasilkan suara yang natural. Ini adalah versi di mana Aira pertama kali benar-benar *terdengar* seperti Aira.

#### Fitur

- Kokoro TTS menggantikan pyttsx3 sebagai TTS provider
- Voice selection dan konfigurasi pitch/speed
- Optimasi memory management untuk menjalankan Whisper + Ollama + Kokoro

#### Kriteria Selesai

- [ ] Suara Aira terdengar natural, tidak robotic
- [ ] Latency TTS acceptable (< 3 detik untuk respons pendek)
- [ ] Total RAM usage di bawah 3.5GB saat idle
- [ ] Pergantian TTS tidak membutuhkan perubahan di luar `providers/tts/`

#### Perubahan Arsitektur

Tidak ada perubahan arsitektur. Hanya penambahan `providers/tts/kokoro_provider.py` dan perubahan satu baris di `main.py`. Ini adalah bukti pertama bahwa abstraction layer yang dibangun di v0.1 bekerja.

---

### v0.3 — Ingatan

**Status:** 📋 Planned
**Prioritas:** High

#### Pertanyaan yang dijawab versi ini
*"Apakah Aira ingat siapa Pasha?"*

#### Scope

Mengimplementasikan long-term memory — kemampuan Aira untuk mengingat konteks, preferensi, dan informasi tentang Pasha yang bertahan antar sesi. Ini adalah versi di mana hubungan antara Pasha dan Aira mulai memiliki kedalaman.

#### Fitur

- Persistent conversation history (survive restart)
- User profile: Aira menyimpan informasi penting tentang Pasha
- Semantic memory: Aira bisa mencari informasi relevan dari memory sebelumnya
- Memory management: strategi untuk menghindari context window overflow

#### Kriteria Selesai

- [ ] Aira ingat konteks percakapan dari sesi sebelumnya
- [ ] Aira ingat fakta-fakta penting tentang Pasha yang pernah disebutkan
- [ ] Memory tidak membuat LLM call menjadi terlalu lambat
- [ ] Ada mekanisme untuk menghapus atau mengoreksi memory yang salah

#### Perubahan Arsitektur

`ContextManager` di-upgrade dari in-memory ke persistent storage. Interface `ContextManager` tidak berubah — hanya implementasi internalnya. Ini adalah bukti kedua bahwa desain interface yang benar di v0.1 memberikan hasil nyata.

---

### v0.4 — Kesadaran Konteks

**Status:** 📋 Planned
**Prioritas:** Medium

#### Pertanyaan yang dijawab versi ini
*"Apakah Aira tahu apa yang sedang terjadi di layar Pasha?"*

#### Scope

Menambahkan kemampuan vision — Aira bisa mengambil screenshot, memahami isinya, dan merespons berdasarkan konteks visual. Ini memungkinkan Pasha untuk bertanya tentang sesuatu yang ada di layar tanpa perlu mendeskripsikannya secara verbal.

#### Fitur

- Screenshot capture on-demand
- Vision pipeline: Screenshot → VisionInterface → Multimodal LLM
- Kemampuan Aira untuk memahami dan mengomentari isi layar
- Integrasi dengan conversation context

#### Stack Tambahan

| Komponen | Teknologi (kandidat) |
|---|---|
| Screenshot | `mss` atau `Pillow` |
| Vision LLM | LLaVA via Ollama atau Gemini API |

---

### v0.5 — Tindakan Dasar

**Status:** 📋 Planned
**Prioritas:** Medium

#### Pertanyaan yang dijawab versi ini
*"Bisakah Aira melakukan sesuatu, bukan hanya berbicara?"*

#### Scope

Menambahkan kemampuan tool use — Aira bisa mengeksekusi aksi sederhana di sistem operasi: membuka aplikasi, membaca file, menulis catatan, mengeksekusi command dasar.

#### Fitur

- Tool use framework
- File system read/write
- Application launcher
- System command executor (dengan batas keamanan yang ketat)
- Konfirmasi eksplisit sebelum setiap tindakan yang irreversible

#### Catatan Keamanan

Setiap tindakan yang mengubah state sistem (menulis file, menjalankan command) harus melalui konfirmasi eksplisit dari Pasha. Aira tidak pernah bertindak secara otonom tanpa persetujuan pada fase ini.

---

### v1.0 — Produk

**Status:** 📋 Planned
**Prioritas:** Medium-Low (jauh ke depan)

#### Pertanyaan yang dijawab versi ini
*"Apakah Aira siap digunakan setiap hari?"*

#### Scope

Bukan penambahan fitur baru — ini adalah versi yang menyatukan dan menstabilkan semua yang dibangun di v0.x. Fokus pada reliability, performance, dan kemudahan instalasi.

#### Fitur

- Installer yang mudah digunakan
- Auto-startup dengan sistem operasi
- Error recovery yang graceful
- Logging yang komprehensif
- Performance tuning untuk hardware terbatas
- Dokumentasi pengguna yang lengkap

---

### v2.0 — Agen

**Status:** 💭 Vision
**Prioritas:** Low (sangat jauh ke depan)

#### Pertanyaan yang dijawab versi ini
*"Bisakah Aira menyelesaikan tugas kompleks secara otonom?"*

#### Scope

Ini adalah visi jangka panjang Aira sebagai AI Agent — entitas yang tidak hanya merespons, tapi juga merencanakan, mengeksekusi, dan mengevaluasi hasil secara mandiri.

#### Konsep Fitur

- Planning & reasoning loop (ReAct atau similar)
- Multi-step task execution
- Plugin system
- Skill system yang extensible
- Browser control
- Computer control yang lebih luas

#### Catatan

Detail v2.0 belum didefinisikan secara konkret karena terlalu banyak hal yang akan berubah antara sekarang dan saat v2.0 relevan untuk dibangun. Dokumen ini akan diperbarui ketika v1.0 selesai.

---

## Prinsip yang Memandu Roadmap Ini

### Urutan adalah argumen

Urutan versi ini bukan arbitrary. Ia mencerminkan dependensi yang nyata:

- Aira tidak bisa dikenali (v0.3) sebelum ia bisa berbicara dengan natural (v0.2)
- Aira tidak bisa bertindak (v0.5) sebelum ia bisa melihat konteks (v0.4)
- Aira tidak bisa menjadi agen (v2.0) sebelum ia bisa diandalkan setiap hari (v1.0)

Jika ada desakan untuk melompati urutan ini, pertanyaan pertama yang harus diajukan adalah: *"Fondasi mana yang kita asumsikan sudah cukup baik, dan apakah asumsi itu terbukti?"*

### Setiap versi harus bisa di-demo

Tidak ada versi yang selesai jika tidak bisa didemonstrasikan secara end-to-end. "Hampir selesai" bukan milestone.

### Fondasi lebih penting dari fitur

Jika ada pilihan antara menambahkan fitur baru dan memperkuat fondasi yang ada, pilihan yang benar hampir selalu adalah memperkuat fondasi.

---

## Catatan untuk Masa Depan

Roadmap ini adalah dokumen hidup. Setiap kali ada keputusan yang mengubah scope, timeline, atau urutan versi, perubahan itu harus didokumentasikan di sini beserta alasannya.

Yang tidak boleh berubah tanpa diskusi mendalam:
- Urutan versi (dependensi antar versi adalah argumen teknis, bukan preferensi)
- Definition of Done per versi (standar yang diturunkan adalah tanda fondasi yang lemah)
- Prinsip yang memandu roadmap ini

*Last updated: 2026 | Version: 1.0.0*
