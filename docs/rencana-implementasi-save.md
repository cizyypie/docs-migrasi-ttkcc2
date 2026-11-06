# Rencana Implementasi: Fitur Save + List TTKCC2

> [!IMPORTANT]
> Dokumen ini adalah panduan teknis dan eksekusi dua developer yang bekerja paralel.
> Baca seluruh dokumen sebelum mulai coding. Sepakati API Contract di Bagian 2 dan Klasifikasi Query di Bagian 1.3 terlebih dahulu.
> Aturan proyek: Dilarang menggunakan emoticon / emoji dalam kode, query SQL, maupun dokumentasi.

---

## Daftar Isi

1. [Gambaran Besar & Arsitektur Fitur](#1-gambaran-besar--arsitektur-fitur)
   - 1.1 [Fitur yang Akan Dibuat](#11-fitur-yang-akan-dibuat)
   - 1.2 [File yang Akan Dibuat / Dimodifikasi](#12-file-yang-akan-dibuat--dimodifikasi)
   - 1.3 [Analisis & Klasifikasi Query Database (Existing vs Baru)](#13-analisis--klasifikasi-query-database-existing-vs-baru)
   - 1.4 [Pemetaan Kolom Database ENTBT Terverifikasi](#14-pemetaan-kolom-database-entbt-terverifikasi)
2. [API Contract — Wajib Disepakati Dulu](#2-api-contract--wajib-disepakati-dulu)
   - 2.1 [Create TTK (`POST /a/ttkcc/createTTKCC`)](#21-create-ttk)
   - 2.2 [List TTK (`GET /a/ttkcc/listTTKCC`)](#22-list-ttk)
   - 2.3 [Get Detail TTK (`GET /a/ttkcc/getTTKCC/{btid}`)](#23-get-detail-ttk)
3. [Pembagian Kerja](#3-pembagian-kerja)
4. [Dev A — Frontend](#4-dev-a--frontend)
5. [Dev B — Backend](#5-dev-b--backend)
6. [Urutan Eksekusi & Checkpoint](#6-urutan-eksekusi--checkpoint)
7. [Hal Kritis & Catatan Verifikasi](#7-hal-kritis--catatan-verifikasi)

---

## 1. Gambaran Besar & Arsitektur Fitur

### 1.1 Fitur yang Akan Dibuat

| Fitur | Keterangan |
|-------|------------|
| Tombol Save di TTKCC2 | Tombol hijau di kanan atas toolbar, mirip tombol Save pada sandbox DW |
| Validasi form sebelum save | Cek field wajib, cek kelengkapan data & kalkulasi tarif |
| Konfirmasi dialog | Dialog pop-up: "Are you sure to CREATE new record?" (sama persis dengan sandbox DW) |
| Loading overlay saat proses | "Creating data..." overlay saat HTTP request berlangsung |
| Notifikasi hasil | Banner / popup sukses dengan nomor TTK baru atau pesan gagal |
| Reset & feedback form | Nomor TTK tampil di chip hijau atas, siap cetak atau buat baru |
| Halaman List TTK (`TTKCCList.vue`) | Tabel daftar riwayat TTK corporate kubikasi yang telah tersimpan |
| Filter data list | Filter berdasarkan rentang tanggal, Account corporate, dan status |
| Navigasi detail | Klik baris nomor TTK membuka halaman TTKCC2 dalam mode referensi / detail |

---

### 1.2 File yang Akan Dibuat / Dimodifikasi

| File | Pemilik | Status | Deskripsi |
|------|---------|--------|-----------|
| `web/dev/dcaf/src/pages/TTKCC2.vue` | Dev A | Dimodifikasi | Tambah tombol Save, konfirmasi, method save & payload builder |
| `web/dev/dcaf/src/pages/TTKCCList.vue` | Dev A | **File Baru** | Halaman daftar data TTK, filter, dan tabel riwayat |
| `web/dev/dcaf/src/router/routes.js` | Dev A | Dimodifikasi | Registrasi route `/TTKCCList` (1 baris) |
| `lib/perl/DCAF/Controller/TTKCC.pm` | Dev B | **File Baru** | Controller API DCAF: operasi `createTTKCC`, `listTTKCC`, `getTTKCC` |
| `etc/api/ttkcc.yaml` | Dev B | **File Baru** | OpenAPI 3.0 specification untuk routing DCAF |
| `etc/dcaf.yaml` | Dev B | Dimodifikasi | Registrasi route modul `ttkcc` ke DCAF gateway (1 baris) |
| `lib/db/git/dcaf-data.sql` | Dev B | Dimodifikasi | Menambahkan 4 query SQL baru ke tabel `EDBQ` |

> [!NOTE]
> Pembagian kerja zero overlap. Masing-masing developer bekerja di file terpisah, sehingga tidak terjadi conflict saat pengerjaan paralel.

---

### 1.3 Analisis & Klasifikasi Query Database (Existing vs Baru)

Sistem SWB DCAF tidak mengizinkan penulisan raw SQL di dalam file Perl (`.pm`). Semua query harus terdaftar di tabel database `EDBQ` dan dieksekusi menggunakan method `$ros->q2j('queryName', ...)`.

Berdasarkan analisis menyeluruh terhadap form `TTKCC2.vue`, form CRUD sandbox DW lama (`fnm.50180000ENTBT_m_TTKCC2`), serta daftar query di `lib/db/git/dcaf-data.sql`:

#### A. Query Form Input / Kalkulasi (0 Query Baru — Semua Sudah Ada)
Seluruh query yang digunakan pada form `TTKCC2.vue` saat ini **SUDAH ADA** di database dan sudah aktif berjalan:

1. `whnPriceListBaru` — Query hitung tarif dasar (asal, tujuan, layanan, account). Sudah ada.
2. `whnPriceListAdditional` — Query tarif tambahan (surcharge & packing). Sudah ada.
3. `whnGetRouteListByAccV2` — Query dropdown rute layanan berdasarkan account. Sudah ada.
4. `RELBIBT_TTKC` — Query dropdown tipe layanan TTK Corporate. Sudah ada.
5. `RELBABTP_AccountActive` — Query autocomplete pencarian account corporate. Sudah ada.
6. `pllvWilayahBEAD` — Query autocomplete wilayah asal dan tujuan. Sudah ada.
7. `pllv_shiptoaddr` — Query dropdown alamat shipto supplier / penerima. Sudah ada.
8. `RELBEBANKBTP` — Query dropdown data bank account. Sudah ada.
9. `pllv_dokkembali` — Query dropdown pilihan dokumen bukti kembali. Sudah ada.

**Kesimpulan Form:** Untuk halaman form input TTKCC2, **TIDAK ADA query baru** yang perlu dibuat.

#### B. Query yang Wajib Dibuat untuk Backend Save & List (Total: 4 Query Baru)
Pada aplikasi lama (sandbox DW), aksi simpan dilakukan melalui submit form HTML standar ke `dw.cgi` (`fnm.50180000ENTBT`), yang ditangani internal oleh engine DW tanpa nama query EDBQ tersendiri. Di arsitektur modern SWB, proses simpan dilakukan via endpoint API DCAF (`POST /a/ttkcc/createTTKCC`).

Oleh karena itu, Dev B wajib menambahkan **4 query baru** ke dalam tabel `EDBQ`:

| No | Nama Query | Tipe | Tujuan Penggunaan | Keterangan |
|----|------------|------|-------------------|------------|
| 1 | `ttkccCreateHeader` | INSERT | Simpan data induk (Header TTK) ke tabel `ENTBT` | Dipanggil oleh Controller `TTKCC.pm` saat Save |
| 2 | `ttkccGetNomor` | SELECT | Ambil nomor TTK (`ATRBTNO`) yang terbentuk berdasarkan ID header (`ATRBTID`) | Dipanggil setelah Header berhasil di-insert |
| 3 | `ttkccCreateKoli` | INSERT | Simpan setiap rincian koli (Child Record) ke tabel `ENTBT` | Dipanggil secara berulang (loop) sesuai jumlah baris koli |
| 4 | `ttkccListHeader` | SELECT | Ambil daftar TTK dengan filter tanggal, account, status, limit, dan offset | Dipanggil oleh Controller untuk halaman `TTKCCList.vue` |

*(Catatan: Query `ttkccGetHeader` dan `ttkccGetKoli` dapat ditambahkan sebagai query pendukung jika fitur detail modal / view mode dibuka pada fase integrasi).*

---

### 1.4 Pemetaan Kolom Database ENTBT Terverifikasi

Berdasarkan inspeksi langsung terhadap struktur form sandbox DW (`fnm.50180000ENTBT_m_TTKCC2`) dan subform koli (`5018135703`), berikut adalah pemetaan field form ke kolom database `ENTBT` yang telah terverifikasi:

#### Tabel 1: Header TTK (Form `f5018005322`)
| Kolom ENTBT | Tipe / Format | Field Form TTKCC2 / Payload | Keterangan di Sandbox DW |
|-------------|---------------|-----------------------------|--------------------------|
| `ATRBTTYID` | Integer | (Konstan 5001) | Tipe Transaksi TTK Corporate |
| `ATRBTDT` | Date | `tanggal` | Tanggal Pengiriman |
| `ATRBAIDP` | Integer | `accountId` | ID Account Corporate |
| `ATRBIID` | Integer | `layananId` | ID Layanan Pengiriman |
| `ATRBTNOG` | String | `referensi` | Nomor TTK Manual / Referensi |
| `ATRBTP18` | Integer | `printer` | Format Label Printer (102=Thermal, 103=Kecil, 104=Besar) |
| `ATRBTP21` | Integer | `adaPacking` | Flag Ada Packing/Surcharge (1=Ya, 2=Tidak) |
| `ATRBTP24` | Integer | `buktiKembali` | Bukti Dokumen Kembali (100=Tidak Perlu, 101, dst) |
| `ATRBTP01` | Integer | `asalId` | ID Wilayah Asal |
| `ATRBTP02` | Integer | `tujuanId` | ID Wilayah Tujuan |
| `ATRBTP09` | String | `kodeOps` | Kode Operasional Rute |
| `ATRBTP06` | String | `namaPenerima` | Nama Penerima Barang |
| `ATRBTP07` | String | `telepon` | Nomor Telepon Penerima |
| `ATRBTP08` | String | `alamat` | Alamat Lengkap Penerima |
| `ATRBTP11` | Decimal | `grossWeight` | Berat Kotor Total (Gross Weight KG) |
| `ATRBTQTY` | Decimal | `chargedWeight` | Berat Dikenakan Total (Charged Weight KG) |
| `ATRBTP16` | Integer | `koli` | Jumlah Total Koli |
| `ATRBTP12` | String | `volumetrik` | Dimensi Volumetrik Header (P x L x T) |
| `ATRBTP22` | Decimal | `kubikasi` | Total Volume Kubikasi (M3) |
| `ATRBTP13` | String | `barang` | Nama / Jenis Barang Utama |
| `ATRBTP14` | Decimal | `hargaBarang` | Nilai Pertanggungan / Harga Barang |
| `ATRBTP19` | Integer | `biayaTambahan` | Layanan Tambahan (100=Tanpa, 101=Asuransi, 102=Pickup, 103=Realokasi) |
| `ATRBTP30` | Decimal | `biayaKirimPerKg` | Tarif Biaya Kirim per KG |
| `ATRBTVALU` | Decimal | `biayaKirim` | Subtotal Biaya Kirim Pokok |
| `ATRBTP26` | Decimal | `biayaRealokasi` | Nominal Biaya Realokasi |
| `ATRBTP27` | Decimal | `biayaPickup` | Nominal Biaya Pickup |
| `ATRBTP15` | Decimal | `biayaAsuransi` | Nominal Biaya Asuransi |
| `ATRBTP28` | Decimal | `biayaPacking` | Total Biaya Packing Kayu |
| `ATRBTP29` | Decimal | `biayaSurcharge` | Total Biaya Surcharge |
| `ATRBTP23` | Decimal | `packingBubble` | Total Biaya Packing Bubble Wrap |
| `ATRBTP25` | Decimal | `biayaDiskon` | Nominal Potongan / Diskon |
| `ATRBTVAL` | Decimal | `total` | Grand Total Tagihan |
| `ATRBTP05` | String | `catatan` | Catatan Instruksi Pengiriman |
| `ATRBTP17` | Integer | `maxtm` | Estimasi Lead Time (Hari) |
| `ATRBTSS` | Integer | (Default 1) | Status Transaksi (1 = Baru) |

#### Tabel 2: Child Koli (Subform `5018135703`)
| Kolom ENTBT | Tipe / Format | Properti Item Koli | Keterangan di Sandbox DW |
|-------------|---------------|-------------------|--------------------------|
| `ATRBTTYID` | Integer | (Konstan 5001) | Tipe Transaksi TTK Corporate |
| `ATRBTIDP` | Bigint | `parentBtid` | Relasi ke ID Header (`ENTBT.ATRBTID`) |
| `ATRBTNO` | String | `nomor` | Nomor TTK yang sama dengan header |
| `ATRBTP16` | Integer | `koli` | Nomor urut koli (Koli ke-1, 2, dst) |
| `ATRBTQTY` | Decimal | `grossWeight` | Berat Kotor Koli (KG) |
| `ATRBTP12` | String | `volumetrik` | Dimensi Volumetrik Koli (P x L x T) |
| `ATRBTP22` | Decimal | `kubikasi` | Volume Kubikasi Koli (M3) |
| `ATRBTP04` | Tinyint (0/1) | `isBubble` | Checkbox Penggunaan Bubble Wrap |
| `ATRBTP19` | Tinyint (0/1) | `isKayu` | Checkbox Penggunaan Packing Kayu |
| `ATRBTP13` | String | `jenisBarang` | Nama / Jenis Barang Spesifik Koli |
| `ATRBTCMT` | String | `catatan` | Catatan Khusus untuk Koli Tersebut |
| `ATRBTP20` | Decimal | `beratSurcharge` | Berat Barang yang Terkena Surcharge (KG) |
| `ATRBTP21` | Decimal | `biayaSurcharge` | Nominal Biaya Surcharge Koli |
| `ATRBTP23` | Decimal | `biayaPacking` | Nominal Biaya Packing Kayu Koli |
| `ATRBTP24` | Decimal | `biayaPackingBubble` | Nominal Biaya Packing Bubble Wrap Koli |

---

## 2. API Contract — Wajib Disepakati Dulu

> [!WARNING]
> Kontrak JSON ini mengikat antara Frontend (Dev A) dan Backend (Dev B). Jika terdapat perubahan nama variabel atau struktur payload, kedua pihak harus memperbarui dokumen ini sebelum eksekusi.

### 2.1 Create TTK

```
POST /a/ttkcc/createTTKCC
Content-Type: application/json
Authorization: Bearer <token>
```

**Request Body:**
```json
{
  "tanggal": "2026-11-06",
  "accountId": 188889202,
  "layananId": 7,
  "referensi": "REF-PO-9921",
  "adaPacking": 2,
  "printer": 102,
  "buktiKembali": 100,
  "asalId": 3216,
  "tujuanId": 3171,
  "grossWeight": 26,
  "chargedWeight": 26,
  "koli": 1,
  "volumetrik": "0 x 0 x 0",
  "kubikasi": 1,
  "barang": "SPAREPART MOBIL",
  "hargaBarang": 0,
  "biayaTambahan": 100,
  "namaPenerima": "PT JAYA ABADI",
  "telepon": "08123456789",
  "alamat": "Kawasan Industri MM2100 Blok B2",
  "supplierId": null,
  "kodeOps": "077-159-786-150-700-169-626",
  "maxtm": 2,
  "biayaKirimPerKg": 1700,
  "biayaKirim": 44200,
  "biayaAsuransi": 0,
  "biayaRealokasi": 0,
  "biayaPickup": 0,
  "biayaPacking": 0,
  "packingBubble": 0,
  "biayaSurcharge": 0,
  "biayaDiskon": 0,
  "total": 44200,
  "catatan": "Harap jangan dibanting",
  "listKoli": [
    {
      "koli": 1,
      "grossWeight": 26,
      "volumetrik": "0 x 0 x 0",
      "kubikasi": 1,
      "isBubble": false,
      "isKayu": false,
      "jenisBarang": "SPAREPART MOBIL",
      "catatan": "",
      "beratSurcharge": 0,
      "biayaSurcharge": 0,
      "biayaPacking": 0,
      "biayaPackingBubble": 0
    }
  ]
}
```

**Response Sukses (HTTP 200):**
```json
{
  "code": 0,
  "status": "success",
  "message": "TTK berhasil dibuat",
  "data": {
    "btid": 123456789,
    "nomor": "50180000ENTBT-20261106-001"
  }
}
```

**Response Gagal (HTTP 400 / 500):**
```json
{
  "code": 1,
  "status": "failed",
  "message": "Missing required fields: accountId, asalId, tujuanId"
}
```

---

### 2.2 List TTK

```
GET /a/ttkcc/listTTKCC
Authorization: Bearer <token>
```

**Query Parameters:**

| Parameter | Tipe | Wajib | Keterangan |
|-----------|------|-------|------------|
| `tanggalDari` | String (YYYY-MM-DD) | Tidak | Batas awal tanggal pengiriman |
| `tanggalSampai` | String (YYYY-MM-DD) | Tidak | Batas akhir tanggal pengiriman |
| `accountId` | Integer | Tidak | Filter berdasarkan ID account corporate |
| `status` | Integer | Tidak | Filter status transaksi (1=Baru, dll) |
| `d-limit` | Integer | Tidak | Jumlah baris per halaman (default 25) |
| `d-offset` | Integer | Tidak | Offset paginasi database |

**Response Sukses (HTTP 200):**
```json
{
  "code": 0,
  "status": "success",
  "message": "Successful",
  "data": {
    "body": [
      {
        "btid": 123456789,
        "nomor": "50180000ENTBT-20261106-001",
        "tanggal": "2026-11-06",
        "account": "2100081011: Astra International",
        "asal": "Jawa Barat, Kab. Bekasi",
        "tujuan": "DKI Jakarta, Kota Jakarta Timur",
        "total": 44200,
        "status": 1,
        "statusLabel": "Baru",
        "koli": 1,
        "berat": 26
      }
    ],
    "rows": 1
  }
}
```

---

### 2.3 Get Detail TTK

```
GET /a/ttkcc/getTTKCC/{btid}
Authorization: Bearer <token>
```

**Response Sukses (HTTP 200):**
```json
{
  "code": 0,
  "status": "success",
  "message": "Successful",
  "data": {
    "header": {
      "btid": 123456789,
      "nomor": "50180000ENTBT-20261106-001",
      "tanggal": "2026-11-06",
      "accountId": 188889202,
      "account": "2100081011: Astra International",
      "layananId": 7,
      "layanan": "AISHELL-N",
      "asalId": 3216,
      "asal": "Jawa Barat, Kab. Bekasi",
      "tujuanId": 3171,
      "tujuan": "DKI Jakarta, Kota Jakarta Selatan",
      "grossWeight": 26,
      "chargedWeight": 26,
      "koli": 1,
      "total": 44200,
      "namaPenerima": "PT JAYA ABADI",
      "alamat": "Kawasan Industri MM2100 Blok B2",
      "kodeOps": "077-159-786-150-700-169-626",
      "status": 1
    },
    "koli": [
      {
        "btid": 123456790,
        "koli": 1,
        "grossWeight": 26,
        "volumetrik": "0 x 0 x 0",
        "kubikasi": 1
      }
    ]
  }
}
```

---

## 3. Pembagian Kerja

```
[TAHAP 1: ALIGNMENT]
Sepakati API Contract & Pemetaan Kolom ENTBT.

         |--------------------------------|
         |                                |
         v                                v
  [Dev A - Frontend]              [Dev B - Backend]
  1. Tombol & Validasi Save       1. Tambah 4 Query EDBQ
  2. buildPayload & handleSave    2. Controller TTKCC.pm
  3. Halaman TTKCCList.vue        3. OpenAPI ttkcc.yaml
  4. Routing routes.js            4. Config dcaf.yaml
         |                                |
         |--------------------------------|
         v
  [TAHAP 2: INTEGRATION TEST]
  Colokkan Frontend ke Backend API lokal dev.
         v
  [TAHAP 3: MERGE & DEPLOY]
```

---

## 4. Dev A — Frontend

### 4.1 Modifikasi `TTKCC2.vue`

#### A1. Tambah State Baru di `data()`
Tambahkan pada return object `data()`:
```javascript
// Save state
isSaving: false,
savedNomor: '',
```

#### A2. Tambah Computed `isReadyToSave`
Tambahkan di blok `computed:`:
```javascript
isReadyToSave () {
  return this.isDataLengkap && this.isPricingComplete && !this.isSaving
},
```

#### A3. Tambah Tombol Save di Template
Letakkan di toolbar kanan atas sejajar dengan tab navigasi:
```html
<div class="row items-center q-gutter-x-sm">
  <q-chip
    v-if="savedNomor"
    color="positive"
    text-color="white"
    icon="check_circle"
    :label="savedNomor"
  />
  <q-btn
    color="positive"
    label="Save"
    icon="save"
    :disable="!isReadyToSave"
    :loading="isSaving"
    unelevated
    @click="handleSave"
  >
    <q-tooltip v-if="missingFields.length">
      Belum lengkap: {{ missingFields.join(', ') }}
    </q-tooltip>
  </q-btn>
</div>
```

#### A4. Tambah Method `handleSave()` dan `buildPayload()`
Tambahkan di dalam blok `methods:`:
```javascript
async handleSave () {
  if (!this.isDataLengkap) {
    this.$q.dialog({
      title: 'Data Belum Lengkap',
      message: `Field berikut wajib diisi: ${this.missingFields.join(', ')}`,
      ok: 'OK'
    })
    return
  }

  if (!this.isPricingComplete) {
    this.$q.dialog({
      title: 'Tarif Belum Dihitung',
      message: 'Lengkapi data Layanan, Asal, Tujuan, dan Berat agar tarif terhitung sebelum simpan.',
      ok: 'OK'
    })
    return
  }

  const confirmed = await new Promise(resolve => {
    this.$q.dialog({
      title: 'Konfirmasi',
      message: 'Are you sure to CREATE new record?',
      cancel: { label: 'Cancel', flat: true },
      ok: { label: 'OK', color: 'positive' },
      persistent: true
    })
      .onOk(() => resolve(true))
      .onCancel(() => resolve(false))
  })

  if (!confirmed) return

  this.isSaving = true
  this.$q.loading.show({ message: 'Creating data...' })

  try {
    const payload = this.buildPayload()
    const res = await this.$axios.post('/a/ttkcc/createTTKCC', payload)

    if (res.data && res.data.code === 0) {
      const nomor = res.data.data.nomor
      this.savedNomor = nomor
      this.form.nomor = nomor

      this.$q.notify({
        type: 'positive',
        message: `TTK ${nomor} berhasil dibuat`,
        position: 'top',
        timeout: 5000
      })
    } else {
      this.$q.notify({
        type: 'negative',
        message: res.data.message || 'Gagal menyimpan TTK',
        position: 'top',
        timeout: 5000
      })
    }
  } catch (err) {
    console.error('handleSave error:', err)
    this.$q.notify({
      type: 'negative',
      message: 'Terjadi kesalahan: ' + (err.message || 'Server error'),
      position: 'top',
      timeout: 5000
    })
  } finally {
    this.isSaving = false
    this.$q.loading.hide()
  }
},

buildPayload () {
  const f = this.form
  const getVal = (opt) => {
    if (!opt) return null
    if (typeof opt === 'object') return opt.value !== undefined ? opt.value : opt.id
    return opt
  }

  return {
    tanggal:         f.tanggal,
    accountId:       getVal(f.account),
    layananId:       f.layanan,
    referensi:       f.referensi || '',
    adaPacking:      parseInt(f.adaPacking) || 2,
    printer:         parseInt(f.printer) || 102,
    buktiKembali:    parseInt(f.buktiKembali) || 100,
    asalId:          getVal(f.asal),
    tujuanId:        getVal(f.tujuan),
    grossWeight:     parseFloat(f.grossWeight) || 0,
    chargedWeight:   parseFloat(f.chargedWeight) || 0,
    koli:            parseInt(f.koli) || 1,
    volumetrik:      f.volumetrik || '',
    kubikasi:        parseFloat(f.kubikasi) || 0,
    barang:          f.barang || '',
    hargaBarang:     parseFloat(f.hargaBarang) || 0,
    biayaTambahan:   parseInt(f.biayaTambahan) || 100,
    namaPenerima:    typeof f.namaPenerima === 'object' && f.namaPenerima
                       ? (f.namaPenerima.label || f.namaPenerima)
                       : (f.namaPenerima || ''),
    telepon:         f.teleponPenerima || '',
    alamat:          f.alamatPenerima || '',
    supplierId:      f.supplier || null,
    kodeOps:         f.kodeOps || '',
    maxtm:           parseInt(f.maxtm) || 0,
    biayaKirimPerKg: parseFloat(f.biayaKirimPerKg) || 0,
    biayaKirim:      parseFloat(f.biayaKirim) || 0,
    biayaAsuransi:   parseFloat(f.biayaAsuransi) || 0,
    biayaRealokasi:  parseFloat(f.biayaRealokasi) || 0,
    biayaPickup:     parseFloat(f.biayaPickup) || 0,
    biayaPacking:    parseFloat(f.biayaPacking) || 0,
    packingBubble:   parseFloat(f.packingBubble) || 0,
    biayaSurcharge:  parseFloat(f.biayaSurcharge) || 0,
    biayaDiskon:     parseFloat(f.biayaDiskon) || 0,
    total:           parseFloat(String(f.total || 0).replace(/,/g, '')) || 0,
    catatan:         f.catatan || '',
    listKoli: (this.listKoli || []).map(k => ({
      koli:               parseInt(k.koli) || 1,
      grossWeight:        parseFloat(k.grossWeight) || 0,
      volumetrik:         k.volumetrik || '',
      kubikasi:           parseFloat(k.kubikasi) || 0,
      isBubble:           !!k.isBubble,
      isKayu:             !!k.isKayu,
      jenisBarang:        k.jenisBarang || '',
      catatan:            k.catatan || '',
      beratSurcharge:     parseFloat(k.beratSurcharge) || 0,
      biayaSurcharge:     parseFloat(k.biayaSurcharge) || 0,
      biayaPacking:       parseFloat(k.biayaPacking) || 0,
      biayaPackingBubble: parseFloat(k.biayaPackingBubble) || 0
    }))
  }
}
```

---

### 4.2 Buat File Baru `TTKCCList.vue`
**Path:** `web/dev/dcaf/src/pages/TTKCCList.vue`

Halaman tabel yang memuat data daftar TTK hasil input, dilengkapi pencarian filter tanggal, account corporate, serta tombol aksi untuk membuka detail atau membuat TTK baru.

```vue
<template>
  <q-page class="q-pa-md">
    <!-- Header -->
    <div class="row items-center justify-between q-mb-md">
      <div class="text-h6 text-weight-bold">TTK Corporate Kubikasi — Daftar Transaksi</div>
      <q-btn
        color="positive"
        label="Buat TTK Baru"
        icon="add"
        unelevated
        :to="{ name: 'TTKCC2' }"
      />
    </div>

    <!-- Filter Bar -->
    <q-card flat bordered class="q-mb-md q-pa-sm">
      <div class="row q-col-gutter-sm items-end">
        <div class="col-12 col-md-3">
          <div class="text-caption text-weight-bold q-mb-xs">Tanggal Dari</div>
          <q-input dense outlined type="date" v-model="filter.tanggalDari" />
        </div>
        <div class="col-12 col-md-3">
          <div class="text-caption text-weight-bold q-mb-xs">Tanggal Sampai</div>
          <q-input dense outlined type="date" v-model="filter.tanggalSampai" />
        </div>
        <div class="col-12 col-md-4">
          <div class="text-caption text-weight-bold q-mb-xs">Account Corporate</div>
          <q-select
            dense
            outlined
            v-model="filter.accountId"
            :options="optionsAccount"
            use-input
            fill-input
            hide-selected
            input-debounce="400"
            @filter="filterAccount"
            clearable
            placeholder="Cari nama atau kode account..."
          />
        </div>
        <div class="col-12 col-md-2">
          <q-btn
            color="primary"
            label="Cari"
            icon="search"
            unelevated
            class="full-width"
            :loading="loading"
            @click="loadList"
          />
        </div>
      </div>
    </q-card>

    <!-- Data Table -->
    <q-table
      flat
      bordered
      dense
      :data="listData"
      :columns="columns"
      row-key="btid"
      :loading="loading"
      :pagination.sync="pagination"
      @request="onRequest"
      binary-state-sort
    >
      <template v-slot:body-cell-nomor="props">
        <q-td :props="props">
          <q-btn
            flat
            dense
            no-caps
            color="primary"
            :label="props.value"
            @click="bukaDetail(props.row)"
          />
        </q-td>
      </template>

      <template v-slot:body-cell-total="props">
        <q-td :props="props" class="text-right">
          Rp {{ formatNumber(props.value) }}
        </q-td>
      </template>

      <template v-slot:body-cell-statusLabel="props">
        <q-td :props="props">
          <q-badge :color="statusColor(props.row.status)" :label="props.value" />
        </q-td>
      </template>

      <template v-slot:no-data>
        <div class="full-width text-center q-pa-lg text-grey-6">
          <q-icon name="inbox" size="48px" /><br>
          Belum ada data TTK yang ditemukan
        </div>
      </template>
    </q-table>
  </q-page>
</template>

<script>
export default {
  name: 'TTKCCList',

  data () {
    return {
      loading: false,
      listData: [],
      optionsAccount: [],

      filter: {
        tanggalDari:   new Date().toISOString().slice(0, 10),
        tanggalSampai: new Date().toISOString().slice(0, 10),
        accountId:     null
      },

      pagination: {
        sortBy:      'tanggal',
        descending:  true,
        page:        1,
        rowsPerPage: 25,
        rowsNumber:  0
      },

      columns: [
        { name: 'nomor',       label: 'Nomor TTK',   field: 'nomor',       align: 'left',   sortable: true },
        { name: 'tanggal',     label: 'Tanggal',     field: 'tanggal',     align: 'center', sortable: true },
        { name: 'account',     label: 'Account',     field: 'account',     align: 'left' },
        { name: 'asal',        label: 'Asal',        field: 'asal',        align: 'left' },
        { name: 'tujuan',      label: 'Tujuan',      field: 'tujuan',      align: 'left' },
        { name: 'koli',        label: 'Koli',        field: 'koli',        align: 'center' },
        { name: 'berat',       label: 'Berat (KG)',  field: 'berat',       align: 'right' },
        { name: 'total',       label: 'Total Tagihan', field: 'total',     align: 'right',  sortable: true },
        { name: 'statusLabel', label: 'Status',      field: 'statusLabel', align: 'center' }
      ]
    }
  },

  mounted () {
    this.loadList()
  },

  methods: {
    async loadList () {
      this.loading = true
      try {
        const params = {
          tanggalDari:   this.filter.tanggalDari,
          tanggalSampai: this.filter.tanggalSampai,
          'accountId':   this.filter.accountId
            ? (typeof this.filter.accountId === 'object' ? this.filter.accountId.value : this.filter.accountId)
            : undefined,
          'd-limit':     this.pagination.rowsPerPage,
          'd-offset':    (this.pagination.page - 1) * this.pagination.rowsPerPage
        }
        const res = await this.$axios.get('/a/ttkcc/listTTKCC', { params })
        if (res.data && res.data.code === 0) {
          this.listData = res.data.data.body || []
          this.pagination.rowsNumber = res.data.data.rows || 0
        } else {
          this.$q.notify({ type: 'negative', message: res.data.message || 'Gagal memuat data TTK' })
        }
      } catch (err) {
        console.error('loadList error:', err)
        this.$q.notify({ type: 'negative', message: 'Gagal menghubungi server' })
      } finally {
        this.loading = false
      }
    },

    onRequest (props) {
      const { page, rowsPerPage, sortBy, descending } = props.pagination
      this.pagination.page = page
      this.pagination.rowsPerPage = rowsPerPage
      this.pagination.sortBy = sortBy
      this.pagination.descending = descending
      this.loadList()
    },

    async filterAccount (val, update) {
      if (!val || val.length < 3) {
        update(() => { this.optionsAccount = [] })
        return
      }
      try {
        const res = await this.$axios.get('query/RELBABTP_AccountActive', {
          params: { search: val, 'd-limit': 50 }
        })
        const rows = (res.data && res.data.data && res.data.data.body) || []
        update(() => {
          this.optionsAccount = rows.map(item => ({
            label: item.account || item.Account || item.code || item.id,
            value: item.id || item.ID
          }))
        })
      } catch (err) {
        update(() => { this.optionsAccount = [] })
      }
    },

    bukaDetail (row) {
      this.$router.push({ name: 'TTKCC2', query: { btid: row.btid } })
    },

    formatNumber (val) {
      if (!val && val !== 0) return '0'
      return Number(val).toLocaleString('id-ID')
    },

    statusColor (status) {
      const map = { 1: 'blue', 2: 'green', 3: 'orange', 9: 'red' }
      return map[status] || 'grey'
    }
  }
}
</script>
```

---

### 4.3 Registrasi Route di `web/dev/dcaf/src/router/routes.js`
Tambahkan route `TTKCCList`:
```javascript
{ name: 'TTKCC2', path: '/TTKCC2', component: () => import('pages/TTKCC2.vue') },
{ name: 'TTKCCList', path: '/TTKCCList', component: () => import('pages/TTKCCList.vue') },
```

---

## 5. Dev B — Backend

### 5.1 Buat Controller `lib/perl/DCAF/Controller/TTKCC.pm`

```perl
package DCAF::Controller::TTKCC;

use strict;
use parent 'DCAF::Controller';
use DCAF::Query;

sub new {
    my ($roc, %ha) = @_;
    my $ros = $roc->SUPER::new(%ha);
    bless $ros, $roc;
    $ros->{className} = ref($ros);
    return $ros;
}

sub run {
    my ($ros) = @_;
    $ros->log('SA');

    my $rhqp = $ros->{q}->{q}->query_parameters;
    my $rhcp = $DCAF::ros->{json}->decode($ros->{q}->{q}->content || '{}');
    my $rhpp = $ros->{api}->{pathParam};

    switch: {

        # ----------------------------------------------------------------
        # createTTKCC - POST /a/ttkcc/createTTKCC
        # ----------------------------------------------------------------
        $ros->{api}->{operation} eq 'createTTKCC' && do {

            # 1. Validasi field wajib
            my @missing;
            push @missing, 'accountId'   unless $rhcp->{accountId};
            push @missing, 'layananId'   unless $rhcp->{layananId};
            push @missing, 'asalId'      unless $rhcp->{asalId};
            push @missing, 'tujuanId'    unless $rhcp->{tujuanId};
            push @missing, 'namaPenerima' unless $rhcp->{namaPenerima};
            push @missing, 'alamat'      unless $rhcp->{alamat};
            push @missing, 'grossWeight' unless $rhcp->{grossWeight};
            push @missing, 'koli'        unless $rhcp->{koli};

            if (@missing) {
                $ros->{q}->{e}->{status} = 400;
                $ros->{q}->{e}->{num}    = "$ros->{className}/createTTKCC/01";
                $ros->{q}->{e}->{msg}    = 'Missing required fields: ' . join(', ', @missing);
                die;
            }

            # 2. INSERT Header TTK (ENTBT)
            my $roqHeader = DCAF::Query->new(
                q    => $ros->{q},
                name => 'ttkccCreateHeader',
                data => {
                    tanggal         => $rhcp->{tanggal},
                    accountId       => $rhcp->{accountId},
                    layananId       => $rhcp->{layananId},
                    referensi       => $rhcp->{referensi} || '',
                    printer         => $rhcp->{printer} || 102,
                    adaPacking      => $rhcp->{adaPacking} || 2,
                    buktiKembali    => $rhcp->{buktiKembali} || 100,
                    asalId          => $rhcp->{asalId},
                    tujuanId        => $rhcp->{tujuanId},
                    kodeOps         => $rhcp->{kodeOps} || '',
                    namaPenerima    => $rhcp->{namaPenerima},
                    telepon         => $rhcp->{telepon} || '',
                    alamat          => $rhcp->{alamat},
                    grossWeight     => $rhcp->{grossWeight},
                    chargedWeight   => $rhcp->{chargedWeight} || $rhcp->{grossWeight},
                    koli            => $rhcp->{koli},
                    volumetrik      => $rhcp->{volumetrik} || '',
                    kubikasi        => $rhcp->{kubikasi} || 0,
                    barang          => $rhcp->{barang} || '',
                    hargaBarang     => $rhcp->{hargaBarang} || 0,
                    biayaTambahan   => $rhcp->{biayaTambahan} || 100,
                    biayaKirimPerKg => $rhcp->{biayaKirimPerKg} || 0,
                    biayaKirim      => $rhcp->{biayaKirim} || 0,
                    biayaRealokasi  => $rhcp->{biayaRealokasi} || 0,
                    biayaPickup     => $rhcp->{biayaPickup} || 0,
                    biayaAsuransi   => $rhcp->{biayaAsuransi} || 0,
                    biayaPacking    => $rhcp->{biayaPacking} || 0,
                    biayaSurcharge  => $rhcp->{biayaSurcharge} || 0,
                    packingBubble   => $rhcp->{packingBubble} || 0,
                    biayaDiskon     => $rhcp->{biayaDiskon} || 0,
                    total           => $rhcp->{total} || 0,
                    catatan         => $rhcp->{catatan} || '',
                    maxtm           => $rhcp->{maxtm} || 0,
                }
            );
            $roqHeader->run || die;

            unless ($roqHeader->{result}->{rows}) {
                $ros->{q}->{e}->{status} = 500;
                $ros->{q}->{e}->{num}    = "$ros->{className}/createTTKCC/02";
                $ros->{q}->{e}->{msg}    = 'Gagal menyimpan header TTK ke database';
                die;
            }

            my $newBtid = $roqHeader->{result}->{insertId};

            # 3. Ambil nomor TTK yang terbentuk
            my $rhresNomor = {};
            $ros->q2j('ttkccGetNomor', { btid => $newBtid }, { hashed => 1 }, $rhresNomor);
            my $rhttk = shift @{$rhresNomor->{result}->{body}};

            unless ($rhttk && $rhttk->{nomor}) {
                $ros->{q}->{e}->{status} = 500;
                $ros->{q}->{e}->{num}    = "$ros->{className}/createTTKCC/03";
                $ros->{q}->{e}->{msg}    = 'Header tersimpan tetapi gagal mengambil nomor TTK';
                die;
            }

            my $nomorTTK = $rhttk->{nomor};

            # 4. INSERT Child Koli (Loop)
            my $listKoli = $rhcp->{listKoli} || [];
            foreach my $rhkoli (@{$listKoli}) {
                next unless $rhkoli->{koli} && $rhkoli->{grossWeight};
                my $roqKoli = DCAF::Query->new(
                    q    => $ros->{q},
                    name => 'ttkccCreateKoli',
                    data => {
                        parentBtid         => $newBtid,
                        nomor              => $nomorTTK,
                        koli               => $rhkoli->{koli} || 1,
                        grossWeight        => $rhkoli->{grossWeight} || 0,
                        volumetrik         => $rhkoli->{volumetrik} || '',
                        kubikasi           => $rhkoli->{kubikasi} || 0,
                        isBubble           => $rhkoli->{isBubble} ? 1 : 0,
                        isKayu             => $rhkoli->{isKayu} ? 1 : 0,
                        jenisBarang        => $rhkoli->{jenisBarang} || '',
                        catatan            => $rhkoli->{catatan} || '',
                        beratSurcharge     => $rhkoli->{beratSurcharge} || 0,
                        biayaSurcharge     => $rhkoli->{biayaSurcharge} || 0,
                        biayaPacking       => $rhkoli->{biayaPacking} || 0,
                        biayaPackingBubble => $rhkoli->{biayaPackingBubble} || 0,
                    }
                );
                $roqKoli->run;
            }

            $ros->{templateData}->{jstext} = $DCAF::ros->{json}->encode({
                code    => 0,
                status  => 'success',
                message => 'TTK berhasil dibuat',
                data    => {
                    btid  => $newBtid,
                    nomor => $nomorTTK
                }
            });
            last switch;
        };

        # ----------------------------------------------------------------
        # listTTKCC - GET /a/ttkcc/listTTKCC
        # ----------------------------------------------------------------
        $ros->{api}->{operation} eq 'listTTKCC' && do {
            my $rhpar = {
                tanggalDari   => $rhqp->{'tanggalDari'} || '',
                tanggalSampai => $rhqp->{'tanggalSampai'} || '',
                accountId     => $rhqp->{'accountId'} || 0,
                status        => $rhqp->{'status'} || 0,
            };

            my $rhres = {};
            $ros->q2j('ttkccListHeader', $rhpar, {
                hashed => 1,
                limit  => $rhqp->{'d-limit'} || 25,
                offset => $rhqp->{'d-offset'} || 0,
            }, $rhres);

            $ros->{templateData}->{jstext} = $DCAF::ros->{json}->encode({
                code    => 0,
                status  => 'success',
                message => 'Successful',
                data    => $rhres->{result}
            });
            last switch;
        };

        # ----------------------------------------------------------------
        # getTTKCC - GET /a/ttkcc/getTTKCC/{btid}
        # ----------------------------------------------------------------
        $ros->{api}->{operation} eq 'getTTKCC' && do {
            my $btid = $rhpp->{btid};
            unless ($btid) {
                $ros->{q}->{e}->{status} = 400;
                $ros->{q}->{e}->{num}    = "$ros->{className}/getTTKCC/01";
                $ros->{q}->{e}->{msg}    = 'Parameter btid wajib diisi';
                die;
            }

            my $rhresHeader = {};
            $ros->q2j('ttkccGetHeader', { btid => $btid }, { hashed => 1 }, $rhresHeader);
            my $rhheader = shift @{$rhresHeader->{result}->{body}};

            unless ($rhheader) {
                $ros->{q}->{e}->{status} = 404;
                $ros->{q}->{e}->{num}    = "$ros->{className}/getTTKCC/02";
                $ros->{q}->{e}->{msg}    = 'Data TTK tidak ditemukan';
                die;
            }

            my $rhresKoli = {};
            $ros->q2j('ttkccGetKoli', { parentBtid => $btid }, { hashed => 1 }, $rhresKoli);

            $ros->{templateData}->{jstext} = $DCAF::ros->{json}->encode({
                code    => 0,
                status  => 'success',
                message => 'Successful',
                data    => {
                    header => $rhheader,
                    koli   => $rhresKoli->{result}->{body} || []
                }
            });
            last switch;
        };

        # Default fallback
        $ros->{templateData}->{jstext} = $DCAF::ros->{json}->encode({
            code    => 1,
            status  => 'failed',
            message => 'Operasi tidak dikenal',
            data    => {}
        });
    };

    $ros->log('SO');
    $ros->render || die;
    return 1;
}

1;
```

---

### 5.2 Tambah SQL Queries ke `lib/db/git/dcaf-data.sql`

Tambahkan query ini pada blok tabel `EDBQ`. Pastikan ID EDBQ berurutan setelah record terakhir yang terpasang:

```sql
-- ============================================================
-- Modul TTKCC2 Queries (Tabel EDBQ)
-- ============================================================

-- Query 1: Create Header TTK Corporate Kubikasi
INSERT INTO `EDBQ` VALUES (7001,3,'ttkccCreateHeader','TTKCC: Insert Header ENTBT',
'INSERT INTO ta15_wlogisticsidb_ap.ENTBT
SET
  ATRBTTYID = 5001,
  ATRBTDT   = ?,
  ATRBAIDP  = ?,
  ATRBIID   = ?,
  ATRBTNOG  = ?,
  ATRBTP18  = ?,
  ATRBTP21  = ?,
  ATRBTP24  = ?,
  ATRBTP01  = ?,
  ATRBTP02  = ?,
  ATRBTP09  = ?,
  ATRBTP06  = ?,
  ATRBTP07  = ?,
  ATRBTP08  = ?,
  ATRBTP11  = ?,
  ATRBTQTY  = ?,
  ATRBTP16  = ?,
  ATRBTP12  = ?,
  ATRBTP22  = ?,
  ATRBTP13  = ?,
  ATRBTP14  = ?,
  ATRBTP19  = ?,
  ATRBTP30  = ?,
  ATRBTVALU = ?,
  ATRBTP26  = ?,
  ATRBTP27  = ?,
  ATRBTP15  = ?,
  ATRBTP28  = ?,
  ATRBTP29  = ?,
  ATRBTP23  = ?,
  ATRBTP25  = ?,
  ATRBTVAL  = ?,
  ATRBTP05  = ?,
  ATRBTP17  = ?,
  ATRBTSS   = 1,
  ATRBTDTP  = NOW()
',
'tanggal,accountId,layananId,referensi,printer,adaPacking,buktiKembali,asalId,tujuanId,kodeOps,namaPenerima,telepon,alamat,grossWeight,chargedWeight,koli,volumetrik,kubikasi,barang,hargaBarang,biayaTambahan,biayaKirimPerKg,biayaKirim,biayaRealokasi,biayaPickup,biayaAsuransi,biayaPacking,biayaSurcharge,packingBubble,biayaDiskon,total,catatan,maxtm',
0,NULL,NULL,NOW());

-- Query 2: Get nomor TTK dari ID yang di-generate
INSERT INTO `EDBQ` VALUES (7002,3,'ttkccGetNomor','TTKCC: Ambil ATRBTNO / ATRBTNOG by ID',
'SELECT
  ATRBTID             btid,
  COALESCE(ATRBTNO, ATRBTNOG, ATRBTID) nomor
FROM
  ta15_wlogisticsidb_ap.ENTBT
WHERE
  ATRBTID = ?
',
'btid',
0,NULL,NULL,NOW());

-- Query 3: Create Child Koli Detail
INSERT INTO `EDBQ` VALUES (7003,3,'ttkccCreateKoli','TTKCC: Insert Child Koli ENTBT',
'INSERT INTO ta15_wlogisticsidb_ap.ENTBT
SET
  ATRBTTYID = 5001,
  ATRBTIDP  = ?,
  ATRBTNO   = ?,
  ATRBTP16  = ?,
  ATRBTQTY  = ?,
  ATRBTP12  = ?,
  ATRBTP22  = ?,
  ATRBTP04  = ?,
  ATRBTP19  = ?,
  ATRBTP13  = ?,
  ATRBTCMT  = ?,
  ATRBTP20  = ?,
  ATRBTP21  = ?,
  ATRBTP23  = ?,
  ATRBTP24  = ?,
  ATRBTSS   = 1,
  ATRBTDTP  = NOW()
',
'parentBtid,nomor,koli,grossWeight,volumetrik,kubikasi,isBubble,isKayu,jenisBarang,catatan,beratSurcharge,biayaSurcharge,biayaPacking,biayaPackingBubble',
0,NULL,NULL,NOW());

-- Query 4: List TTK Header dengan Filter dan Paginasi
INSERT INTO `EDBQ` VALUES (7004,3,'ttkccListHeader','TTKCC: List Header TTK Corporate Kubikasi',
'SELECT
  t.ATRBTID          btid,
  COALESCE(t.ATRBTNO, t.ATRBTNOG, t.ATRBTID) nomor,
  date(t.ATRBTDT)    tanggal,
  concat(ba.ATRBACD, '': '', be.ATRBENM) account,
  oa.ATRCAREANML     asal,
  da.ATRCAREANML     tujuan,
  t.ATRBTQTY         berat,
  t.ATRBTP16         koli,
  t.ATRBTVAL         total,
  t.ATRBTSS          status,
  lv.ATRLVNM         statusLabel
FROM
  ta15_wlogisticsidb_ap.ENTBT t
  LEFT JOIN swb.ENTBA ba ON ba.ATRBAID = t.ATRBAIDP
  LEFT JOIN swb.ENTBE be ON be.ATRBEID = ba.ATRBEID
  LEFT JOIN swb.ENTCAREA oa ON oa.ATRCAREAID = t.ATRBTP01
  LEFT JOIN swb.ENTCAREA da ON da.ATRCAREAID = t.ATRBTP02
  LEFT JOIN swb.ENTLV lv ON lv.ATRLVTYID = 9 AND lv.ATRLVCD = t.ATRBTSS
WHERE
  t.ATRBTTYID = 5001
  AND (t.ATRBTIDP = 0 OR t.ATRBTIDP IS NULL)
  AND (? = '''' OR date(t.ATRBTDT) >= ?)
  AND (? = '''' OR date(t.ATRBTDT) <= ?)
  AND (? = 0 OR t.ATRBAIDP = ?)
  AND (? = 0 OR t.ATRBTSS = ?)
ORDER BY
  t.ATRBTDT DESC, t.ATRBTID DESC
',
'tanggalDari,tanggalDari,tanggalSampai,tanggalSampai,accountId,accountId,status,status',
0,NULL,NULL,NOW());
```

---

### 5.3 Buat File Spesifikasi `etc/api/ttkcc.yaml`

```yaml
# dcaf:01
openapi: 3.0.0
info:
  version: 1.0.0
  title: TTKCC
  description: TTK Corporate Kubikasi API
  contact:
    email: aab@pirantimaya.id
servers:
  - url: https://swb.stage.wahana.com/a/ttkcc
  - url: https://swbprod.stage.wahana.com/a/ttkcc
  - url: https://swb.wahana.com/a/ttkcc
  - url: https://localhost:44311/a/ttkcc
  - url: https://swb-rc.wahana.com/a/ttkcc
tags:
  - name: TTKCC
    description: TTK Corporate Kubikasi
components:
  securitySchemes:
    bearer:
      type: http
      scheme: bearer
paths:
  /createTTKCC:
    post:
      tags:
        - TTKCC
      summary: Buat record TTK Corporate Kubikasi baru
      operationId: createTTKCC/TTKCC
      security:
        - bearer: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [tanggal, accountId, layananId, asalId, tujuanId, namaPenerima, alamat, grossWeight, koli]
      responses:
        200:
          description: Berhasil membuat TTK

  /listTTKCC:
    get:
      tags:
        - TTKCC
      summary: Daftar TTK Corporate Kubikasi
      operationId: listTTKCC/TTKCC
      security:
        - bearer: []
      parameters:
        - in: query
          name: tanggalDari
          schema: { type: string, format: date }
        - in: query
          name: tanggalSampai
          schema: { type: string, format: date }
        - in: query
          name: accountId
          schema: { type: integer }
        - in: query
          name: d-limit
          schema: { type: integer, default: 25 }
        - in: query
          name: d-offset
          schema: { type: integer, default: 0 }
      responses:
        200:
          description: Berhasil memuat daftar TTK

  /getTTKCC/{btid}:
    get:
      tags:
        - TTKCC
      summary: Ambil detail TTK Corporate Kubikasi
      operationId: getTTKCC/TTKCC
      security:
        - bearer: []
      parameters:
        - in: path
          name: btid
          required: true
          schema: { type: integer }
      responses:
        200:
          description: Berhasil memuat detail TTK
```

---

### 5.4 Registrasi Endpoint di `etc/dcaf.yaml`
Tambahkan konfigurasi route modul `ttkcc` di bawah section `api:`:
```yaml
api:
   dcaf: /etc/api/dcaf.yaml
   3pl: /etc/api/3pl.yaml
   ttkcc: /etc/api/ttkcc.yaml    # Registrasi controller TTKCC
```

---

## 6. Urutan Eksekusi & Checkpoint

### Phase 1 — Setup & Review
- [ ] Review bersama dokumen ini dan sepakati 4 query database baru.
- [ ] Buat branch git masing-masing di repositori `swb`:
  - Frontend: `feature/ttkcc2-save-frontend`
  - Backend: `feature/ttkcc2-save-backend`

### Phase 2 — Parallel Development
**Checklist Dev A (Frontend):**
- [ ] Modifikasi `TTKCC2.vue` (tambah state `isSaving`, computed `isReadyToSave`, tombol Save).
- [ ] Implementasi fungsi `handleSave` dan `buildPayload` di `TTKCC2.vue`.
- [ ] Buat halaman baru `TTKCCList.vue`.
- [ ] Daftarkan route di `web/dev/dcaf/src/router/routes.js`.
- [ ] Uji visual form & tombol di browser (Dev Server Quasar).

**Checklist Dev B (Backend):**
- [ ] Masukkan 4 query SQL ke database dev / migration script `dcaf-data.sql`.
- [ ] Buat class controller `lib/perl/DCAF/Controller/TTKCC.pm`.
- [ ] Jalankan uji sintaks: `perl -c -Ilib/perl lib/perl/DCAF/Controller/TTKCC.pm`.
- [ ] Buat file OpenAPI `etc/api/ttkcc.yaml`.
- [ ] Daftarkan modul di `etc/dcaf.yaml`.
- [ ] Restart service uWSGI dev dan uji endpoint via Postman / curl.

### Phase 3 — Integration Testing
- [ ] Isi form lengkap di `TTKCC2.vue` lalu klik Save.
- [ ] Verifikasi dialog konfirmasi muncul: "Are you sure to CREATE new record?".
- [ ] Verifikasi data tersimpan di tabel `ENTBT` (baik header maupun child koli).
- [ ] Verifikasi notifikasi nomor TTK tampil di UI form.
- [ ] Buka halaman `TTKCCList.vue` dan pastikan record TTK baru muncul pada tabel.
- [ ] Cek filter tanggal dan filter account pada `TTKCCList.vue`.

### Phase 4 — Merge ke Main
- [ ] Dev B membuka Pull Request (Backend) terlebih dahulu.
- [ ] Dev A membuka Pull Request (Frontend).
- [ ] Cross-review kode antar developer, lakukan merge ke branch utama.

---

## 7. Hal Kritis & Catatan Verifikasi

> [!CAUTION]
> Perhatikan hal-hal berikut sebelum melakukan deploy ke staging / production:

1. **Pemetaan Kolom Sudah Terkonfirmasi:**
   Kolom `ENTBT` baik untuk Header (`f5018005322`) maupun Child Koli (`5018135703`) telah diverifikasi langsung dari struktur form HTML sandbox DW lama. Tidak lagi menggunakan asumsi sementara.
2. **Nomor TTK Generator:**
   Query `ttkccGetNomor` menggunakan fallback `COALESCE(ATRBTNO, ATRBTNOG, ATRBTID)`. Jika tabel `ENTBT` di server database telah dipasangi trigger database auto-numbering, `ATRBTNO` otomatis terisi. Jika belum, nomor sementara menggunakan fallback ID atau nomor referensi.
3. **Privilege Database User `dwtb`:**
   Query `ttkccListHeader` melakukan join antara database `ta15_wlogisticsidb_ap` dan tabel referensi `swb` (`ENTBA`, `ENTBE`, `ENTCAREA`, `ENTLV`). Pastikan user database yang tertera di `etc/dcaf.yaml` memiliki hak akses `SELECT` dan `INSERT` ke database `ta15_wlogisticsidb_ap`.
4. **Disiplin Gaya Kode:**
   Sesuai aturan arsitektur SWB, tidak boleh ada penulisan string SQL di dalam controller Perl. Gunakan selalu `DCAF::Query` atau `$ros->q2j()`.

---
*Dokumen ini diperbarui: 2026-11-06 — Status query terverifikasi dan siap dieksekusi.*
