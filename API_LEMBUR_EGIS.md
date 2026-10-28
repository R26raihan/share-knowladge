# API Lembur EGIS

Dokumentasi integrasi API lembur yang dipakai aplikasi mobile EGIS. Informasi path dan parameter diturunkan dari implementasi Flutter; rincian otorisasi FPKT mengikuti spesifikasi endpoint yang diberikan backend.

## Konfigurasi umum

- **Base URL:** mengikuti konfigurasi environment EGIS (`ApiEndpoints.baseUrl`).
- **Autentikasi:** Bearer token pengguna yang login.
- **Header aplikasi:** `App-Source: IOFFICE_ANDROID`.
- **Format waktu pengajuan:** `yyyy-MM-dd HH:mm:ss`.
- ID lembur boleh berupa angka atau nomor seperti `OVT/00082`; aplikasi mengambil angka di akhir (`82`) untuk endpoint detail dan unduh.

Jangan menyalin access token, refresh token, atau URL berpresigned dari log HTTP ke dokumentasi.

## Endpoint

| Method | Path | Kegunaan |
| --- | --- | --- |
| GET | `/api/v1/overtime/lihat` | Daftar pengajuan lembur untuk layar Lembur. Mendukung query parameter opsional dari pemanggil. |
| GET | `/api/v1/overtime/employee-tad/{employee_id}` | Kuota dan daftar lembur pegawai TAD. Respons yang dipakai aplikasi berisi `data.overtime_quota` dan `data.overtime`. |
| GET | `/api/v1/overtime/datapegawai` | Data pegawai untuk pengajuan lembur. |
| GET | `/api/v1/overtime/detail/{overtime_id}` | Detail pengajuan lembur, termasuk `dataPegawaiOvertime` dan riwayat. |
| POST | `/api/v1/overtime/simpan` | Membuat pengajuan lembur. Body JSON mengikuti skema berikut. |
| POST | `/api/v1/overtime/upload` | Mengunggah lampiran lembur menggunakan multipart form (`FormData`). |
| GET | `/api/v1/overtime/{overtime_id}/fpkt/download?format=pdf` | Mengunduh Formulir Perintah Kerja Tambahan (FPKT) dalam PDF. Dipakai tombol unduh di detail lembur TAD. |
| GET | `/api/v1/overtime/{overtime_id}/download?format=pdf` | Endpoint unduh Surat Tugas lama yang masih tersedia di client API. |

## Unduh FPKT

Detail lembur menampilkan tombol **Download Formulir FPKT** hanya untuk pengguna TAD. Tombol tidak memerlukan `assignment_letter`; EGIS memanggil endpoint FPKT dengan ID lembur dan `format=pdf`, lalu membuka dialog simpan file.

Spesifikasi endpoint backend:

- Kontrak backend juga menerima `/api/v1/overtime/fpkt/download` dengan `overtime_id` sebagai query wajib bila ID tidak dikirim pada path. Client EGIS saat ini menggunakan bentuk path `/api/v1/overtime/{overtime_id}/fpkt/download`.
- PDF biner (`application/pdf`) secara default; format `base64`/`json` tersedia untuk respons JSON berisi base64.
- Operator menerima lembar FPKT miliknya sendiri. Manager, koordinator pembuat, HR, atau System Admin dapat menerima seluruh halaman.
- `employee_id` opsional untuk membatasi cetakan oleh Manager/Koordinator/HR dan harus merupakan operator yang tercantum pada lembur. EGIS tidak mengirim parameter ini untuk unduhan operator TAD.
- Hak akses yang disebutkan backend: operator yang tercantum, Manager, Koordinator pembuat, HR, dan System Admin.

### Contoh request

```http
GET {baseUrl}/api/v1/overtime/82/fpkt/download?format=pdf
Authorization: Bearer {access_token}
Accept: application/pdf
App-Source: IOFFICE_ANDROID
```

### Status respons yang didokumentasikan backend

| Status | Makna |
| --- | --- |
| 200 | PDF berhasil dibuat atau JSON base64 jika format JSON diminta. |
| 400 | Parameter tidak valid atau ID lembur wajib tidak dikirim. |
| 403 | Pengguna tidak memiliki akses pada lembur tersebut. |
| 404 | Lembur atau pegawai tidak ditemukan. |
| 500 | Gagal merender PDF. |

## Body pengajuan lembur

Implementasi form mengirim JSON seperti berikut. `manager_id` dan `department_id` hanya disertakan jika tersedia.

```json
{
  "start_date": "2026-10-28 17:42:00",
  "end_date": "2026-10-28 18:37:00",
  "manager_id": 123,
  "department_id": 45,
  "overtime_type_id": 1,
  "keterangan": "Penyelesaian pekerjaan",
  "lines": [
    {
      "employee_id": 1874,
      "start_date": "2026-10-28 17:42:00",
      "end_date": "2026-10-28 18:37:00",
      "target_output": 1
    }
  ]
}
```

## Contoh data detail

Endpoint detail mengembalikan object pengajuan pada `data`. Field yang dipakai layar meliputi nomor (`unix`), pengaju (`fullName`, `nip`), waktu, alasan (`reason`), lampiran, riwayat, dan daftar operator (`dataPegawaiOvertime`). Field `assignment_letter` tidak wajib untuk mengunduh FPKT.

## Sumber implementasi

- `lib/core/network/api/overtime/overtime_api.dart`
- `lib/core/network/api/overtime/overtime_endpoints.dart`
- `lib/presentation/screens/attendance/overtime/detail/detail_overtime_screen.dart`
- `lib/presentation/screens/attendance/overtime/create/create_overtime_screen.dart`
