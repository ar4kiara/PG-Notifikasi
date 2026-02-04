# NotiForwarder

App Android untuk meneruskan notifikasi pembayaran (DANA, OVO, GoPay, dll.) ke API. Dipasang di HP → saat ada notif pembayaran, app kirim nominal ke backend.

---

## Cara pakai app

1. **Tambah endpoint**: URL = alamat API + `/payment_listener` (mis. `https://dabis-api.onrender.com/payment_listener`), Secret = ar4kiara (kode rahasia)
2. **Buka Pengaturan Akses Notifikasi** → aktifkan untuk NotiForwarder.
3. Nyalakan **Teruskan notifikasi ke API**. Izinkan notifikasi + baterai/Autostart kalau diminta.
4. Cek **Log aktivitas** untuk pastikan notif terkirim.

---

## Integrasi ke bot / script (WA, Telegram, dll.)

Agar bot kamu bisa terima pembayaran lewat app ini, **backend harus punya 3 endpoint** ini:

| Endpoint | Method | Fungsi |
|----------|--------|--------|
| `/create_invoice` | POST | Buat tagihan → dapat nominal yang harus dibayar (unique_amount). |
| `/payment_listener` | POST | Diterima dari **NotiForwarder** saat ada pembayaran → cocokkan nominal, update status lunas. |
| `/cek_pembayaran` | GET | Cek status tagihan (pending/lunas) dari nominal. |

**Config di script bot kamu** (contoh variabel global):

```js
global.API_URL = 'https://dabis-api.onrender.com';   // base URL API (tanpa /payment_listener)
global.unique_code_min = 1;   // batas bawah kode unik (angka)
global.unique_code_max = 99;   // batas atas kode unik (angka)
global.unique_code_mode = 'minus';   // 'minus' = bayar base - kode | 'plus' = bayar base + kode
```

- **API_URL**: base URL API (tanpa `/payment_listener`). Di app, isi URL lengkap: `https://dabis-api.onrender.com/payment_listener`.
- **Secret**: satu kode rahasia yang sama di backend dan di app (ar4kiara).

---

### 1. POST `/create_invoice` — buat tagihan

**Request (body JSON):**

| Field | Wajib | Contoh | Keterangan |
|-------|--------|--------|------------|
| order_id | ✅ | `"INV-001"` | ID pesanan (unik). |
| amount | ✅ | `100000` | Harga dasar (angka). |
| callback_url | ❌ | `"https://..."` | Dipanggil GET saat lunas (opsional). |
| unique_code_mode | ❌ | `"minus"` / `"plus"` | Default `minus`. |
| unique_code_min | ❌ | `1` | Default 1. |
| unique_code_max | ❌ | `99` | Default 99. |

**Response sukses (HTTP 201):**

```json
{
  "success": true,
  "order_id": "INV-001",
  "unique_amount": 99947,
  "message": "Silakan bayar TEPAT sejumlah 99947"
}
```

**Response gagal (400/500):**

```json
{ "success": false, "message": "order_id dan amount wajib diisi" }
```

```json
{ "success": false, "message": "Gagal membuat kode unik (Tabrakan). Silakan coba lagi." }
```

---

### 2. POST `/payment_listener` — terima dari NotiForwarder

Ini yang **dipanggil oleh app NotiForwarder** saat ada notifikasi pembayaran. Backend harus terima body seperti ini:

**Payload yang dikirim app (request body):**

```json
{
  "secret": "rahasia_anda",
  "amount": 99947,
  "currency": "IDR",
  "source": {
    "package": "id.dana",
    "appName": "DANA",
    "patternName": "Dana sukses",
    "patternRegex": "Kamu berhasil menerima Rp[\\d\\.,]+"
  },
  "notification": {
    "title": "Kamu berhasil menerima",
    "text": "Kamu berhasil menerima Rp99.947 dari ...",
    "timestamp": 1732176000000
  },
  "device": { "manufacturer": "Xiaomi", "model": "...", "sdk": 35 }
}
```

**Yang wajib dipakai:** `secret` (validasi) dan `amount` (nominal). Sisanya opsional.

**Response sukses (HTTP 200):**

```json
{
  "success": true,
  "message": "Pembayaran 99947 berhasil dicocokkan."
}
```

**Response gagal:**

- Secret salah (403): `{ "success": false, "message": "Invalid Secret Key" }`
- Amount tidak valid (400): `{ "success": false, "message": "Amount tidak valid" }`
- Tidak ada invoice (404): `{ "success": false, "message": "Tidak ada invoice PENDING untuk jumlah 99947" }`

---

### 3. GET `/cek_pembayaran` — cek status tagihan

**Request:** GET dengan query `unique_amount`.  
Contoh: `GET /cek_pembayaran?unique_amount=99947`

**Response sukses (200):**

```json
{
  "success": true,
  "status": "SUCCESS",
  "order_id": "INV-001",
  "unique_amount": 99947
}
```

`status` bisa `PENDING` atau `SUCCESS`.

**Response gagal (404):**

```json
{
  "success": false,
  "status": "NOT_FOUND",
  "message": "Invoice untuk jumlah tersebut tidak ditemukan"
}
```

---

## Alur singkat

1. Bot panggil **create_invoice** → dapat **unique_amount** → kasih ke customer (bayar tepat segitu).
2. Customer bayar → notif di HP → **NotiForwarder** kirim ke **payment_listener** (secret + amount).
3. Backend cocokkan amount dengan invoice → update SUCCESS.
4. Bot bisa **cek_pembayaran** (polling) atau tunggu callback_url.

---
