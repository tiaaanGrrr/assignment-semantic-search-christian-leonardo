# Semantic Search & LLM Application: Asisten FAQ E-Commerce

Prototype asisten customer service yang mencari FAQ paling relevan dengan **semantic search** (Gemini Embeddings + FAISS),
lalu menyusun jawaban natural dengan **Gemini LLM** berdasarkan FAQ yang ditemukan.
Ini adalah tahap persiapan sebelum membangun sistem RAG penuh.

---

## 1. Problem Statement

### Objective
Tim customer service e-commerce menerima ratusan pertanyaan repetitif setiap hari (pemesanan, pengiriman, pembayaran, pengembalian, dll).
Tujuan prototype ini:
- Menemukan entri FAQ yang paling relevan dengan pertanyaan pelanggan, **walaupun pelanggan memakai kata-kata yang berbeda** dari FAQ
  (sinonim, bahasa santai, parafrase).
- Menyusun jawaban yang natural dan ramah menggunakan LLM, **hanya berdasarkan FAQ yang ditemukan**.

### Sumber dan struktur knowledge base
- File: `data/ecommerce_faq_knowledge_base.csv` (dari materi assignment, **tidak diubah**)
- **59 entri FAQ**, 4 kolom, tanpa nilai kosong dan tanpa pertanyaan duplikat:

| Kolom | Isi |
|---|---|
| `faq_id` | ID unik FAQ (1–59) |
| `category` | Kategori topik |
| `question` | Pertanyaan FAQ |
| `answer` | Jawaban resmi |

- Sebaran kategori: Pemesanan 9, Pengiriman 9, Pembayaran 8, Pengembalian 8, Akun 7, Produk 7, Promo 6, Customer Service 5.

### Batasan dan asumsi
- Sistem **hanya menjawab berdasarkan FAQ yang tersedia**. Jika informasi tidak ada, sistem menjawab dengan pesan fallback dan mengarahkan ke customer service.
- Satu baris FAQ = satu unit retrieval. Tidak ada chunking karena tiap FAQ pendek (dan chunking bukan bagian dari sesi ini).
- Bahasa knowledge base dan jawaban: Bahasa Indonesia.
- Setiap pertanyaan dijawab sendiri-sendiri (tidak ada memori percakapan).
- Prototype ini tidak punya akses ke data pesanan pelanggan, jadi tidak bisa menjawab pertanyaan personal (misalnya "pesanan saya nomor X di mana?").

---

## 2. Alur Sistem

```
            ┌────────────┐   ┌────────────┐   ┌─────────────┐
 CSV FAQ ──►│  INGEST    │──►│   EMBED    │──►│   STORE     │
            │ pandas,    │   │ Gemini     │   │ FAISS       │
            │ gabung Q+A │   │ embedding  │   │ IndexFlatIP │
            └────────────┘   └────────────┘   └──────┬──────┘
                                                     │
 Query user ──► embed query ──► RETRIEVE (top-3) ◄───┘
                                     │
                                     ▼
                     PROMPT (Persona/Task/Context/Format)
                                     │
                                     ▼
                         GENERATE (Gemini LLM) ──► Jawaban + sumber FAQ
```

| Tahap | Implementasi | Detail |
|---|---|---|
| **Ingest** | `pandas.read_csv` | Cek struktur data, lalu gabungkan `Kategori + Pertanyaan + Jawaban` jadi satu teks per FAQ |
| **Embed** | `gemini-embedding-001` | Dokumen di-embed dengan `task_type=RETRIEVAL_DOCUMENT`, query dengan `RETRIEVAL_QUERY`. Hasilnya vektor 3072 dimensi, dinormalisasi L2 |
| **Store** | `faiss.IndexFlatIP` | 59 vektor disimpan dan diverifikasi (`index.ntotal == 59`), lalu ditulis ke `vector_store/faq_index.faiss` |
| **Retrieve** | `semantic_search(query, top_k=3)` | Mengembalikan 3 FAQ terdekat + **cosine similarity** (makin besar makin mirip) |
| **Generate** | `generate_answer(query)` | Hasil retrieval disisipkan sebagai Context di prompt, lalu dikirim ke `gemini-3.5-flash-lite` (cadangan: `gemini-3.6-flash`) |

### Keputusan desain
- **Question + answer digabung**: user sering bertanya dengan kata yang ada di *jawaban*, bukan di pertanyaan FAQ (misal "refund masuk ke mana").
- **Cosine similarity (IndexFlatIP + normalisasi)** dipakai, bukan L2 distance seperti di starter. Untuk vektor ternormalisasi urutan hasilnya sama,
  tapi skor cosine (0–1, makin besar makin mirip) lebih mudah diinterpretasi saat analisis.
- **Flat index (exact search)**: dengan 59 vektor, pencarian brute-force sudah instan dan 100% akurat. Index approximate (IVF/HNSW) baru perlu untuk data yang jauh lebih besar.
- **Prompt terstruktur**:
  - **Persona**: asisten CS e-commerce yang ramah
  - **Task**: jawab HANYA dari Context; abaikan FAQ yang tidak relevan; kalau tidak ada jawabannya, pakai kalimat fallback yang sudah ditentukan
  - **Context**: 3 FAQ hasil retrieval (dengan ID dan kategori)
  - **Format**: 1–4 kalimat, Bahasa Indonesia, diakhiri `Sumber: FAQ #id`. Baris sumber ini memudahkan pengecekan apakah jawabannya *grounded*.
- **Model**: model di starter (`text-embedding-004`, `gemini-2.0-flash`) sudah tidak tersedia untuk API key baru, dan `gemini-3.8-flash`
  saat pengujian sering overload (503/504). Model `flash-lite` dipilih karena cepat dan cukup untuk FAQ pendek, ditambah model cadangan dan retry otomatis.

---

## 3. Hasil Pengujian

7 query uji (brief meminta minimal 5): 2 query yang jelas ada di KB, 3 parafrase/bahasa santai, dan 2 yang tidak ada di KB.
Hasil lengkap beserta jawaban LLM ada di [`documentation/retrieval_test_results.md`](documentation/retrieval_test_results.md).

| # | Tipe | Query | FAQ yang diharapkan | Top-3 hasil | Top-1 score | hit@1 | Fallback |
|---|---|---|---|---|---|---|---|
| 1 | Ada di KB | Metode pembayaran apa saja yang tersedia? | #17 | #17, #22, #20 | 0.796 | ✅ | tidak |
| 2 | Ada di KB | Bagaimana cara membatalkan pesanan? | #3 | #3, #24, #8 | 0.780 | ✅ | tidak |
| 3 | Parafrase | duit gue kapan balik kalau barangnya udah gue retur? | #25 | #25, #27, #24 | 0.782 | ✅ | tidak |
| 4 | Parafrase | paket yang dateng udah hancur, harus ngapain? | #16 | #16, #54, #24 | 0.808 | ✅ | tidak |
| 5 | Parafrase | akun gw ga bisa masuk padahal kemarin masih bisa | #55 | #55, #32, #35 | 0.762 | ✅ | tidak |
| 6 | Tidak ada di KB | Bagaimana resep membuat nasi goreng yang enak? | – | #1, #31, #47 | 0.590 | – | ya ✅ |
| 7 | Tidak ada di KB | Apakah ada program afiliasi untuk influencer? | – | #56, #40, #41 | 0.630 | – | ya ✅ |

**Ringkasan:** hit@1 = **5/5**, hit@3 = **5/5**, fallback benar pada **2/2** query di luar KB, dan **0/5** query yang bisa dijawab salah dibalas fallback.

### Analisis relevansi
1. **Retrieval akurat untuk semua query yang ada jawabannya.** FAQ yang benar selalu ada di peringkat 1 (skor 0.76–0.81).
2. **Semantic search menangkap makna, bukan kata kunci.** Rata-rata skor query parafrase (0.784) hampir sama dengan query langsung (0.788),
   padahal kata-katanya beda: "duit balik / retur" → *refund / pengembalian*, "hancur" → *rusak*, "ga bisa masuk" → *tidak bisa login*.
   Pencarian kata kunci biasa kemungkinan besar gagal di query-query ini.
3. **Peringkat 2–3 sering "topiknya dekat, tapi bukan jawabannya"**, misalnya "membatalkan pesanan" juga menarik FAQ pengembalian barang (#24)
   dan pembatalan otomatis (#8). Selisih skornya dengan FAQ yang benar hanya ~0.07. LLM berhasil memilih FAQ yang tepat dan hanya mengutip FAQ itu sebagai sumber.
4. **FAISS selalu mengembalikan top-k, bahkan untuk query yang tidak ada jawabannya.** "Resep nasi goreng" menarik FAQ yang hanya mirip
   *pola kalimatnya* ("Bagaimana cara …"), dengan skor rendah (0.59). Yang mencegah jawaban mengarang adalah **aturan di prompt**, dan LLM memakai fallback dengan benar.
5. **Kasus paling berisiko adalah query yang topiknya berdekatan** ("afiliasi influencer" vs FAQ promo: 0.63). Jarak skornya ke query yang bisa dijawab
   lebih tipis dibanding query yang benar-benar di luar topik. Kalau prompt-nya longgar, di sinilah LLM paling mungkin berhalusinasi.

---

## 4. Keterbatasan Sistem
- **Tidak ada ambang skor (threshold).** Retrieval selalu memberi 3 hasil, jadi penolakan query di luar KB sepenuhnya bergantung pada kepatuhan LLM terhadap prompt.
  Di pengujian ini, threshold sekitar 0.68 terlihat bisa memisahkan kedua jenis query, tapi 7 query terlalu sedikit untuk menentukan angka itu dengan yakin.
- **Test set kecil** (7 query). Hasil 5/5 menunjukkan sistem bekerja, tapi belum cukup untuk mengukur akurasi secara statistik.
- **Pertanyaan dengan beberapa maksud sekaligus** (misal "cara refund dan berapa lama?") bisa butuh lebih dari 3 FAQ, dan belum diuji.
- **Jawaban LLM tidak deterministik.** Kalimatnya bisa berbeda tiap kali dijalankan (retrieval FAISS tetap sama).
- **Deteksi fallback di evaluasi** memakai pencocokan teks, jadi akan meleset kalau LLM mengubah sedikit kalimat fallback-nya.
- **Ketergantungan pada ketersediaan model Gemini.** Selama pengerjaan, beberapa model sudah dihentikan atau sedang overload.
  Hal ini dimitigasi dengan retry, timeout, dan model cadangan.
- **Embedding dibuat ulang setiap notebook dijalankan.** Untuk 59 FAQ ini tidak masalah (satu request batch), tapi untuk knowledge base besar
  sebaiknya index dimuat dari `vector_store/faq_index.faiss` dan hanya dibangun ulang kalau data berubah.
- Tidak ada memori percakapan dan tidak terhubung ke data pesanan pelanggan.

---

## 5. Cara Menjalankan

### Struktur project
```
semantic-search-assignment/
├── data/
│   └── ecommerce_faq_knowledge_base.csv          # knowledge base asli (tidak diubah)
├── notebook/
│   └── starter_notebook_semantic_search.ipynb    # seluruh pipeline: ingest → embed → store → retrieve → generate → uji
├── vector_store/
│   └── faq_index.faiss                           # FAISS index hasil notebook (59 vektor)
├── documentation/
│   └── retrieval_test_results.md                 # hasil mentah pengujian query
├── .env.example                                  # contoh isi .env
├── README.md
└── requirements.txt
```

### Langkah-langkah (Windows PowerShell)
1. Buat virtual environment dan install dependency:
   ```powershell
   python -m venv venv
   .\venv\Scripts\Activate.ps1
   pip install -r requirements.txt
   ```
2. Salin `.env.example` menjadi `.env`, lalu isi API key Gemini (dari Google AI Studio):
   ```
   GEMINI_API_KEY=your_gemini_api_key_here
   ```
3. Jalankan notebook:
   - **VS Code / Jupyter**: buka `notebook/starter_notebook_semantic_search.ipynb`, pilih kernel dari `venv`, lalu **Run All**.
   - **Dari terminal** (menjalankan semua cell dan menyimpan output ke notebook):
     ```powershell
     python -m jupyter nbconvert --to notebook --execute --inplace notebook/starter_notebook_semantic_search.ipynb
     ```
4. Output yang dihasilkan: `vector_store/faq_index.faiss` dan `documentation/retrieval_test_results.md`.
