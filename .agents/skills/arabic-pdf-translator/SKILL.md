# Penerjemah PDF Arab → Indonesia (Workflow Manual)

## Role & Scope

Skill ini memandu Devin untuk **menerjemahkan dokumen PDF berbahasa Arab
(puluhan sampai ratusan halaman) ke Bahasa Indonesia secara manual, per sesi**.
Output akhir: **satu file markdown per PDF**, dengan format blok
(Arab + Indonesia + Catatan) sesuai skill [`arabic-to-indonesian-translator`](../arabic-to-indonesian-translator/SKILL.md).

> **Prasyarat:** Skill ini memperluas — bukan menggantikan — skill penerjemah
> dasar. Semua aturan gaya (*maknawiyah*, glosarium istilah baku, format blok,
> dsb.) di skill itu **harus diikuti**. Skill ini khusus mengatur workflow PDF.

## Gambaran Workflow

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ 1. Triage PDF│ → │ 2. Ekstrak / │ → │ 3. Normalize │ → │ 4. Translate │ → │ 5. Output &  │
│  (scan/text) │   │    OCR       │   │    & clean   │   │    per blok  │   │    archive   │
└──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
```

## 1. Triage PDF

**Tujuan:** tentukan apakah PDF ini text-based, scan gambar, atau campuran
— dan apakah layout-nya sederhana (single-column) atau kompleks (multi-column,
ada hasyiyah/footnote-pinggir).

### Perintah cek metadata & text-layer

```bash
# Cek metadata
pdfinfo "$PDF"

# Cek apakah ada text layer yang bisa diekstrak
pdftotext -f 1 -l 1 "$PDF" - | head
```

### Kriteria keputusan

| Hasil `pdftotext` halaman pertama          | Kesimpulan                       |
| ------------------------------------------ | -------------------------------- |
| Mengembalikan teks Arab yang bermakna      | **Text-based** — pakai ekstraksi |
| Mengembalikan teks acak / mojibake / kosong| **Scan** — butuh OCR             |
| Mengembalikan sebagian teks + sebagian kosong | **Campuran** — per-halaman check |

### Untuk kasus campuran

Cek per halaman: kalau `pdftotext -f N -l N "$PDF" -` menghasilkan teks bermakna,
pakai ekstraksi. Kalau kosong/acak, pakai OCR untuk halaman itu.

## 2. Ekstraksi / OCR

### 2a. PDF text-based

```bash
pdftotext -f <start> -l <end> "$PDF" page-<N>.txt
```

**JANGAN pakai `-layout` untuk PDF Arab** — flag itu mengacaukan urutan baris
RTL. Mode default (tanpa flag) menghasilkan alur baris yang benar.

### 2b. PDF scan — OCR dengan Tesseract

Langkah:

```bash
# 1. Convert halaman PDF ke gambar (PNG 300 DPI minimum untuk Arabic)
pdftoppm -r 300 -f <N> -l <N> "$PDF" page-<N> -png

# 2. OCR dengan Tesseract (bahasa: Arabic)
tesseract page-<N>-*.png page-<N> -l ara --psm 6

# Hasil: page-<N>.txt
```

**Catatan penting:**
- Default Tesseract `ara` akurasi medium untuk Arabic modern ber-harakat,
  **jelek untuk kitab kuning / naskhi klasik**. Kalau hasil banyak error:
  kasih tau user, sarankan OCR alternatif (Mistral OCR / Google Vision).
- PSM 6 (Assume a single uniform block of text) umumnya bagus untuk halaman
  buku. Untuk multi-kolom, coba PSM 1 atau 3.

### 2c. Layout kompleks (kitab turats: matn + hasyiyah + hamisy)

Crop manual per-region sebelum OCR:

```bash
# Inspect halaman secara visual dulu
pdftoppm -r 200 -f <N> -l <N> "$PDF" preview -png
# buka preview-*.png — identifikasi bounding box matn vs hasyiyah

# Crop tiap region pakai ImageMagick, lalu OCR masing-masing
convert page-<N>-*.png -crop <WxH+X+Y> matn.png
tesseract matn.png matn -l ara --psm 6
```

## 3. Normalize & Clean

Teks Arab dari PDF sering memuat **Arabic Presentation Forms** (blok Unicode
FB50-FEFF) — glyph variant kontekstual yang harus di-normalize ke huruf
Arab standar.

### Python snippet normalisasi

```python
import unicodedata, re

def normalize_arabic(text: str) -> str:
    # 1. NFKC: konversi Presentation Forms → standard Arabic letters
    text = unicodedata.normalize('NFKC', text)

    # 2. Fix tanda baca RTL yang ter-flip:
    #    Di sebagian PDF, « » tertukar dengan » « — periksa konteks.

    # 3. Hilangkan ekstra spasi mid-word yang disebabkan kashida/kerning
    #    HATI-HATI: jangan agresif — bisa merusak pemisahan kata yang sah.
    #    Cukup kolaps multiple spaces jadi satu.
    text = re.sub(r'[ \t]+', ' ', text)

    # 4. Hilangkan baris kosong berlebih
    text = re.sub(r'\n{3,}', '\n\n', text)

    return text.strip()
```

### Isu-isu umum yang HARUS diperiksa manual

1. **Spasi mid-word ganjil** — contoh "م ن" seharusnya "من", "ي شري" → "يشير".
   Biasanya sisa dari kashida/kerning. Fix by hand di output terjemahan
   (Devin yang baca konteks), jangan coba auto-regex karena bisa menggabungkan
   kata yang memang terpisah.
2. **Urutan LAM-MEEM ter-flip** pada kata seperti "لمدينة" → "ملدينة". Ini
   disebabkan encoding ligature yang tidak standar di PDF. Kalau sering
   terjadi: terjemahkan berdasarkan konteks (Devin dapat membaca meski ada
   glitch); kalau halaman bermasalah berat, **tandai halaman itu untuk
   screenshot verification** (lihat §6).
3. **Angka footnote inline** — angka superscript seringkali ikut ter-extract
   sebagai angka biasa di tengah kalimat. Pindahkan ke catatan kaki markdown
   (`[^1]`) saat menerjemahkan.
4. **Teks Latin bocor** — URL, nama penerbit, ISBN dalam halaman copyright.
   Pertahankan apa adanya di blok Arab, jangan diterjemahkan.

## 4. Terjemahkan (Per Blok)

Ikuti skill [`arabic-to-indonesian-translator`](../arabic-to-indonesian-translator/SKILL.md):
gaya *maknawiyah*, format **Arab + Indonesia + Catatan**.

### Chunking yang disarankan

- **Per halaman** sebagai unit utama.
- **Per paragraf** di dalam halaman — nomori `[Paragraf 1]`, `[Paragraf 2]`, dst.
- **Quotation block** (ayat/hadits/kutipan ulama) → pisahkan sebagai blok
  tersendiri, jangan digabung paragraf.
- **Footnote** → terjemahkan terpisah di akhir blok halaman, kaitkan dengan
  `[^N]` markdown.

### Konsistensi lintas halaman

- **Nama diri** (orang, tempat, klan) → transliterasi konsisten dari awal
  sampai akhir. Saat muncul pertama, catat padanan Latin yang akan dipakai
  (mis. عبد المطلب → 'Abd al-Muttalib), lalu pakai itu seterusnya.
- **Istilah teknis disiplin** (fiqih/ushul/balaghah/sosiologi klasik) →
  pertahankan istilah Arab-nya + padanan Indonesia saat pertama muncul,
  lalu boleh pakai istilah Arab saja di kemunculan berikutnya.
- **Gaya penulis** → kalau penulis suka kalimat panjang berlapis (seperti
  ulama klasik), pertahankan ritmenya di Indonesia. Kalau gaya jurnalistik
  modern, pakai kalimat pendek yang alir.

### Per-genre adjustment

| Genre PDF                    | Penyesuaian                                            |
| ---------------------------- | ------------------------------------------------------ |
| Kitab klasik / turats        | Pertahankan struktur matn + syarah kalau ada.          |
| Tafsir Qur'an                | Ayat pakai teks Qur'an standar; tafsir maknawiyah.     |
| Fiqih / ushul fiqih          | Istilah teknis jangan diterjemah paksa.                |
| Kajian sejarah / akademik    | Kutipan & footnote pertahankan, transliterasi nama.    |
| Sastra / puisi               | Prioritas makna & rasa; tawarkan versi prosa.          |
| Artikel modern / berita      | Gaya Indonesia yang luwes, seperti artikel Tempo/Kompas.|

## 5. Output: Satu `.md` per PDF

### Struktur file

```markdown
# <Judul Arab>

**Judul (ID):** <Terjemahan judul>
**Penulis:** <Nama penulis + transliterasi>
**Penerbit:** <Penerbit>
**Tahun terbit:** <Tahun>
**Jumlah halaman:** <N>
**File sumber:** `<nama-file.pdf>`

---

> **Catatan editorial:**
> Gaya: maknawiyah.
> Sumber: PDF text-based / scan / campuran.
> [Info spesifik genre kalau relevan — mis. "kajian revisionis", "kitab
> turats", "tafsir kontemporer", dsb.]

---

## Halaman <N> — <Topik Halaman>

### [Paragraf 1]

**Arab:**
<teks Arab asli, sudah di-normalize>

**Indonesia:**
<terjemahan maknawiyah>

**Catatan:**
- <catatan istilah / idiom / ambiguitas>
- <rujukan kalau perlu>

### [Paragraf 2]
...

---

## Halaman <N+1> — ...
```

### Naming convention

- File markdown: `<slug-judul>.md` di folder `translations/` di root repo kerja
  (atau di working directory yang disepakati).
- Slug: pakai kebab-case ASCII, transliterasi dari judul Arab.
  Contoh: `al-hizb-al-hashimi-wa-tasis-al-dawlah.md`.
- Kalau user punya preferensi path lain, ikuti permintaan user.

### Checkpoint & resume

Untuk dokumen panjang (>20 halaman), **commit progress ke file tiap 10-20
halaman** dan log halaman terakhir yang selesai di awal file. Kalau sesi
terputus, sesi berikut bisa resume dari halaman berikutnya tanpa mulai ulang.

Format log singkat di awal file:

```markdown
> **Progress log:**
> - [2025-11-10] Halaman 1-20 selesai
> - [2025-11-10] Halaman 21-40 selesai
> - [RESUME FROM: Halaman 41]
```

## 6. Verifikasi Halaman Meragukan (Screenshot Embed)

Kalau hasil ekstraksi/OCR untuk suatu halaman terlihat meragukan (banyak
mojibake, kata pecah aneh, urutan kacau), **embed screenshot halaman asli**
di markdown untuk verifikasi visual.

### Cara generate screenshot

```bash
pdftoppm -r 150 -f <N> -l <N> "$PDF" verify-page-<N> -png
# hasil: verify-page-<N>-1.png
```

### Cara embed di markdown

Upload screenshot ke storage dengan `upload_attachment` untuk dapat URL,
lalu embed:

```markdown
### Halaman <N> — Verifikasi Visual

![Halaman <N> asli](<URL dari upload_attachment>)

> ⚠ Hasil ekstraksi halaman ini terlihat cacat pada baris X-Y. Terjemahan
> di bawah dibuat berdasarkan pembacaan visual dari screenshot di atas.
```

**Kapan perlu screenshot:**
- Halaman dengan banyak diagram / tabel / persamaan.
- Halaman yang hasil ekstraksinya mengandung >20% kata tidak-bermakna.
- Halaman yang konteksnya kritis (pembuka bab, definisi istilah kunci,
  kutipan ayat/hadits) dan ada keraguan akurasi ekstraksi.
- Halaman yang berisi *marginalia* (catatan pinggir) — hasyiyah di kitab
  kuning.

## 7. Quality Checklist (Sebelum Report Completion)

- [ ] Semua halaman yang diminta user sudah diterjemahkan (bukan sampel saja).
- [ ] Setiap halaman punya blok **Arab + Indonesia + Catatan** sesuai format.
- [ ] Transliterasi nama diri konsisten dari halaman pertama sampai akhir.
- [ ] Footnote dari PDF asli terhubung benar (superscript di teks ↔ isi
  footnote di akhir blok).
- [ ] Istilah syar'i baku mengikuti glosarium di skill penerjemah dasar
  (shalat, zakat, taqwa, dll — tidak diterjemah paksa).
- [ ] Quotation block (ayat Qur'an / hadits / kutipan ulama) dipisahkan
  dari paragraf biasa.
- [ ] Halaman yang meragukan sudah di-screenshot + disertakan (§6).
- [ ] Metadata (judul, penulis, tahun, jumlah halaman) di awal file lengkap.
- [ ] Progress log ter-update kalau dokumen di-chunk antar-sesi.
- [ ] File markdown valid (tidak ada broken link, tabel rusak, dsb.) — cek
  dengan preview sebelum delivery.

## 8. Kasus Khusus

### PDF dengan konten sensitif / kontroversial

Kalau isi buku kontroversial (kajian revisionis, kritik agama, perdebatan
madzhab, polemik), **terjemahkan apa adanya**. Jangan menyensor, menambahkan
tafsir, atau memihak. Tambahkan **Catatan editorial** netral di awal file
yang menyebut genre dan posisi kajian — pembaca berhak tahu karakter
sumbernya tanpa dihakimi oleh penerjemah.

### PDF bilingual (Arab + Indonesia sudah ada)

Kalau PDF ternyata sudah punya terjemahan Indonesia di sebelahnya (mis.
Qur'an Kemenag, kitab terjemahan bilingual), **tanyakan user dulu**: mau
dibikin ulang (terjemahan baru), atau hanya di-extract ulang untuk
dirapikan formatnya?

### PDF proteksi / password

Kalau PDF di-password: kasih tau user, minta password, atau kalau user
gak punya — proses berhenti di triage. Jangan coba brute-force.

### PDF dengan gambar / diagram penting

Screenshot gambar asli + kasih *caption* terjemahan di bawahnya. Jangan
coba "menerjemahkan gambar" — cukup terjemahkan teks di dalamnya kalau ada.

### Kitab kuning dengan harakat lengkap

Pertahankan harakat di blok Arab output — bantu pembaca memverifikasi
bacaan. Tesseract mungkin tidak menangkap semua harakat dari scan;
kalau meragukan, pakai screenshot verification.

## 9. Interaction Protocol

1. **User kirim PDF tanpa instruksi lain** → jalankan Triage (§1), lalu
   laporkan temuan ke user: "PDF text-based/scan, N halaman, topik X,
   estimasi Y blok teks." Tanyakan: full translation atau sampel dulu?
2. **User minta sampel** → terjemahkan 3-5 halaman pertama (termasuk
   cover + TOC + halaman awal konten) sebagai *proof-of-concept*, minta
   feedback, baru lanjutkan full.
3. **User minta full** → jalankan pipeline lengkap dengan progress log,
   commit/update file tiap chunk.
4. **User minta halaman spesifik** (mis. "halaman 40-60") → langsung
   terjemahkan range itu, simpan sebagai `<slug>-p40-60.md`.
5. **User gak yakin mau apa** → rekomendasikan sampel dulu, jelaskan kenapa
   (kalibrasi gaya sebelum commit ke full).

## 10. Anti-Pola (Jangan Dilakukan)

- ❌ Pakai `pdftotext -layout` untuk PDF Arab (merusak urutan RTL).
- ❌ Auto-regex agresif untuk "fix" spasi ganjil di Arabic (bisa
  menggabung kata yang memang terpisah).
- ❌ Skip halaman yang ekstraksinya cacat tanpa screenshot verifikasi.
- ❌ Meringkas halaman yang panjang supaya hemat waktu — user minta
  **terjemahan**, bukan ringkasan.
- ❌ Mengganti footnote bernomor dari PDF asli dengan nomor sendiri
  tanpa catatan.
- ❌ Menyensor / mengedit konten kontroversial — terjemahkan apa adanya
  dengan catatan editorial netral.
- ❌ Delivery tanpa metadata halaman + progress log untuk dokumen panjang.

## 11. Contoh Output (Mini)

Untuk halaman dengan 1 paragraf + 1 footnote:

```markdown
## Halaman 7 — Awal Bab 1: "Pendirian (1)"

### [Paragraf 1]

**Arab:**
«إذا أراد الله إنشاء دولة خلق لها أمثال هؤلاء.»

قالها «عبد المطلب بن هاشم» وهو يشير إلى أبنائه وحفدته.[^1]

[^1]: المصدر: رواية شفهية من تراث مكة.

**Indonesia:**
*"Apabila Allah hendak mendirikan sebuah negara, Ia ciptakan untuknya
orang-orang seperti mereka."*

Kalimat itu diucapkan oleh **'Abd al-Muttalib bin Hashim** sambil
menunjuk pada anak-anak dan cucu-cucunya.[^1-id]

[^1-id]: Sumber: riwayat lisan dari tradisi Mekkah.

**Catatan:**
- Ucapan ini tidak memiliki sanad sahih; ditampilkan sebagai *epigraph*.
- *Ḥafadah* (حفدة) — "cucu-cucu"; menekankan kesinambungan klan.
```
