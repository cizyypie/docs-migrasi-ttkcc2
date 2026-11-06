# Rencana Implementasi: Fitur Save + List TTKCC2

> [!IMPORTANT]
> Dokumen ini adalah panduan eksekusi dua developer yang bekerja paralel.
> Baca seluruh dokumen sebelum mulai coding. Sepakati API Contract di Bagian 2 terlebih dahulu.

---

## Daftar Isi

1. [Gambaran Besar](#1-gambaran-besar)
2. [API Contract — Wajib Disepakati Dulu](#2-api-contract--wajib-disepakati-dulu)
3. [Pembagian Kerja](#3-pembagian-kerja)
4. [Dev A — Frontend](#4-dev-a--frontend)
5. [Dev B — Backend](#5-dev-b--backend)
6. [Urutan Eksekusi & Checkpoint](#6-urutan-eksekusi--checkpoint)
7. [Hal Kritis yang Harus Dikonfirmasi](#7-hal-kritis-yang-harus-dikonfirmasi)

---

## 1. Gambaran Besar

### Fitur yang Akan Dibuat

| Fitur | Keterangan |
|-------|------------|
| Tombol Save di TTKCC2 | Tombol hijau di kanan atas, mirip sandbox |
| Validasi form sebelum save | Cek field wajib, cek tarif sudah dihitung |
| Konfirmasi dialog | "Are you sure to CREATE new record?" |
| Loading overlay saat proses | "Creating data..." |
| Notifikasi hasil | Sukses / gagal |
| Reset form setelah sukses | Form bersih + nomor TTK tampil |
| Halaman List TTK | Tabel daftar TTK yang sudah disimpan |
| Filter list | Tanggal, Account, Status |
| Detail TTK dari list | Klik baris → buka form dalam mode view |

### File yang Akan Dibuat / Dimodifikasi

| File | Pemilik | Status |
|------|---------|--------|
| `web/dev/dcaf/src/pages/TTKCC2.vue` | Dev A | Dimodifikasi |
| `web/dev/dcaf/src/pages/TTKCCList.vue` | Dev A | **File baru** |
| `web/dev/dcaf/src/router/routes.js` | Dev A | Dimodifikasi (1 baris) |
| `lib/perl/DCAF/Controller/TTKCC.pm` | Dev B | **File baru** |
| `etc/api/ttkcc.yaml` | Dev B | **File baru** |
| `etc/dcaf.yaml` | Dev B | Dimodifikasi (1 baris) |
| `lib/db/git/dcaf-data.sql` | Dev B | Dimodifikasi (tambah queries) |

> [!NOTE]
> Zero overlap antar file. Satu-satunya potensi konflik adalah `etc/dcaf.yaml` dan `routes.js` — keduanya hanya 1 baris tambahan, lakukan di akhir setelah merge.

---

## 2. API Contract — Wajib Disepakati Dulu

> [!WARNING]
> Ini adalah "jembatan" antara FE dan BE. Kedua dev harus setuju dengan shape ini sebelum split kerja. Kalau ada perubahan, harus dikomunikasikan ke keduanya.

### 2.1 Create TTK

```
POST /a/ttkcc/createTTKCC
Content-Type: application/json
Authorization: Bearer <token>
```

**Request Body:**
```json
{
  "tanggal":        "2026-11-06",
  "accountId":      188889202,
  "layananId":      7,
  "referensi":      "REF-123",
  "adaPacking":     "2",
  "printer":        3,
  "buktiKembali":   [2, 3],
  "asalId":         3216,
  "tujuanId":       3171,
  "grossWeight":    26,
  "chargedWeight":  26,
  "koli":           1,
  "volumetrik":     "0 x 0 x 0",
  "kubikasi":       1,
  "barang":         "TESTING",
  "hargaBarang":    0,
  "biayaTambahan":  [1],
  "namaPenerima":   "TESTING",
  "telepon":        "",
  "alamat":         "TESTING",
  "supplierId":     null,
  "kodeOps":        "077-159-786-150-700-169-626",
  "maxtm":          2,
  "biayaKirimPerKg": 1700,
  "biayaKirim":     44200,
  "biayaAsuransi":  0,
  "biayaRealokasi": 0,
  "biayaPickup":    0,
  "biayaPacking":   0,
  "packingBubble":  "",
  "biayaSurcharge": 0,
  "biayaDiskon":    0,
  "total":          44200,
  "catatan":        "",
  "listKoli": [
    {
      "koli":               1,
      "grossWeight":        26,
      "volumetrik":         "0 x 0 x 0",
      "kubikasi":           1,
      "isBubble":           false,
      "isKayu":             false,
      "jenisBarang":        "TESTING",
      "catatan":            "",
      "beratSurcharge":     0,
      "biayaSurcharge":     0,
      "biayaPacking":       0,
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

**Response Gagal (HTTP 400/500):**
```json
{
  "code": 1,
  "status": "failed",
  "message": "Missing required fields: accountId, asalId"
}
```

---

### 2.2 List TTK

```
GET /a/ttkcc/listTTKCC
Authorization: Bearer <token>
```

**Query Parameters:**

| Param | Tipe | Keterangan |
|-------|------|------------|
| `tanggalDari` | string (YYYY-MM-DD) | Filter tanggal mulai |
| `tanggalSampai` | string (YYYY-MM-DD) | Filter tanggal akhir |
| `accountId` | integer | Filter account |
| `status` | integer | Filter status (1=baru, dll) |
| `d-limit` | integer | Jumlah baris (default 25) |
| `d-offset` | integer | Offset paginasi |

**Response Sukses:**
```json
{
  "code": 0,
  "status": "success",
  "message": "Successful",
  "data": {
    "body": [
      {
        "btid":    123456789,
        "nomor":   "50180000ENTBT-20261106-001",
        "tanggal": "2026-11-06",
        "account": "2100081011: Astra Intern",
        "asal":    "Jawa Barat, Kab. Bekasi",
        "tujuan":  "DKI Jakarta",
        "total":   44200,
        "status":  1,
        "statusLabel": "Baru",
        "koli":    1,
        "berat":   26
      }
    ],
    "rows": 42
  }
}
```

---

### 2.3 Get Detail TTK

```
GET /a/ttkcc/getTTKCC/{btid}
Authorization: Bearer <token>
```

**Response Sukses:**
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
      "account": "2100081011: Astra Intern",
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
      "namaPenerima": "TESTING",
      "alamat": "TESTING",
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

### Ringkasan

```
[BERSAMA - 1 jam]
Baca dokumen ini, sepakati API Contract, pastikan tidak ada ambiguitas.

         ┌────────────────────────────────┐
         ▼                                ▼
  [Dev A - Frontend]              [Dev B - Backend]
  TTKCC2.vue + TTKCCList.vue      TTKCC.pm + SQL + YAML
         │                                │
         ▼                                ▼
  [CHECKPOINT - Integration Test]
  Colokkan FE ke BE, test end-to-end
         │
         ▼
  [BERSAMA - Fix bugs, polish]
```

### Aturan Kolaborasi

- Semua komunikasi perubahan API Contract lewat dokumen ini (update Bagian 2)
- Dev A tidak menyentuh file `lib/`, `etc/`, dan `lib/db/`
- Dev B tidak menyentuh file di `web/dev/dcaf/src/pages/`
- Kalau butuh query baru untuk testing FE → Dev A pakai mock data dulu
- Merge ke branch utama hanya setelah integration test lulus

---

## 4. Dev A — Frontend

### 4.1 Modifikasi `TTKCC2.vue`

#### A1. Tambah State Baru di `data()`

Tambahkan di dalam blok `data()`, setelah baris `loadingDropdowns: false`:

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

> [!NOTE]
> `isDataLengkap` dan `isPricingComplete` sudah ada di computed yang sudah ada. Tinggal gabungkan.

#### A3. Tambah Tombol Save di Template

Tombol ini ditaruh di row atas, sejajar dengan q-tabs. Cari bagian:
```html
<!-- 1. TAB SELECTOR CONTROL & TOOLBAR (Gambar 4) -->
<div class="row items-center justify-between q-mb-sm">
```

Tambahkan tombol di dalam `<div class="row items-center justify-between q-mb-sm">` sebelum penutup `</div>`:

```html
<!-- Toolbar kanan: Tombol Save -->
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

Tambahkan di blok `methods:`:

```javascript
async handleSave () {
  // Guard: validasi ulang
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
      message: 'Lengkapi data Layanan, Asal, Tujuan, dan Berat agar tarif bisa dihitung sebelum menyimpan.',
      ok: 'OK'
    })
    return
  }

  // Konfirmasi (sama seperti sandbox)
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

  // Loading
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
    adaPacking:      f.adaPacking,
    printer:         f.printer,
    buktiKembali:    f.buktiKembali || [],
    asalId:          getVal(f.asal),
    tujuanId:        getVal(f.tujuan),
    grossWeight:     parseFloat(f.grossWeight) || 0,
    chargedWeight:   parseFloat(f.chargedWeight) || 0,
    koli:            parseInt(f.koli) || 0,
    volumetrik:      f.volumetrik || '',
    kubikasi:        parseFloat(f.kubikasi) || 0,
    barang:          f.barang || '',
    hargaBarang:     parseFloat(f.hargaBarang) || 0,
    biayaTambahan:   f.biayaTambahan || [],
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
    packingBubble:   f.packingBubble || '',
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
},
```

---

### 4.2 Buat File Baru `TTKCCList.vue`

**Path:** `web/dev/dcaf/src/pages/TTKCCList.vue`

```vue
<template>
  <q-page class="q-pa-md">
    <!-- Header -->
    <div class="row items-center justify-between q-mb-md">
      <div class="text-h6 text-weight-bold">TTK Corporate Kubikasi — Daftar</div>
      <q-btn
        color="positive"
        label="Buat TTK Baru"
        icon="add"
        unelevated
        :to="{ name: 'TTKCC2' }"
      />
    </div>

    <!-- Filter -->
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
          <div class="text-caption text-weight-bold q-mb-xs">Account</div>
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
            placeholder="Semua account..."
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

    <!-- Tabel List -->
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
      <!-- Kolom Nomor — klik untuk detail -->
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

      <!-- Kolom Total — format currency -->
      <template v-slot:body-cell-total="props">
        <q-td :props="props" class="text-right">
          Rp {{ formatNumber(props.value) }}
        </q-td>
      </template>

      <!-- Kolom Status -->
      <template v-slot:body-cell-statusLabel="props">
        <q-td :props="props">
          <q-badge :color="statusColor(props.row.status)" :label="props.value" />
        </q-td>
      </template>

      <!-- No data -->
      <template v-slot:no-data>
        <div class="full-width text-center q-pa-lg text-grey-6">
          <q-icon name="inbox" size="48px" /><br>
          Belum ada data TTK ditemukan
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
        { name: 'total',       label: 'Total',       field: 'total',       align: 'right',  sortable: true },
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
          this.$q.notify({ type: 'negative', message: res.data.message || 'Gagal memuat data' })
        }
      } catch (err) {
        console.error('loadList error:', err)
        this.$q.notify({ type: 'negative', message: 'Gagal memuat data list TTK' })
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
      // Buka halaman form TTKCC2 dengan btid (mode view/edit — fase berikutnya)
      // Untuk sekarang bisa navigate ke list dengan query param
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

### 4.3 Tambah Route di `routes.js`

Cari baris:
```javascript
{ name: 'TTKCC2', path: '/TTKCC2', component: () => import('pages/TTKCC2.vue') },
```

Tambahkan setelahnya:
```javascript
{ name: 'TTKCCList', path: '/TTKCCList', component: () => import('pages/TTKCCList.vue') },
```

---

## 5. Dev B — Backend

### 5.1 Buat File `lib/perl/DCAF/Controller/TTKCC.pm`

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
        # createTTKCC — POST /a/ttkcc/createTTKCC
        # ----------------------------------------------------------------
        $ros->{api}->{operation} eq 'createTTKCC' && do {

            # Validasi field wajib
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

            # INSERT header TTK ke ENTBT
            my $roqHeader = DCAF::Query->new(
                q    => $ros->{q},
                name => 'ttkccCreateHeader',
                data => {
                    tanggal      => $rhcp->{tanggal},
                    accountId    => $rhcp->{accountId},
                    layananId    => $rhcp->{layananId},
                    asalId       => $rhcp->{asalId},
                    tujuanId     => $rhcp->{tujuanId},
                    grossWeight  => $rhcp->{grossWeight},
                    chargedWeight => $rhcp->{chargedWeight} || $rhcp->{grossWeight},
                    koli         => $rhcp->{koli},
                    volumetrik   => $rhcp->{volumetrik} || '',
                    kubikasi     => $rhcp->{kubikasi} || 0,
                    barang       => $rhcp->{barang} || '',
                    hargaBarang  => $rhcp->{hargaBarang} || 0,
                    namaPenerima => $rhcp->{namaPenerima},
                    telepon      => $rhcp->{telepon} || '',
                    alamat       => $rhcp->{alamat},
                    kodeOps      => $rhcp->{kodeOps} || '',
                    maxtm        => $rhcp->{maxtm} || 0,
                    biayaKirimPerKg => $rhcp->{biayaKirimPerKg} || 0,
                    biayaKirim   => $rhcp->{biayaKirim} || 0,
                    biayaAsuransi => $rhcp->{biayaAsuransi} || 0,
                    biayaRealokasi => $rhcp->{biayaRealokasi} || 0,
                    biayaPickup  => $rhcp->{biayaPickup} || 0,
                    biayaPacking => $rhcp->{biayaPacking} || 0,
                    biayaSurcharge => $rhcp->{biayaSurcharge} || 0,
                    biayaDiskon  => $rhcp->{biayaDiskon} || 0,
                    total        => $rhcp->{total} || 0,
                    catatan      => $rhcp->{catatan} || '',
                    supplierId   => $rhcp->{supplierId} || 0,
                    adaPacking   => $rhcp->{adaPacking} || 2,
                    printer      => $rhcp->{printer} || 0,
                    referensi    => $rhcp->{referensi} || '',
                }
            );
            $roqHeader->run || die;

            unless ($roqHeader->{result}->{rows}) {
                $ros->{q}->{e}->{status} = 500;
                $ros->{q}->{e}->{num}    = "$ros->{className}/createTTKCC/02";
                $ros->{q}->{e}->{msg}    = 'Gagal menyimpan header TTK';
                die;
            }

            my $newBtid = $roqHeader->{result}->{insertId};

            # Ambil nomor TTK yang di-generate
            my $rhresNomor = {};
            $ros->q2j('ttkccGetNomor', { btid => $newBtid }, { hashed => 1 }, $rhresNomor);
            my $rhttk = shift @{$rhresNomor->{result}->{body}};

            unless ($rhttk) {
                $ros->{q}->{e}->{status} = 500;
                $ros->{q}->{e}->{num}    = "$ros->{className}/createTTKCC/03";
                $ros->{q}->{e}->{msg}    = 'Gagal mengambil nomor TTK';
                die;
            }

            my $nomorTTK = $rhttk->{nomor};

            # INSERT koli detail (loop)
            my $listKoli = $rhcp->{listKoli} || [];
            foreach my $rhkoli (@{$listKoli}) {
                next unless $rhkoli->{koli} && $rhkoli->{grossWeight};
                my $roqKoli = DCAF::Query->new(
                    q    => $ros->{q},
                    name => 'ttkccCreateKoli',
                    data => {
                        parentBtid       => $newBtid,
                        nomor            => $nomorTTK,
                        koli             => $rhkoli->{koli} || 1,
                        grossWeight      => $rhkoli->{grossWeight} || 0,
                        volumetrik       => $rhkoli->{volumetrik} || '',
                        kubikasi         => $rhkoli->{kubikasi} || 0,
                        isBubble         => $rhkoli->{isBubble} ? 1 : 0,
                        isKayu           => $rhkoli->{isKayu} ? 1 : 0,
                        jenisBarang      => $rhkoli->{jenisBarang} || '',
                        catatan          => $rhkoli->{catatan} || '',
                        beratSurcharge   => $rhkoli->{beratSurcharge} || 0,
                        biayaSurcharge   => $rhkoli->{biayaSurcharge} || 0,
                        biayaPacking     => $rhkoli->{biayaPacking} || 0,
                        biayaPackingBubble => $rhkoli->{biayaPackingBubble} || 0,
                    }
                );
                $roqKoli->run;
                # Koli insert failure bukan fatal, lanjutkan
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
        # listTTKCC — GET /a/ttkcc/listTTKCC
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
                code     => 0,
                status   => 'success',
                message  => 'Successful',
                data     => $rhres->{result}
            });
            last switch;
        };

        # ----------------------------------------------------------------
        # getTTKCC — GET /a/ttkcc/getTTKCC/{btid}
        # ----------------------------------------------------------------
        $ros->{api}->{operation} eq 'getTTKCC' && do {
            my $btid = $rhpp->{btid};
            unless ($btid) {
                $ros->{q}->{e}->{status} = 400;
                $ros->{q}->{e}->{num}    = "$ros->{className}/getTTKCC/01";
                $ros->{q}->{e}->{msg}    = 'Missing btid';
                die;
            }

            # Ambil header
            my $rhresHeader = {};
            $ros->q2j('ttkccGetHeader', { btid => $btid }, { hashed => 1 }, $rhresHeader);
            my $rhheader = shift @{$rhresHeader->{result}->{body}};
            unless ($rhheader) {
                $ros->{q}->{e}->{status} = 404;
                $ros->{q}->{e}->{num}    = "$ros->{className}/getTTKCC/02";
                $ros->{q}->{e}->{msg}    = 'TTK tidak ditemukan';
                die;
            }

            # Ambil koli
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

        # Default — operasi tidak dikenal
        $ros->{templateData}->{jstext} = $DCAF::ros->{json}->encode({
            code    => 1,
            status  => 'failed',
            message => 'Unhandled request',
            data    => {}
        });
    };

    $ros->log('SO');
    $ros->render || die;
    return 1;
}

1;
```

> [!NOTE]
> Setelah membuat file ini, wajib run syntax check:
> ```bash
> perl -c -Ilib/perl lib/perl/DCAF/Controller/TTKCC.pm
> ```

---

### 5.2 Tambah SQL Queries ke `lib/db/git/dcaf-data.sql`

Tambahkan baris berikut di akhir blok INSERT `EDBQ`, **setelah** baris terakhir yang ada (baris 17847 saat ini).

> [!IMPORTANT]
> Gunakan ID EDBQ yang belum terpakai. Cek dulu ID terakhir dengan:
> ```bash
> grep "INSERT INTO \`EDBQ\`" lib/db/git/dcaf-data.sql | tail -5
> ```
> Gunakan ID berikutnya secara berurutan. Contoh di bawah menggunakan ID 7001-7005 — sesuaikan jika sudah ada yang pakai.

```sql
-- ============================================================
-- TTKCC2 Queries
-- ============================================================

-- 7001: Create header TTK Corporate Kubikasi
INSERT INTO `EDBQ` VALUES (7001,3,'ttkccCreateHeader','TTKCC: Create header record',
'INSERT INTO ta15_wlogisticsidb_ap.ENTBT
SET
  ATRBTTYID = 5001,
  ATRBTDT   = ?,
  ATRBAIDP  = ?,
  ATRBIID   = ?,
  ATRBTP01  = ?,
  ATRBTP02  = ?,
  ATRBTSS   = 1,
  ATRBTP11  = ?,
  ATRBTQTY  = ?,
  ATRBTP16  = ?,
  ATRBTP12  = ?,
  ATRBTP13  = ?,
  ATRBTP14  = ?,
  ATRBTP03  = ?,
  ATRBTP06  = ?,
  ATRBTP07  = ?,
  ATRBTP08  = ?,
  ATRBTP09  = ?,
  ATRBTP17  = ?,
  ATRBTP30  = ?,
  ATRBTVALU = ?,
  ATRBTP15  = ?,
  ATRBTP27  = ?,
  ATRBTP28  = ?,
  ATRBTP26  = ?,
  ATRBTP29  = ?,
  ATRBTVAL  = ?,
  ATRBTP05  = ?,
  ATRBTP20  = ?,
  ATRBIID2  = ?,
  ATRBTP23  = ?,
  ATRBTDTP  = now()
',
'tanggal,accountId,layananId,asalId,tujuanId,grossWeight,chargedWeight,koli,volumetrik,barang,hargaBarang,namaPenerima,namaPenerima,telepon,alamat,kodeOps,maxtm,biayaKirimPerKg,biayaKirim,biayaAsuransi,biayaRealokasi,biayaPickup,biayaPacking,biayaSurcharge,biayaDiskon,total,catatan,adaPacking,printer,referensi',
0,NULL,NULL,NOW());

-- 7002: Get nomor TTK setelah insert
INSERT INTO `EDBQ` VALUES (7002,3,'ttkccGetNomor','TTKCC: Get nomor by ATRBTID',
'SELECT
  ATRBTID   btid,
  ATRBTNO   nomor
FROM
  ta15_wlogisticsidb_ap.ENTBT
WHERE
  ATRBTID = ?
',
'btid',
0,NULL,NULL,NOW());

-- 7003: Create koli detail (child record)
INSERT INTO `EDBQ` VALUES (7003,3,'ttkccCreateKoli','TTKCC: Create koli detail',
'INSERT INTO ta15_wlogisticsidb_ap.ENTBT
SET
  ATRBTTYID = 5001,
  ATRBTIDP  = ?,
  ATRBTNO   = ?,
  ATRBTQTY  = ?,
  ATRBTP11  = ?,
  ATRBTP12  = ?,
  ATRBTP22  = ?,
  ATRBTP21  = ?,
  ATRBTP13  = ?,
  ATRBTP05  = ?,
  ATRBTP25  = ?,
  ATRBTP29  = ?,
  ATRBTP26  = ?,
  ATRBTP24  = ?
',
'parentBtid,nomor,koli,grossWeight,volumetrik,isBubble,isKayu,jenisBarang,catatan,beratSurcharge,biayaSurcharge,biayaPacking,biayaPackingBubble',
0,NULL,NULL,NOW());

-- 7004: List TTK header
INSERT INTO `EDBQ` VALUES (7004,3,'ttkccListHeader','TTKCC: List header records',
'SELECT
  t.ATRBTID          btid,
  t.ATRBTNO          nomor,
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
  AND t.ATRBTIDP = 0
  AND (? = '''' OR date(t.ATRBTDT) >= ?)
  AND (? = '''' OR date(t.ATRBTDT) <= ?)
  AND (? = 0 OR t.ATRBAIDP = ?)
  AND (? = 0 OR t.ATRBTSS = ?)
ORDER BY
  t.ATRBTDT DESC
',
'tanggalDari,tanggalDari,tanggalSampai,tanggalSampai,accountId,accountId,status,status',
0,NULL,NULL,NOW());

-- 7005: Get detail header TTK by ATRBTID
INSERT INTO `EDBQ` VALUES (7005,3,'ttkccGetHeader','TTKCC: Get header detail by btid',
'SELECT
  t.ATRBTID           btid,
  t.ATRBTNO           nomor,
  date(t.ATRBTDT)     tanggal,
  t.ATRBAIDP          accountId,
  concat(ba.ATRBACD, '': '', be.ATRBENM) account,
  t.ATRBIID           layananId,
  bi.ATRBINM          layanan,
  t.ATRBTP01          asalId,
  oa.ATRCAREANML      asal,
  t.ATRBTP02          tujuanId,
  da.ATRCAREANML      tujuan,
  t.ATRBTP11          grossWeight,
  t.ATRBTQTY          chargedWeight,
  t.ATRBTP16          koli,
  t.ATRBTP12          volumetrik,
  t.ATRBTP13          barang,
  t.ATRBTP14          hargaBarang,
  t.ATRBTP03          namaPenerima,
  t.ATRBTP07          telepon,
  t.ATRBTP08          alamat,
  t.ATRBTP09          kodeOps,
  t.ATRBTP17          maxtm,
  t.ATRBTP30          biayaKirimPerKg,
  t.ATRBTVALU         biayaKirim,
  t.ATRBTP15          biayaAsuransi,
  t.ATRBTVAL          total,
  t.ATRBTP05          catatan,
  t.ATRBTSS           status,
  lv.ATRLVNM          statusLabel
FROM
  ta15_wlogisticsidb_ap.ENTBT t
  LEFT JOIN swb.ENTBA ba ON ba.ATRBAID = t.ATRBAIDP
  LEFT JOIN swb.ENTBE be ON be.ATRBEID = ba.ATRBEID
  LEFT JOIN swb.ENTBI bi ON bi.ATRBIID = t.ATRBIID
  LEFT JOIN swb.ENTCAREA oa ON oa.ATRCAREAID = t.ATRBTP01
  LEFT JOIN swb.ENTCAREA da ON da.ATRCAREAID = t.ATRBTP02
  LEFT JOIN swb.ENTLV lv ON lv.ATRLVTYID = 9 AND lv.ATRLVCD = t.ATRBTSS
WHERE
  t.ATRBTID = ?
',
'btid',
0,NULL,NULL,NOW());

-- 7006: Get koli detail by parent ATRBTID
INSERT INTO `EDBQ` VALUES (7006,3,'ttkccGetKoli','TTKCC: Get koli detail by parentBtid',
'SELECT
  t.ATRBTID           btid,
  t.ATRBTQTY          koli,
  t.ATRBTP11          grossWeight,
  t.ATRBTP12          volumetrik,
  t.ATRBTP22          isBubble,
  t.ATRBTP21          isKayu,
  t.ATRBTP13          jenisBarang,
  t.ATRBTP05          catatan,
  t.ATRBTP25          beratSurcharge,
  t.ATRBTP29          biayaSurcharge,
  t.ATRBTP26          biayaPacking,
  t.ATRBTP24          biayaPackingBubble
FROM
  ta15_wlogisticsidb_ap.ENTBT t
WHERE
  t.ATRBTIDP = ?
ORDER BY
  t.ATRBTID ASC
',
'parentBtid',
0,NULL,NULL,NOW());
```

> [!WARNING]
> Query `ttkccListHeader` dan `ttkccGetHeader` menggunakan JOIN lintas database (`ta15_wlogisticsidb_ap` dan `swb`). Pastikan user DB `dwtb` memiliki akses SELECT ke `ta15_wlogisticsidb_ap`. Konfirmasi ke DBA/DevOps sebelum deploy.

---

### 5.3 Buat File `etc/api/ttkcc.yaml`

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
    header:
      type: apiKey
      in: header
      name: X-API-Key
    cookie:
      type: apiKey
      in: cookie
      name: dcaf
    query:
      type: apiKey
      in: query
      name: apiKey
paths:
  /createTTKCC:
    post:
      tags:
        - TTKCC
      summary: Create new TTK Corporate Kubikasi
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
              properties:
                tanggal:         { type: string, format: date }
                accountId:       { type: integer }
                layananId:       { type: integer }
                asalId:          { type: integer }
                tujuanId:        { type: integer }
                grossWeight:     { type: number }
                chargedWeight:   { type: number }
                koli:            { type: integer }
                namaPenerima:    { type: string }
                telepon:         { type: string }
                alamat:          { type: string }
                total:           { type: number }
                listKoli:
                  type: array
                  items:
                    type: object
                    properties:
                      koli:        { type: integer }
                      grossWeight: { type: number }
      responses:
        200:
          description: Successful
          content:
            application/json:
              schema:
                type: object
                properties:
                  code:    { type: integer }
                  status:  { type: string }
                  message: { type: string }
                  data:
                    type: object
                    properties:
                      btid:  { type: integer }
                      nomor: { type: string }

  /listTTKCC:
    get:
      tags:
        - TTKCC
      summary: List TTK Corporate Kubikasi
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
          description: Successful
          content:
            application/json:
              schema:
                type: object

  /getTTKCC/{btid}:
    get:
      tags:
        - TTKCC
      summary: Get detail TTK by ATRBTID
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
          description: Successful
          content:
            application/json:
              schema:
                type: object
```

---

### 5.4 Tambah Registrasi di `etc/dcaf.yaml`

Cari blok `api:` dan tambahkan satu baris:

```yaml
api:
   dcaf: /etc/api/dcaf.yaml
   3pl: /etc/api/3pl.yaml
   # ... entries yang sudah ada ...
   ttkcc: /etc/api/ttkcc.yaml    # <-- TAMBAHKAN INI
```

---

## 6. Urutan Eksekusi & Checkpoint

### Phase 1 — Setup (Bersama, ~30 menit)
- [ ] Baca dokumen ini bersama
- [ ] Sepakati API Contract di Bagian 2
- [ ] Konfirmasi akses DB `ta15_wlogisticsidb_ap` dari DCAF SWB (tanya DevOps)
- [ ] Konfirmasi mekanisme auto-generate nomor TTK (cek query `drpInsertChildBTMultiKoli` yang ada)
- [ ] Buat branch masing-masing: `feature/ttkcc2-frontend` dan `feature/ttkcc2-backend`

### Phase 2 — Parallel Development

**Dev A checklist:**
- [ ] A1: Tambah `isSaving`, `savedNomor` di `data()`
- [ ] A2: Tambah computed `isReadyToSave`
- [ ] A3: Tambah tombol Save di template
- [ ] A4: Tambah method `handleSave()` dan `buildPayload()`
- [ ] A5: Buat `TTKCCList.vue`
- [ ] A6: Tambah route `TTKCCList` di `routes.js`
- [ ] A7: Test UI secara visual (gunakan mock data jika BE belum siap)

**Dev B checklist:**
- [ ] B1: Tambah SQL queries ke `dcaf-data.sql` (IDs 7001-7006)
- [ ] B2: Jalankan SQL di database dev
- [ ] B3: Buat `lib/perl/DCAF/Controller/TTKCC.pm`
- [ ] B4: Run syntax check: `perl -c -Ilib/perl lib/perl/DCAF/Controller/TTKCC.pm`
- [ ] B5: Buat `etc/api/ttkcc.yaml`
- [ ] B6: Tambah `ttkcc` ke `etc/dcaf.yaml`
- [ ] B7: Restart server dev, test endpoint via curl/Postman

### Phase 3 — Integration Test (Bersama)
- [ ] Test `POST /a/ttkcc/createTTKCC` dari TTKCC2.vue → data masuk DB
- [ ] Test `GET /a/ttkcc/listTTKCC` tampil di TTKCCList.vue
- [ ] Test `GET /a/ttkcc/getTTKCC/{btid}` untuk detail
- [ ] Test validasi: field kosong → error yang benar
- [ ] Test edge case: koli = 0, berat = 0, harga = 0

### Phase 4 — Merge & Deploy
- [ ] Dev A buat PR `feature/ttkcc2-frontend` → main
- [ ] Dev B buat PR `feature/ttkcc2-backend` → main
- [ ] Review cross: Dev A review BE, Dev B review FE
- [ ] Merge BE dulu (tidak ada FE dependency)
- [ ] Merge FE setelah BE merged dan tested

---

## 7. Hal Kritis yang Harus Dikonfirmasi

> [!CAUTION]
> Jangan mulai coding sebelum poin-poin ini dikonfirmasi.

### 7.1 Akses Database Lintas DB
Query `ttkccListHeader` dan `ttkccGetHeader` menggunakan JOIN antara `ta15_wlogisticsidb_ap.ENTBT` dan `swb.ENTBA`, `swb.ENTCAREA`. Harus konfirmasi bahwa:
- User MySQL `dwtb` (dari `etc/dcaf.yaml`) punya `SELECT` privilege ke `ta15_wlogisticsidb_ap`
- User `dwtb` juga punya `INSERT` ke `ta15_wlogisticsidb_ap.ENTBT`

### 7.2 Nomor TTK Auto-Generate
Di sandbox DW, nomor TTK di-generate otomatis oleh engine DW. Di TTKCC.pm kita pakai `SELECT ATRBTNO WHERE ATRBTID = newBtid` — ini berfungsi hanya jika DB sudah punya trigger atau default value untuk `ATRBTNO`. Konfirmasi ke DBA apakah `ATRBTNO` auto-generate via trigger, atau harus kita generate sendiri.

### 7.3 Mapping Kolom ATRBTP yang Belum Pasti
Beberapa mapping kolom di SQL masih asumsi berdasarkan query DRP yang sudah ada:
- `ATRBTP22` = isBubble
- `ATRBTP21` = isKayu
- `ATRBTP25` = beratSurcharge
- `ATRBTP24` = biayaPackingBubble

Konfirmasi ke tim yang mengelola sistem DW lama, atau cek langsung struktur tabel `ta15_wlogisticsidb_ap.ENTBT` sebelum deploy.

### 7.4 Kolom `buktiKembali` dan `biayaTambahan`
Dua field array ini (checkbox) belum ada mapping kolom yang pasti di ENTBT. Kemungkinan disimpan sebagai bitmask integer atau tabel relasi terpisah. Perlu cek di DW lama.

---

*Dokumen ini dibuat: 2026-11-06. Update jika ada perubahan API Contract atau temuan baru.*
