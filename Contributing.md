# CONTRIBUTING — Project Aira

Terima kasih sudah tertarik berkontribusi pada Aira.

Dokumen ini menjelaskan bagaimana cara berkontribusi dengan benar — bukan hanya secara teknis, tapi juga secara filosofis sesuai dengan nilai-nilai proyek ini.

---

## Baca Ini Sebelum Apapun

Sebelum membuat Pull Request atau Issue, baca dokumen berikut:

1. [VISION.md](./VISION.md) — Pahami *mengapa* Aira ada
2. [ARCHITECTURE.md](./ARCHITECTURE.md) — Pahami struktur sistem
3. [ADR.md](./docs/adr/ADR.md) — Pahami keputusan yang sudah diambil

Kontribusi yang bertentangan dengan prinsip di dokumen-dokumen di atas akan ditolak — bukan karena kodenya buruk, tapi karena tidak sesuai dengan arah proyek.

---

## Jenis Kontribusi yang Diterima

### ✅ Diterima

- Bug fix dengan reproduction case yang jelas
- Peningkatan dokumentasi (klarifikasi, koreksi, contoh tambahan)
- Unit test untuk komponen yang belum ter-cover
- Peningkatan performa yang tidak mengubah arsitektur
- Provider baru (LLM, STT, TTS) yang mengimplementasikan interface yang ada

### ❌ Tidak Diterima (untuk saat ini)

- Fitur yang belum ada di ROADMAP tanpa diskusi terlebih dahulu
- Perubahan yang melanggar prinsip arsitektur (mis. `core/` mengimpor `providers/`)
- Penambahan dependency baru tanpa ADR yang menyertai
- Perubahan MASTER_PROMPT tanpa diskusi

---

## Workflow Kontribusi

### 1. Buat Issue Dulu

Sebelum menulis kode, buat Issue yang menjelaskan:
- Apa masalahnya atau apa yang ingin ditambahkan
- Mengapa ini penting untuk proyek
- Pendekatan solusi yang diusulkan (jika sudah ada gambaran)

Ini menghindari situasi di mana kamu sudah menulis banyak kode tapi arahnya tidak sesuai dengan visi proyek.

### 2. Fork dan Clone

```bash
git clone https://github.com/username/aira.git
cd aira
```

### 3. Buat Branch

```bash
# Format: type/short-description
git checkout -b fix/whisper-timeout-handling
git checkout -b feat/openai-provider
git checkout -b docs/improve-architecture-diagram
```

### 4. Kerjakan Perubahan

Ikuti panduan di [DEVELOPMENT_GUIDE.md](./docs/DEVELOPMENT_GUIDE.md).

### 5. Pastikan Semua Check Lulus

```bash
ruff check aira/
ruff format aira/
mypy aira/ --strict
pytest
```

### 6. Commit dengan Pesan yang Jelas

```bash
# Format: type(scope): description
git commit -m "fix(whisper): handle timeout when audio file is too long"
git commit -m "feat(providers): add OpenAI provider implementation"
git commit -m "docs(adr): add ADR-008 for Kokoro TTS upgrade"
```

**Tipe commit yang valid:**
- `feat` — fitur baru
- `fix` — bug fix
- `docs` — perubahan dokumentasi
- `test` — menambah atau memperbaiki test
- `refactor` — refactoring tanpa perubahan fungsional
- `chore` — perubahan konfigurasi, dependency, dll

### 7. Buat Pull Request

Judul PR mengikuti format commit message. Deskripsi PR harus menjelaskan:
- Apa yang berubah
- Mengapa perubahan ini perlu
- Bagaimana cara mengujinya
- Issue mana yang di-resolve (jika ada)

---

## Standar Kode

### Aturan yang Tidak Boleh Dilanggar

```
❌ core/ tidak boleh mengimpor dari providers/
❌ interfaces/ tidak boleh mengimpor library eksternal
❌ Tidak ada magic number — semua nilai ada di config/settings.py
❌ Tidak ada print() di kode produksi — gunakan logging
❌ Tidak ada bare except: — selalu tangkap exception yang spesifik
```

### Hal yang Diharapkan

```
✅ Semua public method memiliki docstring
✅ Semua function memiliki type annotation
✅ Semua Provider baru memiliki unit test dasar
✅ Setiap dependency baru disertai alasan di PR description
✅ ADR baru jika keputusan arsitektur berubah
```

---

## Menambahkan Provider Baru

Ini adalah kontribusi yang paling umum dan paling mudah diterima.

**Contoh: Menambahkan OpenAI Provider**

```python
# providers/llm/openai_provider.py
from aira.interfaces.llm_interface import LLMInterface  # hanya untuk dokumentasi


class OpenAIProvider:
    """
    LLM Provider menggunakan OpenAI API.

    Mengimplementasikan LLMInterface (diverifikasi oleh mypy).
    """

    def __init__(self, api_key: str, model: str = "gpt-4o-mini"):
        import openai
        self._client = openai.OpenAI(api_key=api_key)
        self._model = model

    def generate(self, messages: list[dict[str, str]]) -> str:
        response = self._client.chat.completions.create(
            model=self._model,
            messages=messages,
        )
        return response.choices[0].message.content.strip()
```

Setelah Provider dibuat, tambahkan test di `tests/providers/test_openai_provider.py` dan dokumentasikan keputusan ini sebagai ADR baru.

---

*Last updated: 2025 | Version: 1.0.0*
