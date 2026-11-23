# Retrieval Test Results

Embedding model: `gemini-embedding-001` | LLM: `gemini-3.5-flash-lite` (cadangan: `gemini-3.6-flash`) | top_k = 3 | skor = cosine similarity

| # | Tipe | Query | Expected | Top-3 FAQ | Top-1 score | hit@1 | hit@3 | Fallback |
|---|---|---|---|---|---|---|---|---|
| 1 | Ada di KB | Metode pembayaran apa saja yang tersedia? | #17 | #17, #22, #20 | 0.7957 | ✅ | ✅ | tidak |
| 2 | Ada di KB | Bagaimana cara membatalkan pesanan? | #3 | #3, #24, #8 | 0.7796 | ✅ | ✅ | tidak |
| 3 | Parafrase | duit gue kapan balik kalau barangnya udah gue retur? | #25 | #25, #27, #24 | 0.7815 | ✅ | ✅ | tidak |
| 4 | Parafrase | paket yang dateng udah hancur, harus ngapain? | #16 | #16, #54, #24 | 0.8078 | ✅ | ✅ | tidak |
| 5 | Parafrase | akun gw ga bisa masuk padahal kemarin masih bisa | #55 | #55, #32, #35 | 0.7623 | ✅ | ✅ | tidak |
| 6 | Tidak ada di KB | Bagaimana resep membuat nasi goreng yang enak? | - | #1, #31, #47 | 0.5898 | - | - | ya |
| 7 | Tidak ada di KB | Apakah ada program afiliasi untuk influencer? | - | #56, #40, #41 | 0.6303 | - | - | ya |

## Jawaban LLM per query

### 1. Metode pembayaran apa saja yang tersedia?
*Tipe: Ada di KB*

Halo! Kami mendukung transfer bank, kartu kredit/debit, e-wallet, dan pembayaran melalui minimarket rekanan untuk metode pembayaran yang tersedia. 

Sumber: FAQ #17

### 2. Bagaimana cara membatalkan pesanan?
*Tipe: Ada di KB*

Halo! Pesanan dapat dibatalkan melalui halaman 'Pesanan Saya' selama statusnya masih 'Diproses'. Perlu diingat bahwa pembatalan tidak dapat dilakukan setelah status pesanan berubah menjadi 'Dikirim'. 

Sumber: FAQ #3

### 3. duit gue kapan balik kalau barangnya udah gue retur?
*Tipe: Parafrase*

Halo! Dana refund Anda biasanya akan diproses dalam waktu 3-7 hari kerja setelah barang yang dikembalikan diterima dan diverifikasi oleh penjual. 

Sumber: FAQ #25

### 4. paket yang dateng udah hancur, harus ngapain?
*Tipe: Parafrase*

Halo! Jika paket diterima dalam kondisi rusak, silakan segera ambil foto atau video kondisi paket saat diterima. Setelah itu, laporkan melalui menu komplain dalam waktu 2x24 jam sejak barang diterima.

Sumber: FAQ #16

### 5. akun gw ga bisa masuk padahal kemarin masih bisa
*Tipe: Parafrase*

Halo! Akun Anda dapat terkunci sementara karena kesalahan login berulang atau terindikasi aktivitas mencurigakan. Silakan reset password atau hubungi customer service untuk mengatasinya.

Sumber: FAQ #55

### 6. Bagaimana resep membuat nasi goreng yang enak?
*Tipe: Tidak ada di KB*

Maaf, informasi tersebut belum tersedia di FAQ kami. Silakan hubungi customer service melalui live chat, email resmi, atau telepon di halaman bantuan.

Sumber: -

### 7. Apakah ada program afiliasi untuk influencer?
*Tipe: Tidak ada di KB*

Maaf, informasi tersebut belum tersedia di FAQ kami. Silakan hubungi customer service melalui live chat, email resmi, atau telepon di halaman bantuan.

Sumber: -
