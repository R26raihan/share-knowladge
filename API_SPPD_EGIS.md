# API SPPD EGIS — Pengajuan dan Realisasi

Diperbarui: 5 November 2026. Referensi berdasarkan implementasi aplikasi mobile `ioffice_plneg`, bukan hasil pengujian server langsung. Hanya endpoint berawalan `/api/v1` yang dicantumkan pada tabel API.

## Koneksi dan autentikasi

URL: `{BASE_URL_ODOO}/api/v1/...`. Base URL berasal dari konfigurasi environment melalui `ApiConfig.baseUrl` dan dapat dioverride saat membuat `ApiFactory`. Jangan menggandakan `/api/v1` pada base URL.

Header: `Authorization: Bearer <access_token>`, ditambahkan oleh interceptor `DioClient`. Payload biasa berupa JSON; unggahan menggunakan `multipart/form-data` dengan boundary dari HTTP client. ID pada contoh hanya ilustrasi; gunakan ID dari respons/master aktual.

## Pengajuan SPPD

| Method | Endpoint | Parameter / body yang dikirim mobile |
| --- | --- | --- |
| GET | `/api/v1/sppd/requests` | Query `page=1`, `limit=5`; opsional `search`, `state`, `employee_id`, `date_from`, `date_to`. Wrapper juga menerima tambahan `queryParameters`. |
| GET | `/api/v1/sppd/requests/{id}` | ID pengajuan. |
| POST | `/api/v1/sppd/requests` | JSON pengajuan, lihat contoh di bawah. |
| POST | `/api/v1/sppd/attachments/upload` | Multipart: `file`; opsional `sppd_request_id` (integer). Nama file dapat dioverride oleh caller. |
| POST | `/api/v1/sppd/requests/{id}/submit` | Tanpa body eksplisit. Ajukan draft. |
| POST | `/api/v1/sppd/requests/{id}/get-cost-list` | JSON `{}`. Ambil biaya dari pemetaan biaya. |
| GET | `/api/v1/sppd/need-approval` | Query `hanya_bisa_approve=false`, `page=1`, `limit=10`; opsional `state`, `date_from`, `date_to`, `search`. `state=all` diabaikan dan spasi dalam state diganti `_`. |
| GET | `/api/v1/sppd/approvals/admin` | Query `state=approve_travel_admin`, `limit=5`, `offset=0`; opsional `search`, `employee_id` (integer). |
| POST | `/api/v1/sppd/requests/{id}/approve-atasan` | Tanpa body eksplisit. |
| POST | `/api/v1/sppd/requests/{id}/approve-travel-admin` | Tanpa body eksplisit. |
| POST | `/api/v1/sppd/requests/{id}/reject` | Tanpa body eksplisit; wrapper saat ini tidak mengirim alasan penolakan. |
| POST | `/api/v1/sppd/requests/{id}/cancel` | Tanpa body eksplisit. |

### Payload pembuatan pengajuan

Contoh struktur yang diserialisasi `CreateSppdRequest.toJson()`; bukan pernyataan bahwa semua field wajib di backend. Tanggal contoh menggunakan `YYYY-MM-DD`.

```json
{
  "employee_id": 123,
  "activity": "Perjalanan dinas koordinasi",
  "source_document": "ST/2026/001",
  "document_date": "2026-11-05",
  "start_date": "2026-11-06",
  "end_date": "2026-11-07",
  "sppd_location": 1,
  "sppd_location_line_id": 1,
  "tipe_perjalanan": 1,
  "description": "Koordinasi pekerjaan",
  "is_lumpsum": false,
  "trips": [{
    "country_from_id": 100,
    "country_to_id": 100,
    "kota_asal_id": 10,
    "kota_tujuan_id": 20,
    "start_date": "2026-11-06",
    "end_date": "2026-11-07"
  }],
  "transportations": [{
    "country_from_id": 100,
    "country_to_id": 100,
    "city_from": 10,
    "city_to": 20,
    "date_ticket": "2026-11-06",
    "transportation": 1,
    "price": 500000
  }],
  "documents": [{"url": "https://files.example.com/surat-tugas.pdf"}]
}
```

ID bertipe integer, harga bertipe number, `is_lumpsum` boolean, dan field teks/tanggal bertipe string. Nilai `null` dibuang. Array `trips`, `transportations`, dan `documents` hanya dikirim jika tidak null. Field kota pada `trips` bernama `kota_asal_id`/`kota_tujuan_id`, sedangkan pada `transportations` bernama `city_from`/`city_to`.

Perilaku serializer saat ini: `document_date` fallback ke `start_date`, `sppd_location_line_id` default `1`, `is_lumpsum` default `false`; `source_document` fallback ke internal order lama atau nomor `ST/...` yang dibentuk mobile. `sppd_location` fallback ke `sppd_by_location_id`; `tipe_perjalanan` fallback ke `sppd_by_type_id`. **Field `sppd_type` tidak dikirim** karena kategori SPPD sementara disembunyikan pada UI. Field lama seperti `erp_prk`, `erp_coa`, dan `sppd_details` bukan bagian payload JSON baru ini.

### Urutan pengajuan di aplikasi

1. POST `/api/v1/sppd/requests` untuk membuat draft dan mengambil ID pengajuan dari respons.
2. Jika ada lampiran lokal, upload setiap file melalui `/api/v1/sppd/attachments/upload` dengan `sppd_request_id` tersebut.
3. POST `/api/v1/sppd/requests/{id}/submit` untuk mengajukan ke approval.
4. Gunakan daftar need-approval/admin dan aksi approval sesuai tahap serta hak pengguna.

Create menerima HTTP 200 atau 201. Mobile juga memeriksa `status=error` atau `success=false`. Jika respons create tidak menyediakan ID, proses submit dihentikan. Create, upload, dan submit merupakan request terpisah; kegagalan sesudah create dapat meninggalkan draft.

## Realisasi SPPD

`{realization_id}` adalah ID realisasi, berbeda dari ID pengajuan. `{id_transportation}` adalah ID baris transportasi realisasi, bukan ID master moda transportasi.

| Method | Endpoint | Parameter / body yang dikirim mobile |
| --- | --- | --- |
| GET | `/api/v1/sppd/realizations` | Query `include_split=false`, `limit=5`, `offset=0`; opsional `status`, `sppd_request_id` (integer), `employee_id`, `search`, `date_from`, `date_to`. |
| GET | `/api/v1/sppd/realizations/{realization_id}` | Detail realisasi. |
| GET | `/api/v1/sppd/realizations/need-approval` | Query `hanya_bisa_approve=false`, `page=1`, `limit=10`; opsional `state`, `date_from`, `date_to`, `search`. |
| POST | `/api/v1/sppd/realizations/{realization_id}/submit` | Tanpa body eksplisit. Draft ke tahap approval atasan. |
| POST | `/api/v1/sppd/realizations/{realization_id}/approve` | Tanpa body eksplisit. Menyetujui tahap yang berjalan. |
| POST | `/api/v1/sppd/realizations/{realization_id}/reject` | JSON `{"reason":"Alasan penolakan"}`. `reason` wajib pada wrapper mobile. |
| POST | `/api/v1/sppd/realizations/{realization_id}/pengajuan-ulang` | Tanpa body eksplisit. Ajukan ulang realisasi rejected setelah koreksi harga tiket/e-ticket. |
| POST | `/api/v1/sppd/realizations/{realization_id}/transportations/{id_transportation}/eticket` | Multipart: `file` dan `hargaTicket` (number). Perhatikan kapitalisasi `hargaTicket`. |
| POST | `/api/v1/sppd/realizations/{realization_id}/documents` | Multipart: `file`; opsional `description` (string nonkosong). |
| DELETE | `/api/v1/sppd/realizations/{realization_id}/documents/{document_id}` | ID dokumen realisasi; tanpa body eksplisit. |

### Alur bukti perjalanan dan pengajuan ulang

Ambil detail realisasi, unggah e-ticket per baris transportasi beserta harga, lalu unggah dokumen pendukung. Invoice, boarding pass, dan e-ticket umum pada form bukti perjalanan menggunakan endpoint **documents realisasi**, bukan attachments pengajuan. E-ticket yang terkait baris transportasi menggunakan endpoint **transportations/.../eticket**.

Form mobile memvalidasi harga tiket sebagai angka >= 0 untuk setiap e-ticket transportasi yang dipilih. Timeout kirim/terima unggahan pengajuan, dokumen, dan e-ticket adalah 120 detik.

Gunakan kapabilitas respons detail seperti `can_edit`, `can_submit`, `submit_block_reason`, `can_approve`, dan `can_pengajuan_ulang` untuk menentukan aksi yang tersedia. Detail juga dibaca melalui field `approval_steps`, `trips`, `transportations`, `documents`, `reject_reason`, `request_amount`, `untaxed_amount`, `selisih_amount`, dan `selisih_status`.

Wrapper `SppdApi` saat ini tidak menyediakan POST create maupun PUT/PATCH edit realisasi pada `/api/v1/sppd/realizations`. Dokumentasi ini tidak mengasumsikan endpoint tersebut tersedia.

## Master pendukung form

| Method | Endpoint | Query yang dikirim mobile |
| --- | --- | --- |
| GET | `/api/v1/sppd/locations` | `limit=5`, `offset=0`. |
| GET | `/api/v1/sppd/location-lines` | Opsional `sppd_location_id`, `name`. Untuk pilihan `sppd_location_line_id`. |
| GET | `/api/v1/sppd/tipe-perjalanan` | Tanpa query. Dipanggil melalui `getTypeCategory()`. |
| GET | `/api/v1/sppd/trips` | `limit=5`, `offset=0`; opsional `name`, `is_tad` (dikirim string boolean). Master kategori `sppd_type`; field ini saat ini tidak diserialisasi pada create. |
| GET | `/api/v1/sppd/transportations` | `limit=5`, `offset=0`. |
| GET | `/api/v1/sppd/options/cities` | `limit=5`, `offset=0`; opsional `search`. |
| GET | `/api/v1/sppd/options/countries` | `page=1`, `limit=100`; opsional `search`, `sppd_location_line_id`. |
| GET | `/api/v1/master/provinces` | Opsional `search`. |
| GET | `/api/v1/master/cities` | Opsional `province_id` (> 0), `jenis`, `search`. |

### GET negara / country

Endpoint: `/api/v1/sppd/options/countries`. Dipanggil oleh `getCountries()` untuk pilihan negara SPPD.

| Query | Tipe | Perilaku mobile |
| --- | --- | --- |
| `search` | string | Opsional; tidak dikirim jika kosong. |
| `sppd_location_line_id` | integer | Opsional pada wrapper; digunakan untuk memfilter negara berdasarkan pilihan SPPD Location Name. Bukan `sppd_location_id`. |
| `page` | integer | Default 1. |
| `limit` | integer | Default 100. |

Contoh request (ID hanya ilustrasi):

```http
GET /api/v1/sppd/options/countries?sppd_location_line_id=2&search=Japan&page=1&limit=100
Authorization: Bearer <access_token>
```

Datasource menggabungkan hasil seluruh halaman: mulai `page=1`, naik selama `pagination.has_next == true`. Jika `pagination` tidak tersedia atau `has_next` bukan true, perulangan berhenti. HTTP selain 200 atau `success=false` dianggap gagal. Ketika pilihan location line di form berubah, aplikasi memuat negara menggunakan ID location line yang dipilih.

### GET provinsi

Endpoint: `/api/v1/master/provinces`. Dipanggil oleh `getProvinceList()` melalui `getMasterProvinces()`.

| Query | Tipe | Perilaku mobile |
| --- | --- | --- |
| `search` | string | Opsional; tidak dikirim jika kosong. |

```http
GET /api/v1/master/provinces?search=Jawa
Authorization: Bearer <access_token>
```

Tanpa pencarian, panggil `/api/v1/master/provinces`. Wrapper ini tidak mengirim `page`, `limit`, atau filter negara. ID provinsi dipakai sebagai `province_id` pada request kabupaten/kota. Pada BLoC create SPPD, pencarian provinsi juga dapat dilakukan lokal terhadap master yang sudah dimuat.

### GET kabupaten / kota

Endpoint: `/api/v1/master/cities`. Inilah endpoint yang digunakan datasource untuk daftar kabupaten/kota dan daftar berdasarkan provinsi.

| Query | Tipe | Perilaku mobile |
| --- | --- | --- |
| `province_id` | integer | Opsional; hanya dikirim jika > 0. Memfilter berdasarkan ID provinsi. |
| `jenis` | string | Opsional di wrapper; datasource `getCityList()` dan `getCityByProvince()` saat ini mengirim **`kabupaten`**. |
| `search` | string | Opsional; tidak dikirim jika kosong. |

Semua kabupaten/kota yang diminta form:

```http
GET /api/v1/master/cities?jenis=kabupaten
Authorization: Bearer <access_token>
```

Berdasarkan provinsi, dengan pencarian:

```http
GET /api/v1/master/cities?province_id=10&jenis=kabupaten&search=Bandung
Authorization: Bearer <access_token>
```

Nilai `province_id=10` hanya contoh. Jangan mengasumsikan kode wilayah BPS sebagai ID master backend. Saat provinsi dipilih, BLoC memanggil `getCityByProvince(provinceId, '')`; pencarian kota juga tersedia lokal menggunakan nama dan relasi provinsi. Tidak ada parameter pagination pada wrapper master ini. Nilai enum `jenis` selain yang dikirim aplikasi belum diverifikasi terhadap backend.

### GET opsi kota SPPD

Endpoint terpisah: `/api/v1/sppd/options/cities`, melalui `getOptionCities()`.

| Query | Tipe | Default |
| --- | --- | --- |
| `limit` | integer | 5 |
| `offset` | integer | 0 |
| `search` | string | Opsional; diabaikan jika kosong. |

```http
GET /api/v1/sppd/options/cities?search=Bandung&limit=5&offset=0
Authorization: Bearer <access_token>
```

Endpoint opsi kota ini tersedia pada service, sedangkan `getCityList()`/`getCityByProvince()` menggunakan `/api/v1/master/cities`. Wrapper opsi kota tidak menerima `province_id` maupun `jenis`.

### GET lokasi SPPD dan Location Name

1. Ambil jenis/lokasi SPPD melalui `/api/v1/sppd/locations?limit=5&offset=0`.
2. Ambil nama lokasi melalui `/api/v1/sppd/location-lines?sppd_location_id=1`. Query opsional lain adalah `name` untuk nama location line; wrapper ini tidak mengirim pagination.
3. Untuk negara yang sesuai nama lokasi, gunakan `/api/v1/sppd/options/countries?sppd_location_line_id=2&page=1&limit=100`.

`sppd_location_id=1` dan `sppd_location_line_id=2` adalah contoh dari dua master berbeda. Payload create memakai nama field **`sppd_location`** untuk lokasi dan **`sppd_location_line_id`** untuk nama lokasi.

### GET tipe perjalanan, kategori SPPD, dan transportasi

| Keperluan | Contoh request | Penggunaan |
| --- | --- | --- |
| Tipe perjalanan | `/api/v1/sppd/tipe-perjalanan` | `getTypeCategory()`, tanpa query; pilihan untuk `tipe_perjalanan`. |
| Kategori SPPD | `/api/v1/sppd/trips?limit=5&offset=0` | Opsional `name` dan `is_tad=true`/`false`. Master `sppd_type`; kategori sedang disembunyikan dan field `sppd_type` tidak dikirim oleh serializer create. |
| Moda transportasi | `/api/v1/sppd/transportations?limit=5&offset=0` | ID pilihan dikirim pada `transportations[].transportation`. |

### Bentuk respons master yang dibaca client

`SppdMasterResponse` menerima array pada `data` atau `data.content`. Contoh berikut adalah ilustrasi format parser, bukan rekaman respons server:

```json
{
  "data": [
    {"id": 20, "name": "Kabupaten contoh", "province_id": 10, "province_name": "Provinsi contoh"}
  ]
}
```

Parser membaca `id` (integer atau string angka) dan `name`; nama juga dapat berasal dari `province_name`, `city_name`, `country_name`, atau `transportation_name`. Relasi provinsi dapat dibaca melalui `state_id`, `province_id`, atau `provinceId`, dan namanya melalui `state_name`, `province_name`, atau `provinceName`. Location line mendukung `sppd_location_id` dan `sppd_location_name`. Gunakan ID item terpilih untuk payload, bukan label nama.

Pemetaan ke payload pengajuan:

| Pilihan master | Field payload |
| --- | --- |
| Negara asal / tujuan | `trips[].country_from_id` / `country_to_id`; juga `transportations[].country_from_id` / `country_to_id` jika diisi. |
| Kabupaten/kota asal / tujuan trip | `trips[].kota_asal_id` / `kota_tujuan_id`. |
| Kabupaten/kota asal / tujuan transportasi | `transportations[].city_from` / `city_to`. |
| Provinsi | Dipakai untuk filter kota; tidak ada field provinsi tersendiri pada serializer trip baru. |
| Lokasi SPPD / nama lokasi | `sppd_location` / `sppd_location_line_id`. |

## Pagination, respons, dan batas cakupan

Pengajuan memakai `page`/`limit`; daftar realisasi dan approval admin memakai `offset`/`limit`; need-approval memakai `page`/`limit`. Filter daftar pengajuan memakai **state**, sedangkan daftar realisasi memakai **status**. Pada wrapper kedua daftar, alias argumen `startDate`/`endDate` dikirim sebagai `date_from`/`date_to`; argumen `dateFrom`/`dateTo` diprioritaskan.

Parser daftar pengajuan mendukung `data: [...]` dengan `pagination` berisi `page`, `limit`, `total_data`, `total_page`, `has_next`, serta beberapa format kompatibilitas. Jangan menyamakan `count` jumlah item halaman dengan total data. Dokumen ini mencatat parameter dan field yang dipakai client; skema respons lengkap, kewajiban field backend, batas ukuran/tipe file, dan seluruh enum state perlu merujuk kontrak backend.

Beberapa wrapper lama tetap berada di kode, misalnya `sppd/api/pascaSppd` untuk `sppdReport`/`submitReport`, `sppd/api/adjustment`, dan endpoint reimbursement. Path tersebut **bukan API v1** dan tidak diubah menjadi `/api/v1` dalam dokumentasi ini.

## Sumber implementasi

Path relatif terhadap root repository `ioffice_plneg`:

- `lib/core/network/api/sppd/sppd_endpoints.dart` — path endpoint.
- `lib/core/network/api/sppd/sppd_api.dart` — method HTTP, query, body, multipart, timeout.
- `lib/data/models/create_sppd_request.dart` — payload pengajuan aktual.
- `lib/data/datasources/remote/isppd_remote_ds.dart` — rangkaian create/upload/submit dan penanganan respons.
- `lib/data/models/isppd_model.dart` dan `lib/data/models/sppd_realization_model.dart` — parser respons.
- `lib/presentation/screens/more/isppd/controllers/upload_bukti_perjalanan_controller.dart` — routing bukti perjalanan.
- `lib/core/network/api/api_factory.dart`, `lib/core/network/api_config.dart`, `lib/core/network/dio_client.dart` — base URL dan autentikasi.
