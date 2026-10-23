# Panduan SSO Microsoft 365 + MFA

Buka `index.html` untuk membaca dokumentasi lengkap. Semua aset tersedia lokal; tidak membutuhkan build atau instalasi dependency. Diagram Archify tersedia di `diagram.html`, spesifikasi di `flow.sequence.json`, dokumen sumber di `FLOW_SSO_MICROSOFT_365_MFA.md`. Viewer diagram menggunakan kontrol bahasa Inggris; konten panduan berbahasa Indonesia.

## Deploy ke Netlify

1. Ekstrak ZIP bila menggunakan paket ZIP.
2. Buka Netlify → Add new project → Deploy manually (Netlify Drop).
3. Seret folder ini yang berisi `index.html`, bukan seluruh repository Flutter.
4. Netlify menerbitkan situs statis; tidak perlu build command.

Untuk deploy melalui Git: base directory `docs/microsoft-sso-guide`, publish directory `.`, build command kosong. Konfigurasi `netlify.toml` sudah disertakan.

Dokumentasi ini belum dideploy. Tidak ada tenant ID, kredensial, atau token nyata di paket.
