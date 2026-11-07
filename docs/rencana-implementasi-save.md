# Rencana Implementasi: Fitur Save + List TTKCC2 (Diselaraskan dengan Rules .agents & .ai)

> [!IMPORTANT]
> Dokumen ini adalah panduan teknis dan eksekusi dua developer yang bekerja paralel.
> Seluruh isi dokumen ini telah diselaraskan 100% dengan aturan terbaru pada `.agents/rules/` (`01` s/d `05`), `.agents/GEMINI.md`, dan `.ai/`.
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
   - 4.1 [Modifikasi `TTKCC2.vue`](#41-modifikasi-ttkcc2vue)
   - 4.2 [Buat File Baru `PVwTTKCCLs.vue`](#42-buat-file-baru-pvwttkcclsvue)
   - 4.3 [Registrasi Route di `routes.js`](#43-registrasi-route-di-webdevdcafsrcrouterroutesjs)
5. [Dev B — Backend](#5-dev-b--backend)
   - 5.1 [Buat Controller `TTKCC.pm`](#51-buat-controller-libperldcafcontrollerttkccpm)
   - 5.2 [Definisi SQL Query EDBQ & EDBQP (Aturan 253)](#52-definisi-sql-query-edbq--edbqp-aturan-253)
   - 5.3 [Spesifikasi OpenAPI `ttkcc.yaml`](#53-buat-file-spesifikasi-etcapittkccyaml)
   - 5.4 [Registrasi Gateway `dcaf.yaml`](#54-registrasi-endpoint-di-etcdcafyaml)
6. [Urutan Eksekusi & Checkpoint](#6-urutan-eksekusi--checkpoint)
7. [Hal Kritis & Catatan Verifikasi](#7-hal-kritis--catatan-verifikasi)

---

## 1. Gambaran Besar & Arsitektur Fitur

### 1.1 Fitur yang Akan Dibuat

| Fitur | Keterangan |
|-------|------------|
| Tombol Save di TTKCC2 | Tombol hijau di kanan atas toolbar form `TTKCC2.vue` |
| Validasi form sebelum save | Cek field wajib, cek kelengkapan data & kalkulasi tarif |
| Konfirmasi dialog | Dialog pop-up: "Are you sure to CREATE new record?" (sama persis dengan sandbox DW) |
| Loading overlay saat proses | "Creating data..." overlay saat HTTP request berlangsung |
| Notifikasi hasil | Banner / popup sukses dengan nomor TTK baru atau pesan gagal |
| Reset & feedback form | Nomor TTK tampil di chip hijau atas, siap cetak resi / buat baru |
| Halaman List TTK (`PVwTTKCCLs.vue`) | Tabel daftar riwayat TTK corporate kubikasi yang telah tersimpan |
| Filter data list | Filter berdasarkan rentang tanggal, Account corporate, dan status |
| Navigasi detail | Klik baris nomor TTK membuka halaman TTKCC2 dalam mode referensi / detail |

---

### 1.2 File yang Akan Dibuat / Dimodifikasi

| File | Pemilik | Status | Deskripsi |
|------|---------|--------|-----------|
| `web/dev/dcaf/src/pages/TTKCC2.vue` | Dev A | Dimodifikasi | Tambah tombol Save, konfirmasi, method save & payload builder |
| `web/dev/dcaf/src/pages/PVwTTKCCLs.vue` | Dev A | **File Baru** | Halaman daftar data TTK, filter, dan tabel riwayat (standar `PVw...Ls`) |
| `web/dev/dcaf/src/router/routes.js` | Dev A | Dimodifikasi | Registrasi route `/PVwTTKCCLs` (1 baris) |
| `lib/perl/DCAF/Controller/TTKCC.pm` | Dev B | **File Baru** | Controller API DCAF: operasi `createTTKCC`, `listTTKCC`, `getTTKCC` |
| `etc/api/ttkcc.yaml` | Dev B | **File Baru** | OpenAPI 3.0 specification untuk routing DCAF |
| `etc/dcaf.yaml` | Dev B | Dimodifikasi | Registrasi route modul `ttkcc` ke DCAF gateway (1 baris) |
| `lib/db/git/dcaf-data.sql` | Dev B | Dimodifikasi | Menambahkan query SELECT baru (`253xxx`) ke tabel `EDBQ` dan `EDBQP` |

---

### 1.3 Analisis & Klasifikasi Query Database (Existing vs Baru)

Sistem SWB DCAF tidak mengizinkan penulisan raw SQL di dalam file Perl (`.pm`). Semua query harus terdaftar di tabel database `EDBQ` dan dieksekusi menggunakan method `$ros->q2j('queryName', ...)` atau `DCAF::Query`.

Sesuai aturan di `.agents/rules/03-database-rules.md`:

#### A. Query Form Input & Lookup (0 Query Baru — Semua Sudah Ada)
1. `whnPriceListBaru` — Hitung tarif dasar. Sudah ada.
2. `whnPriceListAdditional` — Hitung tarif surcharge & packing. Sudah ada.
3. `whnGetRouteListByAccV2` — Dropdown rute layanan berdasarkan account. Sudah ada.
4. `RELBIBT_TTKC` — Dropdown tipe layanan TTK Corporate. Sudah ada.
5. `RELBABTP_AccountActive` — Autocomplete pencarian account corporate. Sudah ada.
6. `pllvWilayahBEAD` — Autocomplete wilayah asal dan tujuan. Sudah ada.
7. `pllv_shiptoaddr` — Dropdown alamat shipto supplier / penerima. Sudah ada.
8. `RELBEBANKBTP` — Dropdown data bank account. Sudah ada.
9. `pllv_dokkembali` — Dropdown pilihan dokumen bukti kembali. Sudah ada.

#### B. Query INSERT ENTBT (0 Query Baru — WAJIB Menggunakan Generic Query)
* **Aturan `03-database-rules.md` Baris 341:**
  > `entbtCreate` (ID `253005`) adalah generic insert ke `ENTBT` di DB `ta15_wlogisticsidb_ap` (connection 3). Gunakan untuk setiap insert baru ke `ENTBT`, dilarang membuat query INSERT baru.
* **Keputusan:**
  Operasi simpan Header TTK dan Child Koli **WAJIB memakai `entbtCreate`**. Parameter yang dikirimkan adalah nama kolom `ENTBT`.

#### C. Query Baru yang Wajib Didaftarkan (Hanya 2 Query SELECT Baru)
Mengikuti **The 253 Query ID Rule** (ID di rentang `253000` - `253999`), serta mendaftarkan seluruh parameter ke `EDBQP`:

| No | Nama Query | ID EDBQ | Tipe | Tujuan Penggunaan |
|----|------------|---------|------|-------------------|
| 1 | `ttkccGetNomor` | `253006` | SELECT | Ambil nomor TTK (`ATRBTNO`) berdasarkan `ATRBTID` |
| 2 | `ttkccListHeader` | `253007` | SELECT | Daftar riwayat TTK header dengan filter tanggal, account, dan status untuk `PVwTTKCCLs.vue` |

---

### 1.4 Pemetaan Kolom Database ENTBT Terverifikasi

#### Tabel 1: Header TTK (Disimpan via `entbtCreate`)
| Kolom ENTBT | Tipe Data | Field Payload Form TTKCC2 | Keterangan |
|-------------|-----------|---------------------------|------------|
| `ATRBAIDO` | Integer | (Default 3002) | ID Cabang Origin / Owner Account |
| `ATRBAIDM` | Integer | (Session User / Default 1) | ID User Pembuat Transaksi |
| `ATRBTTYID` | Integer | (Konstan 5001) | Tipe Transaksi TTK Corporate |
| `ATRBTDT` | Date | `tanggal` | Tanggal Pengiriman |
| `ATRBAIDP` | Integer | `accountId` | ID Account Corporate |
| `ATRBIID` | Integer | `layananId` | ID Layanan Pengiriman |
| `ATRBTNOG` | String | `referensi` | Nomor TTK Manual / Ref PO / DN |
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
| `ATRBTBS` | Integer | (Default 1) | Status Bisnis |
| `ATRBTDTP` | DateTime | (NOW()) | Tanggal Posting |

#### Tabel 2: Child Koli (Disimpan via `entbtCreate` per baris koli)
| Kolom ENTBT | Tipe Data | Properti Item Koli | Keterangan |
|-------------|-----------|-------------------|------------|
| `ATRBAIDO` | Integer | (Default 3002) | ID Cabang Origin / Owner Account |
| `ATRBAIDM` | Integer | (Session User / Default 1) | ID User Pembuat Transaksi |
| `ATRBTTYID` | Integer | (Konstan 5001) | Tipe Transaksi TTK Corporate |
| `ATRBAIDP` | Integer | `accountId` | Mengikuti ID Account Header |
| `ATRBTIDP` | Bigint | `parentBtid` | Relasi ke ID Header (`ENTBT.ATRBTID`) |
| `ATRBTNO` | String | `nomor` | Nomor TTK yang sama dengan header |
| `ATRBTDT` | Date | `tanggal` | Mengikuti tanggal header |
| `ATRBTP16` | Integer | `koli` | Nomor urut koli (Koli ke-1, 2, dst) |
| `ATRBTQTY` | Decimal | `grossWeight` | Berat Kotor Koli (KG) |
| `ATRBTP12` | String | `volumetrik` | Dimensi Volumetrik Koli (P x L x T) |
| `ATRBTP22` | Decimal | `kubikasi` | Volume Kubikasi Koli (M3) |
| `ATRBTP04` | Tinyint | `isBubble` | Checkbox Bubble Wrap (1=Ya, 0=Tidak) |
| `ATRBTP19` | Tinyint | `isKayu` | Checkbox Packing Kayu (1=Ya, 0=Tidak) |
| `ATRBTP13` | String | `jenisBarang` | Nama / Jenis Barang Spesifik Koli |
| `ATRBTCMT` | String | `catatan` | Catatan Khusus Koli |
| `ATRBTP20` | Decimal | `beratSurcharge` | Berat Kena Surcharge Koli (KG) |
| `ATRBTP21` | Decimal | `biayaSurcharge` | Nominal Biaya Surcharge Koli |
| `ATRBTP23` | Decimal | `biayaPacking` | Nominal Biaya Packing Kayu Koli |
| `ATRBTP24` | Decimal | `biayaPackingBubble`| Nominal Biaya Packing Bubble Wrap Koli |
| `ATRBTSS` | Integer | (Default 1) | Status Transaksi (1 = Baru) |
| `ATRBTBS` | Integer | (Default 1) | Status Bisnis |
| `ATRBTDTP` | DateTime | (NOW()) | Tanggal Posting |

---

## 2. API Contract — Wajib Disepakati Dulu

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
    "nomor": "15656747607"
  }
}
```

---

### 2.2 List TTK

```
GET /a/ttkcc/listTTKCC
Authorization: Bearer <token>
```

**Query Parameters:**
- `tanggalDari` (Date, optional)
- `tanggalSampai` (Date, optional)
- `accountId` (Integer, optional)
- `status` (Integer, optional)
- `d-limit` (Integer, default 25)
- `d-offset` (Integer, default 0)

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
        "nomor": "15656747607",
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

---

## 3. Pembagian Kerja

```
[TAHAP 1: ALIGNMENT]
Sepakati API Contract & Konfirmasi Pemetaan Kolom ENTBT.

         |------------------------------------|
         |                                    |
         v                                    v
  [Dev A - Frontend]                  [Dev B - Backend]
  1. Tombol & Validasi Save           1. Tambah Query 253006 & 253007
  2. buildPayload & handleSave        2. Registrasi EDBQP & Flush Redis
  3. Halaman PVwTTKCCLs.vue           3. Controller TTKCC.pm (entbtCreate)
  4. Routing routes.js                4. OpenAPI ttkcc.yaml & dcaf.yaml
         |                                    |
         |------------------------------------|
         v
  [TAHAP 2: INTEGRATION TEST]
  Colokkan Frontend ke Backend API lokal dev.
         v
  [TAHAP 3: MERGE & DEPLOY]
```

---

## 4. Dev A — Frontend

### 4.1 Modifikasi `TTKCC2.vue`

Tambahkan state, computed, method, dan tombol Save di toolbar atas:

```javascript
// data()
isSaving: false,
savedNomor: '',

// computed:
isReadyToSave () {
  return this.isDataLengkap && this.isPricingComplete && !this.isSaving
},
```

```javascript
// methods:
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
}
```

---

### 4.2 Buat File Baru `PVwTTKCCLs.vue`
**Path:** `web/dev/dcaf/src/pages/PVwTTKCCLs.vue`
*(Nama disesuaikan dengan konvensi standar `04-frontend-rules.md`: `PVw<Feature>Ls.vue`).*

File ini memuat tabel `q-table` riwayat TTK, form filter rentang tanggal, pencarian account, dan tombol pintas navigasi ke detail transaksi atau pembuatan transaksi baru.

---

### 4.3 Registrasi Route di `web/dev/dcaf/src/router/routes.js`

```javascript
// TTK Corporate Kubikasi
{ name: 'TTKCC2', path: '/TTKCC2', component: () => import('pages/TTKCC2.vue') },
{ name: 'PVwTTKCCLs', path: '/PVwTTKCCLs', component: () => import('pages/PVwTTKCCLs.vue') },
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

            # 1. Validasi field wajib di trust boundary
            my @missing;
            push @missing, 'accountId'    unless $rhcp->{accountId};
            push @missing, 'layananId'    unless $rhcp->{layananId};
            push @missing, 'asalId'       unless $rhcp->{asalId};
            push @missing, 'tujuanId'     unless $rhcp->{tujuanId};
            push @missing, 'namaPenerima' unless $rhcp->{namaPenerima};
            push @missing, 'alamat'       unless $rhcp->{alamat};
            push @missing, 'grossWeight'  unless $rhcp->{grossWeight};
            push @missing, 'koli'         unless $rhcp->{koli};

            if (@missing) {
                $ros->{q}->{e}->{status} = 400;
                $ros->{q}->{e}->{num}    = "$ros->{className}/createTTKCC/01";
                $ros->{q}->{e}->{msg}    = 'Field wajib belum lengkap: ' . join(', ', @missing);
                die;
            }

            # 2. INSERT Header TTK menggunakan Generic Query entbtCreate (253005)
            my $ownerId = $ros->{q}->{session}->{user}->{ownerId} || 3002;
            my $userId  = $ros->{q}->{session}->{user}->{accountId} || 1;

            my $roqHeader = DCAF::Query->new(
                q    => $ros->{q},
                name => 'entbtCreate',
                data => {
                    ATRBAIDO           => $ownerId,
                    ATRBAIDM           => $userId,
                    ATRBAIDP           => $rhcp->{accountId},
                    ATRBTTYID          => 5001,
                    ATRBIID            => $rhcp->{layananId},
                    ATRBTNO            => '[AUTO]',
                    ATRBTNOG           => $rhcp->{referensi} || '',
                    ATRBTDT            => $rhcp->{tanggal},
                    ATRBTSS            => 1,
                    ATRBTBS            => 1,
                    ATRBTP01           => $rhcp->{asalId},
                    ATRBTP02           => $rhcp->{tujuanId},
                    ATRBTP06           => $rhcp->{namaPenerima},
                    ATRBTP07           => $rhcp->{telepon} || '',
                    ATRBTP08           => $rhcp->{alamat},
                    ATRBTP09           => $rhcp->{kodeOps} || '',
                    ATRBTP11           => $rhcp->{grossWeight},
                    ATRBTQTY           => $rhcp->{chargedWeight} || $rhcp->{grossWeight},
                    ATRBTP12           => $rhcp->{volumetrik} || '',
                    ATRBTP13           => $rhcp->{barang} || '',
                    ATRBTP14           => $rhcp->{hargaBarang} || 0,
                    ATRBTP15           => $rhcp->{biayaAsuransi} || 0,
                    ATRBTP16           => $rhcp->{koli},
                    ATRBTP17           => $rhcp->{maxtm} || 0,
                    ATRBTP18           => $rhcp->{printer} || 102,
                    ATRBTP19           => $rhcp->{biayaTambahan} || 100,
                    ATRBTP21           => $rhcp->{adaPacking} || 2,
                    ATRBTP22           => $rhcp->{kubikasi} || 0,
                    ATRBTP23           => $rhcp->{packingBubble} || 0,
                    ATRBTP24           => $rhcp->{buktiKembali} || 100,
                    ATRBTP25           => $rhcp->{biayaDiskon} || 0,
                    ATRBTP26           => $rhcp->{biayaRealokasi} || 0,
                    ATRBTP27           => $rhcp->{biayaPickup} || 0,
                    ATRBTP28           => $rhcp->{biayaPacking} || 0,
                    ATRBTP29           => $rhcp->{biayaSurcharge} || 0,
                    ATRBTP30           => $rhcp->{biayaKirimPerKg} || 0,
                    ATRBTVALU          => $rhcp->{biayaKirim} || 0,
                    ATRBTVAL           => $rhcp->{total} || 0,
                    ATRBTP05           => $rhcp->{catatan} || '',
                    ATRBTDTP           => 'NOW()',
                }
            );
            $roqHeader->run || die;

            my $newBtid = $roqHeader->{result}->{insertId};
            unless ($newBtid) {
                $ros->{q}->{e}->{status} = 500;
                $ros->{q}->{e}->{num}    = "$ros->{className}/createTTKCC/02";
                $ros->{q}->{e}->{msg}    = 'Gagal menyimpan header TTK ke database';
                die;
            }

            # 3. Ambil nomor TTK yang terbentuk via query ttkccGetNomor
            my $rhresNomor = {};
            $ros->q2j('ttkccGetNomor', { btid => $newBtid }, { hashed => 1 }, $rhresNomor);
            my $rhttk = shift @{$rhresNomor->{result}->{body}};

            my $nomorTTK = ($rhttk && $rhttk->{nomor}) ? $rhttk->{nomor} : $newBtid;

            # 4. INSERT Child Koli menggunakan Generic Query entbtCreate
            my $listKoli = $rhcp->{listKoli} || [];
            foreach my $rhkoli (@{$listKoli}) {
                next unless $rhkoli->{koli} && $rhkoli->{grossWeight};
                my $roqKoli = DCAF::Query->new(
                    q    => $ros->{q},
                    name => 'entbtCreate',
                    data => {
                        ATRBAIDO           => $ownerId,
                        ATRBAIDM           => $userId,
                        ATRBAIDP           => $rhcp->{accountId},
                        ATRBTTYID          => 5001,
                        ATRBTIDP           => $newBtid,
                        ATRBTNO            => $nomorTTK,
                        ATRBTDT            => $rhcp->{tanggal},
                        ATRBTSS            => 1,
                        ATRBTBS            => 1,
                        ATRBTP16           => $rhkoli->{koli} || 1,
                        ATRBTQTY           => $rhkoli->{grossWeight} || 0,
                        ATRBTP12           => $rhkoli->{volumetrik} || '',
                        ATRBTP22           => $rhkoli->{kubikasi} || 0,
                        ATRBTP04           => $rhkoli->{isBubble} ? 1 : 0,
                        ATRBTP19           => $rhkoli->{isKayu} ? 1 : 0,
                        ATRBTP13           => $rhkoli->{jenisBarang} || '',
                        ATRBTCMT           => $rhkoli->{catatan} || '',
                        ATRBTP20           => $rhkoli->{beratSurcharge} || 0,
                        ATRBTP21           => $rhkoli->{biayaSurcharge} || 0,
                        ATRBTP23           => $rhkoli->{biayaPacking} || 0,
                        ATRBTP24           => $rhkoli->{biayaPackingBubble} || 0,
                        ATRBTDTP           => 'NOW()',
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

### 5.2 Definisi SQL Query EDBQ & EDBQP (Aturan 253)

Hanya 2 query SELECT baru yang perlu didaftarkan ke `lib/db/git/dcaf-data.sql` dan dieksekusi di database dev:

```sql
-- ============================================================
-- Modul TTKCC2 Queries (Tabel EDBQ & EDBQP - Sesuai Range 253)
-- ============================================================

-- 1. Query ttkccGetNomor (ID 253006)
INSERT INTO `EDBQ` (`ADBQID`, `ADBCID`, `ADBQCD`, `ADBQNM`, `ADBQSCR`, `ADBQPAR`, `ADBQCTM`, `ADBQCMT`, `ADBQDTP`)
VALUES (
  253006,
  3,
  'ttkccGetNomor',
  'TTKCC: Ambil ATRBTNO / ATRBTNOG by ID',
  'SELECT
  ATRBTID             btid,
  COALESCE(ATRBTNO, ATRBTNOG, ATRBTID) nomor
FROM
  ta15_wlogisticsidb_ap.ENTBT
WHERE
  ATRBTID = ?',
  'btid',
  0,
  'Dipakai setelah insert entbtCreate untuk mengambil nomor TTK hasil trigger',
  NOW()
);

INSERT INTO `EDBQP` (`ADBQPID`, `ADBQID`, `ADBQPCD`, `ADBQPNM`, `ADBQPDV`)
VALUES
  (253006001, 253006, 'btid', 'Business Transaction ID', NULL);


-- 2. Query ttkccListHeader (ID 253007)
INSERT INTO `EDBQ` (`ADBQID`, `ADBCID`, `ADBQCD`, `ADBQNM`, `ADBQSCR`, `ADBQPAR`, `ADBQCTM`, `ADBQCMT`, `ADBQDTP`)
VALUES (
  253007,
  3,
  'ttkccListHeader',
  'TTKCC: List Header TTK Corporate Kubikasi',
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
  t.ATRBTDT DESC, t.ATRBTID DESC',
  'tanggalDari,tanggalDari,tanggalSampai,tanggalSampai,accountId,accountId,status,status',
  0,
  'Daftar header transaksi TTK Corporate untuk PVwTTKCCLs.vue',
  NOW()
);

INSERT INTO `EDBQP` (`ADBQPID`, `ADBQID`, `ADBQPCD`, `ADBQPNM`, `ADBQPDV`)
VALUES
  (253007001, 253007, 'tanggalDari', 'Filter Tanggal Dari', ''),
  (253007002, 253007, 'tanggalSampai', 'Filter Tanggal Sampai', ''),
  (253007003, 253007, 'accountId', 'Filter Partner Account ID', '0'),
  (253007004, 253007, 'status', 'Filter Status Sistem', '0');
```

> **Wajib Dilakukan Langsung (Redis Cache Busting):**
> Jalankan perintah ini di container backend tepat setelah INSERT SQL berhasil:
> ```bash
> redis-cli keys "drp.qc*ttkccGetNomor*" | xargs redis-cli del
> redis-cli keys "drp.qc*ttkccListHeader*" | xargs redis-cli del
> ```

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
```

---

### 5.4 Registrasi Endpoint di `etc/dcaf.yaml`

```yaml
api:
   dcaf: /etc/api/dcaf.yaml
   3pl: /etc/api/3pl.yaml
   ttkcc: /etc/api/ttkcc.yaml    # Registrasi controller TTKCC
```

---

## 6. Urutan Eksekusi & Checkpoint

### Phase 1 — Setup & Review
- [ ] Review bersama dokumen ini dan konfirmasi pemanfaatan query `entbtCreate` (253005).
- [ ] Buat branch git masing-masing di repositori `swb`:
  - Frontend: `feature/ttkcc2-save-frontend`
  - Backend: `feature/ttkcc2-save-backend`

### Phase 2 — Parallel Development
**Checklist Dev A (Frontend):**
- [ ] Modifikasi `TTKCC2.vue` (tambah state `isSaving`, computed `isReadyToSave`, tombol Save).
- [ ] Implementasi fungsi `handleSave` dan `buildPayload` di `TTKCC2.vue`.
- [ ] Buat halaman baru `PVwTTKCCLs.vue`.
- [ ] Daftarkan route di `web/dev/dcaf/src/router/routes.js`.
- [ ] Uji visual form & tombol di browser (Dev Server Quasar).

**Checklist Dev B (Backend):**
- [ ] Eksekusi query `253006` dan `253007` (EDBQ & EDBQP) di database dev.
- [ ] Langsung flush Redis cache untuk kedua query tersebut.
- [ ] Buat class controller `lib/perl/DCAF/Controller/TTKCC.pm`.
- [ ] Jalankan uji sintaks: `perl -c -Ilib/perl lib/perl/DCAF/Controller/TTKCC.pm`.
- [ ] Buat file OpenAPI `etc/api/ttkcc.yaml`.
- [ ] Daftarkan modul di `etc/dcaf.yaml`.
- [ ] Restart service uWSGI dev dan uji endpoint via Postman / curl.

### Phase 3 — Integration Testing
- [ ] Isi form lengkap di `TTKCC2.vue` lalu klik Save.
- [ ] Verifikasi dialog konfirmasi muncul: "Are you sure to CREATE new record?".
- [ ] Verifikasi data tersimpan di tabel `ENTBT` (header dan child koli).
- [ ] Verifikasi notifikasi nomor TTK tampil di UI form.
- [ ] Buka halaman `PVwTTKCCLs.vue` dan pastikan record TTK baru muncul pada tabel.
- [ ] Cek filter tanggal dan filter account pada `PVwTTKCCLs.vue`.

---

## 7. Hal Kritis & Catatan Verifikasi

1. **Penggunaan Generic Query `entbtCreate` (ID `253005`):**
   Tidak perlu membuat query INSERT custom baru. `entbtCreate` sudah dirancang khusus untuk menangani tabel `ENTBT` pada koneksi 3 (`ta15_wlogisticsidb_ap`).
2. **Kepatuhan Aturan 253 & EDBQP:**
   Semua query baru menggunakan prefix ID `253`, dan seluruh parameternya wajib terdaftar di tabel `EDBQP`.
3. **Disiplin Redis Cache:**
   Setiap ada penambahan atau pengubahan query di `EDBQ`/`EDBQP`, cache Redis **wajib langsung di-flush sendiri** oleh developer/agent.
4. **Disiplin Gaya Kode:**
   Tidak ada inline SQL di Perl, dan tidak ada emoticon / emoji di semua baris file maupun komentar.

---
*Dokumen diperbarui: 2026-11-07 — Telah diselaraskan dengan aturan terbaru .agents/rules dan .ai.*
