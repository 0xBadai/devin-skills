# devin-skills

Kumpulan `SKILL.md` untuk [Devin](https://devin.ai) — mendefinisikan kapabilitas khusus
yang otomatis dimuat di sesi Devin yang berjalan di repo ini.

## Struktur

```
.agents/skills/
└── <skill-name>/
    └── SKILL.md
```

Setiap folder di bawah `.agents/skills/` berisi satu skill. `SKILL.md`-nya dibaca
Devin sebagai panduan perilaku ketika skill tersebut diaktifkan.

## Skill yang tersedia

- **[arabic-to-indonesian-translator](.agents/skills/arabic-to-indonesian-translator/SKILL.md)** —
  Penerjemah Arab → Indonesia dengan gaya *maknawiyah* (berbasis makna).
- **[arabic-pdf-translator](.agents/skills/arabic-pdf-translator/SKILL.md)** —
  Workflow untuk menerjemahkan **file PDF berbahasa Arab** (puluhan-ratusan
  halaman) ke satu file markdown per dokumen. Menggunakan `pdftotext` untuk
  PDF text-based dan Tesseract OCR untuk PDF scan. Memperluas skill
  penerjemah dasar di atas.
