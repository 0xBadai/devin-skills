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
