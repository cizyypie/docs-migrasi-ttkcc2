Berdasarkan hasil perbandingan mendalam antara file ui.js (dari form legacy Dev Wahana fnm.50180000ENTBT_m_TTKCC2) dengan kode komponen web/dev/dcaf/src/pages/TTKCC2.vue, berikut adalah rincian logika dan fitur yang ada di ui.js tetapi belum diimplementasikan atau masih parsial/statis di TTKCC2.vue:

1. Biaya Tambahan yang Masih Hardcoded di TTKCC2.vue (Di ui.js Dinamis via DB)

Pada TTKCC2.vue, Asuransi, Pickup, dan Realokasi sudah berhasil membaca query database whnPriceListAdditional. Namun, tiga biaya koli berikut masih memakai angka statis/hardcoded di lokal:


1. Packing Kayu (setBiayaPacking - kode ASVC-PKW):

  - Di ui.js: Mengambil tiering tarif dari hasil query whnPriceListAdditional untuk baris ASVC-PKW (val, minval, rangesa, rangeso, rangein). Biaya dihitung per koli yang dicentang isKayu (gwt×valgwt \times valgwt×val), dijumlahkan, dan diterapkan batas minimal total (minvalminvalminval).

  - Di TTKCC2.vue: Masih menggunakan nilai statis Math.max(gwt * 2000, 15000) * koli.


2. Packing Bubble (setBiayaPackingBubble - kode ASVC-PKB):

  - Di ui.js: Mengambil tarif dari whnPriceListAdditional untuk baris ASVC-PKB. Dihitung dari penjumlahan dimensi koli (P+L+T)/100(P + L + T) / 100(P+L+T)/100 dikali tarif valvalval, lalu dibandingkan dengan batas minimal (minvalminvalminval).

  - Di TTKCC2.vue: Masih menggunakan nilai statis Math.max(keliling * 2500, 2500) * koli.


3. Surcharge Berat / Overweight (setBiayaSurcharge - kode ASVC-SCG):

  - Di ui.js: Mengambil tiering berat dari baris ASVC-SCG (rangesa sampai rangeso, kelipatan rangein, pengali tarif val×biayaPerKgval \times biayaPerKgval×biayaPerKg, dan minval).

  - Di TTKCC2.vue: Masih dipatok statis jika berat >50 kg> 50\text{ kg}>50 kg.


4. Diskon Tarif Otomatis (SVC - Diskon):

  - Di ui.js: Mengecek item dengan biccd = SVC dan isdis = 1 dari whnPriceListAdditional untuk langsung memotong nilai tarif kirim (ATRBTP25).

  - Di TTKCC2.vue: Field diskon ada di form, tetapi belum membaca data diskon otomatis dari query tarif.



2. Fitur Otomasi & Lookup Bisnis yang Belum Ada di TTKCC2.vue

1. Otomasi Bukti Balik / POD Return (setBuktiBalik):

  - Di ui.js: Memanggil view/query whnGetBABuktiBalik berdasarkan pengirim (ATRBAIDP). Jika akun pelanggan memiliki SOP bukti balik tertentu, checklist bukti balik (Surat Jalan Balik, Faktur, TTK Balik, dsb.) langsung tercentang secara otomatis.

  - Di TTKCC2.vue: Belum ada pemanggilan query whnGetBABuktiBalik maupun fungsi setBuktiBalik.


2. Lookup Nomor DRA Pelanggan Korporat (setDestByDRA):

  - Di ui.js: Memanggil whnNestleGetDRA saat nomor DRA dimasukkan. Data kota tujuan, nama penerima, alamat lengkap, dan nomor kontak langsung terisi otomatis.

  - Di TTKCC2.vue: Belum ada integrasi DRA.


3. Validasi & Lookup TTK Kirim Ulang / Pengganti (cekTTKkirimUlangdanPengganti & getDataTTKRef):

  - Di ui.js:
    - cekTTKkirimUlangdanPengganti: Memvalidasi agar nomor TTK referensi tidak dapat dibuatkan TTK pengganti lebih dari satu kali (mencegah duplikasi klaim/retur).
    - getDataTTKRef (cekdatareferensi): Otomatis menarik seluruh data dari TTK lama (pengirim, penerima, berat, dimensi koli, nilai barang) ke dalam form TTK baru.

  - Di TTKCC2.vue: Belum ada logika validasi atau penarikan data TTK referensi.


4. Lookup Kode Armada dari Plat Nomor (setKodeArmadaNopol):

  - Di ui.js: Memanggil getDataKendaranFromNopol saat plat nomor kendaraan dipilih, untuk otomatis mengisi kode armada (ATRBTP05).

  - Di TTKCC2.vue: Belum ada lookup otomatis kode kendaraan ini.


5. Validasi Barang Kiriman / Stiker Penanganan (cekbarangkiriman):

  - Di ui.js: Memanggil infobarangatk saat jenis barang dipilih. Jika jenis barang memerlukan penanganan khusus, sistem menampilkan info stiker mandatory (Stiker POD, Stiker Fragile, atau peringatan barang berbahaya).

  - Di TTKCC2.vue: Belum ada pengecekan popup/notifikasi stiker penanganan.


6. Validasi Batas Dimensi Sisi Maksimal 150 cm (setBeratKirim):

  - Di ui.js: Terdapat proteksi: jika salah satu dimensi sisi (PPP, LLL, atau TTT) melebihi 150 cm, sistem menolak input dan memunculkan peringatan "Salah satu sisi lebih dari 150cm".

  - Di TTKCC2.vue: Belum ada validasi batas 150 cm per sisi koli.


7. Tarif Kategori Komoditas Khusus (setBiayaKirimKategori):

  - Di ui.js: Memanggil whnPriceListByCategory jika terdapat categoryId khusus pada koli atau transaksi.

  - Di TTKCC2.vue: Kalkulasi tarif saat ini murni berdasarkan rute origin-destination dan berat koli.


8. Validasi Stok Jagaan TTK Agen (cekKandelTTK):

  - Di ui.js: Memanggil pv2TTKstjagang untuk memastikan nomor TTK yang diinput sesuai dengan jatah stok agen yang bersangkutan.

  - Di TTKCC2.vue: Belum ada validasi jagaan stok agen.


Rekomendasi Prioritas untuk Dikerjakan:
1. Langkah 1: Sinkronisasi kalkulasi Packing Kayu (ASVC-PKW), Packing Bubble (ASVC-PKB), dan Surcharge (ASVC-SCG) di TTKCC2.vue agar memakai baris data dari whnPriceListAdditional yang sudah aktif.
2. Langkah 2: Menambahkan validasi batas dimensi sisi koli maksimal 150 cm.
3. Langkah 3: Integrasi setBuktiBalik atau lookup TTK Referensi jika dibutuhkan untuk kebutuhan operasional saat ini.
Apakah Anda ingin kita mulai menyinkronkan kalkulasi Packing Kayu, Packing Bubble dan Surcharge dari whnPriceListAdditional terlebih dahulu?
