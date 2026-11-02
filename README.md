Bagian yang **masih belum sesuai** antara `ui.js`, kode di TTKCC2.vue dan query database yang ada:

---

### 1. Perhitungan Ongkir Layanan Reguler (`setBiayaKirim`)
* **Di `ui.js`**:
  Query `whnPriceList` mengembalikan flag `usetotal`:
  * Jika `usetotal == 1` (layanan kargo/bertingkat): mengambil `total` dan `kg` dari query.
  * Jika `usetotal == 0` (layanan reguler, contohnya ATRBIID = 7): ongkir dihitung manual dengan rumus:
    ```javascript
    biaya = berat * biayaperkg;
    ```
    *(Hal ini karena di SQL `whnPriceList`, kolom `total` untuk layanan reguler menghasilkan nilai `0`).*
* **Di `TTKCC2.vue`**:
  Flag `row.usetotal` diabaikan dan langsung menembak `this.form.biayaKirim = totalOngkir`. 
  **Akibatnya**: Jika user memilih layanan reguler, nilai ongkir akan menjadi `0` atau salah hitung.

---

### 2. Biaya Tambahan Di-Hardcode (Belum Terhubung ke Query DB)
Di `ui.js`, fungsi `setBiayaTambahan` memanggil query `whnPriceListAdditional` (`baid`, `biid`) untuk mengambil tarif dinamis dari tabel `ENTOTRF`: batas berat (`rangesa`, `rangeso`), kelipatan (`rangein`), tarif unit (`val`), dan tarif minimum (`minval`).

Di `TTKCC2.vue`, **sama sekali tidak memanggil query biaya tambahan**, melainkan memakai angka statis (hardcoded):
* **Packing Kayu**: Di-hardcode `Math.max(gwt * 2000, 15000)`.
* **Packing Bubble**: Di-hardcode `Math.max(keliling * 2500, 2500)`.
* **Overweight Surcharge**: Di-hardcode berat `> 50 kg` dan pengali `1x biayaPerKg`.
* **Pickup**: Di-hardcode `Math.max(berat * 500, 10000)`.
* **Realokasi**: Di-hardcode `Math.max(berat * 1000, 15000)`.
* **Diskon Layanan (`SVC`)**: Tidak ada query diskon (`biayaDiskon` selalu `0`).

---

### 3. Query Database `drpWhnPriceListAdditional` Terkunci Hanya untuk Packing Kayu
* Di database DCAF saat ini, query yang ada adalah `drpWhnPriceListAdditional` (Query ID: 8229).
* Namun, di baris akhir SQL query tersebut terdapat filter:
  ```sql
  and t6.ATRBICCD = 'ASVC-PKW'
  ```
* **Masalah**: Query ini hanya mengembalikan data Packing Kayu (`ASVC-PKW`). Query ini **tidak bisa** digunakan untuk mengambil biaya Pickup (`ASVC-PIC`), Realokasi (`ASVC-PKR`), Asuransi (`ASVC-INS`), Bubble (`ASVC-PKB`), Surcharge (`ASVC-SCG`), dan Diskon (`SVC`).

---

### 4. Perbedaan Persentase Premi Asuransi
* **Di `ui.js`**:
  Persentase standar asuransi adalah **0.5%** (`harga * (0.5 / 100.0)`):
  ```javascript
  var asuransi = Math.ceil(harga * (0.5 / 100.0));
  ```
* **Di `TTKCC2.vue`**:
  Di-hardcode sebesar **0.2%** dan minimum 5.000:
  ```javascript
  const premi = Math.max(Math.ceil(harga * 0.002), 5000);
  ```

---

### 5. Tarif Berdasarkan Kategori Koli (`whnPriceListByCategory`)
* **Di `ui.js`**: Terdapat fungsi `setBiayaKirimKategori` yang memanggil `view/vnm.whnPriceListByCategory` berdasarkan `categoryId` tiap koli.
* **Di `TTKCC2.vue`**: Belum ada implementasi dan query `whnPriceListByCategory` belum ada di DCAF.

---

### Ringkasan Yang Perlu Dibuat / Diperbaiki:
1. **Buat Query Baru**: `whnPriceListAdditional` (berdasarkan Query 8229 tetapi hilangkan baris filter `and t6.ATRBICCD = 'ASVC-PKW'`).
2. **Perbaiki Logika di `TTKCC2.vue`**:
   * Evaluasi `row.usetotal` pada fungsi `setBiayaKirim()`.
   * Ganti nilai hardcode di `setBiayaTambahan()` dengan data dari query `whnPriceListAdditional`.
   * Koreksi persentase premi asuransi (0.5% vs 0.2%).
