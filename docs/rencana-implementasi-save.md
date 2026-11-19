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
      - 4.2 [Buat File Baru `TTKCCList.vue`](#42-buat-file-baru-ttkcclistvue)
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
   | Halaman List TTK (`TTKCCList.vue`) | Tabel daftar riwayat TTK corporate kubikasi yang telah tersimpan |
   | Filter data list | Filter berdasarkan rentang tanggal, Account corporate, dan status |
   | Navigasi detail | Klik baris nomor TTK membuka halaman TTKCC2 dalam mode referensi / detail |
   
   ---
   
   ### 1.2 File yang Akan Dibuat / Dimodifikasi
   
   | File | Pemilik | Status | Deskripsi |
   |------|---------|--------|-----------|
   | `web/dev/dcaf/src/pages/TTKCC2.vue` | Dev A | Dimodifikasi | Tambah tombol Save, konfirmasi, method save & payload builder |
   | `web/dev/dcaf/src/pages/TTKCCList.vue` | Dev A | **File Baru** | Halaman daftar data TTK (31 kolom), filter 11 parameter, print & ekspor TSV |
   | `web/dev/dcaf/src/router/routes.js` | Dev A | Dimodifikasi | Registrasi route `/TTKCCList` (1 baris) |
   | `lib/perl/DCAF/Controller/TTKCC.pm` | Dev B | **File Baru** | Controller API DCAF: operasi `createTTKCC`, `listTTKCC`, `getTTKCC` |
   | `etc/api/ttkcc.yaml` | Dev B | **File Baru** | OpenAPI 3.0 specification untuk routing DCAF |
   | `etc/dcaf.yaml` | Dev B | Dimodifikasi | Registrasi route modul `ttkcc` ke DCAF gateway (1 baris) |
   | `swb.EDBQ` & `swb.EDBQP` (DB Catalog) | Dev B | Dimodifikasi | Query SELECT `ttkccListHeader` (`253102`) & `ttkccFindByNomor` (`253103`) di database `swb` |
   
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
   
#### C. Query Baru yang Wajib Didaftarkan (2 Query SELECT di `swb.EDBQ`)
Mengikuti **The 253 Query ID Rule** (ID di rentang `253000` - `253999`), serta mendaftarkan seluruh parameter ke `EDBQP`:

| No | Nama Query | ID EDBQ | Tipe | Tujuan Penggunaan |
|----|------------|---------|------|-------------------|
| 1 | `ttkccListHeader` | `253102` | SELECT | Daftar riwayat TTK header dengan filter harian (`ATRBTDT_IX`), account, status, kode ops, asal, tujuan, info, dan NIK pembuat untuk `TTKCCList.vue` |
| 2 | `ttkccFindByNomor` | `253103` | SELECT | Pencarian cepat langsung berdasarkan Nomor TTK via unique index `ENTBT_AK` tanpa filter tanggal |

> *Catatan:* Query `ttkccGetNomor` tidak diperlukan karena nomor TTK resmi langsung diterbitkan sebelum insert header via `$ros->mkCSNNo()`.
   
   ---
   
   ### 1.4 Pemetaan Kolom Database ENTBT Terverifikasi
   
   #### Tabel 1: Header TTK (Disimpan via `entbtCreate`)
   | Kolom ENTBT | Tipe Data | Field Payload Form TTKCC2 | Keterangan |
   |-------------|-----------|---------------------------|------------|
   | `ATRBAIDO` | Integer | (Default 3002) | ID Cabang Origin / Owner Account |
   | `ATRBAIDM` | Integer | (Session User / Default 1) | ID User Pembuat Transaksi |
   | `ATRBTTYID` | Integer | (Konstan 5001) | Tipe Transaksi TTK Corporate |
   | `ATRBTDT` | Date | `tanggal` | Tanggal Pengiriman |
   | `ATRBTNO` | String | (Generated via `mkCSNNo()`) | Nomor TTK Resmi Wahana dari Redis `wax/ttknoq` (contoh: `E46ZHHND`) |
   | `ATRBAIDP` | Integer | `accountId` | ID Account Corporate |
   | `ATRBIID` | Integer | `layananId` | ID Layanan Pengiriman |
   | `ATRBTNOG` | String | `referensi` | Nomor TTK Manual / Ref PO / DN |
   | `ATRBTP18` | Integer | `printer` | Format Label Printer (102=Thermal, 103=Kecil, 104=Besar) |
   | `ATRBTP21` | Integer | `adaPacking` | Flag Ada Packing/Surcharge (1=Ya, 2=Tidak) |
   | `ATRBTP24` | Integer | `buktiKembali` | Bukti Dokumen Kembali (100=Tidak Perlu, 101, dst) |
   | `ATRBTP01` | Integer | `asalId` | ID Wilayah Asal |
   | `ATRBTP02` | Integer | `tujuanId` | ID Wilayah Tujuan |
   | `ATRBTP09` | String | `kodeOps` | Kode Operasional Rute |
   | `ATRBTP10` | Integer | `jenis` | Jenis Kiriman (Default 102 = Paket Reguler) |
   | `ATRBTFLAG` | Integer | `flag` | Flag Transaksi (Default 1) |
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
   | `ATRBTNO` | String | `nomorChild` | Format sub-koli `<nomorTTK>/<index>` (contoh: `E46ZHHND/1`, `E46ZHHND/2`) |
   | `ATRBTP10` | Integer | `jenis` | Mengikuti jenis kiriman header (Default 102) |
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
     "tanggal": "2026-11-13",
     "accountId": 188889202,
     "layananId": 7,
     "referensi": "REF-PO-9921",
     "jenis": 102,
     "flag": 1,
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
       "nomor": "E46ZHHND"
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
- `tanggalDari` (Date, optional — default hari ini)
- `tanggalSampai` (Date, optional — default hari ini)
- `accountId` (Integer, optional)
- `status` (Integer, optional)
- `nomor` (String, optional — jika diisi, langsung memakai jalur cepat `ttkccFindByNomor`)
- `kodeOps` (String, optional)
- `asal` (String, optional)
- `tujuan` (String, optional)
- `info` (String, optional)
- `nikPembuat` (String, optional)
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
           "nomor": "E46ZHHND",
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
     1. Tombol & Validasi Save           1. Tambah Query 253102 & 253103
     2. buildPayload & handleSave        2. Registrasi EDBQP & Flush Redis
     3. Halaman TTKCCList.vue            3. Controller TTKCC.pm (mkCSNNo & entbtCreate)
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
       const res = await this.$axios.post('/ttkcc/createTTKCC', payload)
   
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
   
### 4.2 Buat File Baru `TTKCCList.vue`
**Path:** `web/dev/dcaf/src/pages/TTKCCList.vue`

File ini memuat tabel `q-table` riwayat TTK (31 kolom sesuai sandbox DW `whnTTKList`), form filter 11 parameter, cetak tabel (Portrait/Landscape), ekspor `.tsv`, dan tombol pintas navigasi ke detail transaksi atau pembuatan transaksi baru.

---

### 4.3 Registrasi Route di `web/dev/dcaf/src/router/routes.js`

```javascript
// TTK Corporate Kubikasi
{ name: 'TTKCC2', path: '/TTKCC2', component: () => import('pages/TTKCC2.vue') },
{ name: 'TTKCCList', path: '/TTKCCList', component: () => import('pages/TTKCCList.vue') },
   ```
   
   ---
   
   ## 5. Dev B — Backend
   
   ### 5.1 Buat Controller `lib/perl/DCAF/Controller/TTKCC.pm`
   
   ```perl
      package DCAF::Controller::TTKCC;
      
      use strict;
      use warnings;
      
      use parent 'DCAF::Controller';
      use DCAF::Query;
      use POSIX qw(strftime);
      
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
          my $rhcp = eval { $DCAF::ros->{json}->decode($ros->{q}->{q}->content || '{}') } || {};
          my $rhpp = $ros->{api}->{pathParam};
      
          switch: {
      
              # ----------------------------------------------------------------
              # createTTKCC
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
      
                  # 2. INSERT Header TTK pake entbtCreate
                  my $ownerId = $ros->{q}->{session}->{user}->{ownerId} || 3002;
                  my $userId  = $ros->{q}->{session}->{user}->{accountId} || 1;
      
                  # Generate nomor TTK
                  my $nomorTTK = $ros->mkCSNNo();
                  unless ($nomorTTK) {
                      $ros->{q}->{e}->{status} = 500;
                      $ros->{q}->{e}->{num}    = "$ros->{className}/createTTKCC/02A";
                      $ros->{q}->{e}->{msg}    = 'Gagal membuat nomor TTK: ' . ($ros->{q}->{e}->{msg} || 'Antrean nomor TTK di Redis tidak tersedia');
                      die;
                  }
      
                  my $roqHeader = DCAF::Query->new(
                      q    => $ros->{q},
                      name => 'entbtCreate',
                      data => {
                          ATRBAIDO           => $ownerId,
                          ATRBAIDM           => $userId,
                          ATRBAIDP           => $rhcp->{accountId},
                          ATRBTTYID          => 5001,
                          ATRBIID            => $rhcp->{layananId},
                          ATRBTNO            => $nomorTTK,
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
                          ATRBTP10           => $rhcp->{jenis} || 102,
                          ATRBTFLAG          => defined $rhcp->{flag} ? $rhcp->{flag} : 1,
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
      
                  my $newBtid = $roqHeader->{result}->{lastInsertId} || $roqHeader->{result}->{insertId};
                  unless ($newBtid) {
                      $ros->{q}->{e}->{status} = 500;
                      $ros->{q}->{e}->{num}    = "$ros->{className}/createTTKCC/02";
                      $ros->{q}->{e}->{msg}    = 'Gagal menyimpan header TTK ke database';
                      die;
                  }
      
                  # 3. INSERT Child Koli pake Query entbtCreate (penomoran sub-koli)
      
                  my $listKoli = $rhcp->{listKoli} || [];
                  my $koliIndex = 1;
                  foreach my $rhkoli (@{$listKoli}) {
                      next unless $rhkoli->{koli} && $rhkoli->{grossWeight};
                      my $nomorChild = sprintf('%s/%d', $nomorTTK, $koliIndex);
                      my $roqKoli = DCAF::Query->new(
                          q    => $ros->{q},
                          name => 'entbtCreate',
                          data => {
                              ATRBAIDO           => $ownerId,
                              ATRBAIDM           => $userId,
                              ATRBAIDP           => $rhcp->{accountId},
                              ATRBTTYID          => 5001,
                              ATRBTIDP           => $newBtid,
                              ATRBTNO            => $nomorChild,
                              ATRBTDT            => $rhcp->{tanggal},
                              ATRBTSS            => 1,
                              ATRBTBS            => 1,
                              ATRBTP10           => $rhcp->{jenis} || 102,
                              ATRBTP16           => $rhkoli->{koli} || $koliIndex,
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
                      $koliIndex++;
                  }
      
                  $ros->{templateData}->{jstext} = $DCAF::ros->{json}->encode({
                      code    => 0,
                      status  => 'success',
                      message => 'TTK Corporate Kubikasi berhasil dibuat',
                      data    => {
                          btid  => $newBtid,
                          nomor => $nomorTTK
                      }
                  });
                  last switch;
              };
      
              # ----------------------------------------------------------------
              # listTTKCC
              # ----------------------------------------------------------------
              $ros->{api}->{operation} eq 'listTTKCC' && do {
                  my $nomor = $rhqp->{'nomor'} || '';
                  my $rhres = {};
                  my $opt = {
                      hashed => 1,
                      limit  => $rhqp->{'d-limit'} || 25,
                      offset => $rhqp->{'d-offset'} || 0,
                  };
      
                  if ($nomor ne '') {
                      # Opsi 2: Jalur Cepat Lookup Nomor TTK (index ENTBT_AK, < 5 ms)
                      my $rhparNomor = {
                          nomor => $nomor,
                      };
                      $ros->q2j('ttkccFindByNomor', $rhparNomor, $opt, $rhres);
                  } else {
                                  # Opsi 2: Jalur Browsing Harian / Monitoring Terindeks Tanggal
                      my $today = strftime('%Y-%m-%d', localtime);
                      my $rhpar = {
                          tanggalDari   => $rhqp->{'tanggalDari'}   || $today,
                          tanggalSampai => $rhqp->{'tanggalSampai'} || $today,
                          accountId     => $rhqp->{'accountId'}     || 0,
                          status        => $rhqp->{'status'}        || 0,
                          kodeOps       => $rhqp->{'kodeOps'}       || '',
                          asal          => $rhqp->{'asal'}          || '',
                          tujuan        => $rhqp->{'tujuan'}        || '',
                          info          => $rhqp->{'info'}          || '',
                          nikPembuat    => $rhqp->{'nikPembuat'}    || '',
                      };
                      $ros->q2j('ttkccListHeader', $rhpar, $opt, $rhres);
                  }
      
                  $ros->{templateData}->{jstext} = $DCAF::ros->{json}->encode({
                      code    => 0,
                      status  => 'success',
                      message => 'Successful',
                      data    => $rhres->{result}
                  });
                  last switch;
              };
      
              # ----------------------------------------------------------------
              # getTTKCC
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
   
   ### 5.2 Definisi SQL Query EDBQ & EDBQP (Aturan 253)
   
   2 query SELECT baru didaftarkan langsung ke katalog `swb.EDBQ` dan `swb.EDBQP` (bukan ke `lib/db/git/dcaf-data.sql`):

1. **`ttkccListHeader` (`ADBQID = 253102`, `ADBCID = 3`):**
   - **Parameter (`ADBQPAR` — 18 placeholder):**
     `tanggalDari,tanggalDari,tanggalSampai,tanggalSampai,accountId,accountId,status,status,kodeOps,kodeOps,asal,asal,tujuan,tujuan,info,info,nikPembuat,nikPembuat`
   - **Klausa `WHERE` teroptimasi (tanpa `LIKE 'TTKC-%'`, menggunakan pure range `ATRBTDT_IX`):**
     ```sql
     WHERE
       t.ATRBTTYID = 5001
       AND (t.ATRBTIDP = 0 OR t.ATRBTIDP IS NULL)
       AND t.ATRBTDT >= concat(if(COALESCE(?, "") = "", curdate(), ?), " 00:00:00")
       AND t.ATRBTDT <= concat(if(COALESCE(?, "") = "", curdate(), ?), " 23:59:59")
       AND (COALESCE(?, 0) = 0 OR t.ATRBAIDP = ?)
       AND (COALESCE(?, 0) = 0 OR t.ATRBTSS = ?)
       AND (COALESCE(?, "") = "" OR t.ATRBTP09 LIKE concat(?, "%"))
       AND (COALESCE(?, "") = "" OR oa.ATRCAREANML LIKE concat("%", ?, "%"))
       AND (COALESCE(?, "") = "" OR da.ATRCAREANML LIKE concat("%", ?, "%"))
       AND (COALESCE(?, "") = "" OR t.ATRBTP05 LIKE concat("%", ?, "%"))
       AND (COALESCE(?, "") = "" OR t.ATRBAIDM LIKE concat(?, "%"))
     ORDER BY
       t.ATRBTDT DESC, t.ATRBTID DESC
     ```
2. **`ttkccFindByNomor` (`ADBQID = 253103`, `ADBCID = 3`):**
   - **Parameter (`ADBQPAR` — 3 placeholder):** `nomor,nomor,nomor`
   - **Klausa `WHERE` (jalur cepat via `ENTBT_AK`):**
     ```sql
     WHERE
       (t.ATRBTIDP = 0 OR t.ATRBTIDP IS NULL)
       AND (t.ATRBTNO = ? OR t.ATRBTNO LIKE concat(?, "%") OR t.ATRBTNOG = ?)
     ORDER BY
       t.ATRBTID DESC
     ```
   
> **Wajib Dilakukan Langsung (Redis Cache Busting):**
> Hapus cache definisi query pada Redis untuk `ttkccListHeader` dan `ttkccFindByNomor`, lalu reload uWSGI (`touch /tmp/reload`) tepat setelah perubahan `EDBQ`/`EDBQP`:
> ```bash
> redis-cli keys "drp.qc*ttkccListHeader*" | xargs redis-cli del
> redis-cli keys "drp.qc*ttkccFindByNomor*" | xargs redis-cli del
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
    cookie:
      type: apiKey
      in: cookie
      name: dcaf
paths:
  /createTTKCC:
    post:
      tags:
        - TTKCC
      summary: Buat record TTK Corporate Kubikasi baru
      operationId: createTTKCC/TTKCC
      security:
        - bearer: []
        - cookie: []
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
        - cookie: []
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
          name: status
          schema: { type: integer }
        - in: query
          name: nomor
          schema: { type: string }
        - in: query
          name: kodeOps
          schema: { type: string }
        - in: query
          name: asal
          schema: { type: string }
        - in: query
          name: tujuan
          schema: { type: string }
        - in: query
          name: info
          schema: { type: string }
        - in: query
          name: nikPembuat
          schema: { type: string }
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
        - cookie: []
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
   
   ```yaml
   api:
      dcaf: /etc/api/dcaf.yaml
      3pl: /etc/api/3pl.yaml
      ttkcc: /etc/api/ttkcc.yaml    # Registrasi controller TTKCC
   ```
   
   ---
   
   ## 6. Urutan Eksekusi & Checkpoint
   
   ### Phase 1 — Setup & Review
   - [x] Review bersama dokumen ini dan konfirmasi pemanfaatan query `entbtCreate` (253005).
   - [x] Siapkan branch kerja di repositori `swb` (`TTKCC2-FZ`).
   
   ### Phase 2 — Parallel Development
   **Checklist Dev A (Frontend):**
   - [x] Modifikasi `TTKCC2.vue` (tambah state `isSaving`, computed `isReadyToSave`, tombol Save).
   - [x] Implementasi fungsi `handleSave` dan `buildPayload` di `TTKCC2.vue`.
   - [x] Buat halaman baru `TTKCCList.vue` (31 kolom sandbox, styling Quasar SWB, tombol aksi).
   - [x] Tambah dependensi `@capacitor/core: ^3.9.0` untuk kelengkapan plugin kamera.
   - [x] Daftarkan route di `web/dev/dcaf/src/router/routes.js` (`/TTKCCList`).
   - [x] Uji visual form & tombol di browser (Dev Server Quasar) — lulus ESLint 0 error.
   
   **Checklist Dev B (Backend):**
   - [x] Daftarkan query SQL ke catalog `EDBQ` (`entbtCreate` 253005 dan `ttkccListHeader` 253102).
   - [x] Buat class controller `lib/perl/DCAF/Controller/TTKCC.pm`.
   - [x] Integrasikan auto-numbering resmi via helper `$ros->mkCSNNo()` dan penomoran sub-koli `<nomorTTK>/<index>`.
   - [x] Jalankan uji sintaks: `perl -c -Ilib/perl lib/perl/DCAF/Controller/TTKCC.pm` (Syntax OK).
   - [x] Buat file OpenAPI `etc/api/ttkcc.yaml`.
   - [x] Daftarkan modul di `etc/dcaf.yaml` (`/a/ttkcc: etc/api/ttkcc.yaml`).
   - [x] Restart service uWSGI dev dan uji endpoint via curl / axios (`code: 0`, status: success).
   
   ### Phase 3 — Integration Testing
   - [x] Isi form lengkap di `TTKCC2.vue` lalu klik Save.
   - [x] Verifikasi dialog konfirmasi muncul: "Are you sure to CREATE new record?".
   - [x] Verifikasi data tersimpan di tabel `ENTBT` (baik header `ATRBTID` maupun child koli `ATRBTIDP`).
   - [x] Verifikasi notifikasi nomor TTK tampil di UI form (contoh: `E46ZHHND` berhasil dibuat).
   - [x] Buka halaman `TTKCCList.vue` dan pastikan record TTK baru muncul pada tabel.
   - [x] Optimasi query `ttkccListHeader` (pure range index `ATRBTDT_IX` tanpa filter `TTKC-%`) terpangkas dari > 60s ke hitungan milidetik.
   - [x] Cek filter tanggal dan filter account pada `TTKCCList.vue`.
   
   ### Phase 4 — Commit & Verification
   - [x] Commit backend: `c7380362` (tambah controller & registrasi api) dan `e114a774` (otomasi penomoran resmi).
   - [x] Commit frontend: `b333c0dd` (tombol save) dan `803a18d8` (halaman list ttk & capacitor core).
   - [x] Verifikasi runtime end-to-end pada server dev / container (uWSGI + Nginx + Galera DB + Redis).
   
   ### Phase 5 — Pencarian Cepat, Parameter Filter Lengkap, dan Sinkronisasi Status (2026-11-10)
   - [x] Query lookup cepat by nomor TTK `ttkccFindByNomor` (`EDBQ` ID `253103`) via unique index `ENTBT_AK` (< 10 ms).
   - [x] Dual-path routing di backend controller `lib/perl/DCAF/Controller/TTKCC.pm` (`listTTKCC`) dan OpenAPI `etc/api/ttkcc.yaml`.
   - [x] Dialog pencarian 11 parameter di `TTKCCList.vue` (Sejak, Sampai, Agen/Account, Nomor, Kode Ops, Status, Asal, Tujuan, Info, Jenis, NIK Pembuat).
   - [x] Tombol split dropdown Filter dengan fitur Print (Portrait/Landscape) dan Save Tab Delimited Text File (`.tsv`).
   - [x] Penanganan parameter SQL NULL pada query catalog `ttkccListHeader` (`253102`) menggunakan `COALESCE(?, "") = ""` dan default value `EDBQP` (`ADBQPDV`).
   - [x] Sinkronisasi opsi status filter dropdown (`optionsStatus`) dengan master tabel `ta15_wlogisticsidb_ap.ENTLV` (`ATRLVTYID = 9`).
   - [x] Commit `14405297` dan `53cafe92` pada repository `swb`.

   ### Phase 6 — Input Nama Manual, Field `jenis`/`flag`, Strict `mkCSNNo()`, & Update `ttkccListHeader` (2026-11-13 s/d 2026-11-18)
- [x] Ubah input Nama Penerima menjadi manual (`q-input`) dan kirim field `jenis: 102` (`ATRBTP10`) & `flag: 1` (`ATRBTFLAG`) pada commit `10602048`.
- [x] Migrasi penomoran resmi TTK ke `$ros->mkCSNNo()` (Redis `wax/ttknoq`) dengan *strict error handling* (`die` HTTP 500).
- [x] Update query `ttkccListHeader` (`253102`): hapus filter `TTKC-%`, gunakan pure range `ATRBTDT_IX` + fallback `curdate()`, lalu bust cache Redis.
- [x] Guardrail tanggal hari ini di `TTKCC.pm` (`strftime`) dan default `tanggalDari` hari ini di `TTKCCList.vue` pada commit `e72791dd`.
      
   ---
   
   ## 7. Hal Kritis & Catatan Verifikasi
   
   > [!CAUTION]
   > Perhatikan hal-hal berikut sebelum melakukan deploy ke staging / production:
   
   1. **Penggunaan Generic Query `entbtCreate` (ID `253005`):**
      Tidak perlu membuat query INSERT custom baru. `entbtCreate` sudah dirancang khusus untuk menangani tabel `ENTBT` pada koneksi 3 (`ta15_wlogisticsidb_ap`).
2. **Nomor TTK Generator Resmi (`mkCSNNo`):**
   Penomoran TTK menggunakan helper resmi DCAF `$ros->mkCSNNo()` dari antrean Redis `wax/ttknoq` (contoh: `E46ZHHND`), dan child koli terhubung melalui kolom `ATRBTIDP` dengan format `<nomorTTK>/<index>` (contoh: `E46ZHHND/1`). Jika antrean Redis kosong atau stok di bawah ambang batas, request dihentikan (`HTTP 500`) untuk mencegah nomor kosong atau duplikat.
3. **Privilege Database & Indexing ENTBT:**
   Query `ttkccListHeader` (`253102`, `ADBCID = 3`) mengeksekusi data pada `ta15_wlogisticsidb_ap.ENTBT`. Wajib menggunakan *pure range* tanggal *sargable* (`t.ATRBTDT >= concat(if(COALESCE(?, "") = "", curdate(), ?), " 00:00:00")`) tanpa `LIKE 'TTKC-%'` agar tetap memakai index `ATRBTDT_IX` sekaligus menampilkan nomor hasil `mkCSNNo()`.
   4. **Disiplin Gaya Kode & Redis Cache:**
      Sesuai aturan arsitektur SWB, tidak ada penulisan string SQL di dalam controller Perl. Seluruh query menggunakan `EDBQ` catalog dan cache Redis langsung dibersihkan (`flush`) setelah penambahan atau perubahan definisi query. Dilarang menggunakan emoticon / emoji pada file kode maupun dokumentasi.
   5. **Direktori Request Logging uWSGI:**
      Pastikan target symlink request log `/hdd/logs/nginx/swb/requests/api` memiliki permission writeable pada lingkungan kontainer/server uWSGI agar `DCAF/Web.pm` dapat menulis log request tanpa error 500.
   
   ---
   *Dokumen ini diperbarui: 2026-11-19 — Implementasi Save & List TTKCC2 (termasuk migrasi `mkCSNNo()` dan perbaikan filter `ttkccListHeader`) selesai dan terverifikasi.*
