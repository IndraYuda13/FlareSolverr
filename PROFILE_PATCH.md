# Profile-backed FlareSolverr patch

Patch ini dibuat untuk kasus Cloudflare yang tidak cukup diselesaikan dengan cookie transplant biasa.

## Masalah yang disasar

Pada beberapa target seperti `chatgpt.com`, FlareSolverr standar bisa lolos di browser miliknya sendiri, tetapi clearance itu tidak otomatis nempel ke browser profile asli yang dipakai kerja sehari-hari.

Akibatnya:
- FlareSolverr terlihat sukses
- tetapi profile Chrome asli masih mentok di `Just a moment...`

## Tambahan fitur pada patch ini

### 1. `userDataDir`
Memaksa browser internal FlareSolverr memakai **folder profile Chrome asli**.

Praktisnya:
- FlareSolverr tidak lagi solve di profile sementara yang terpisah
- dia solve langsung di profile target yang memang ingin dipulihkan

### 2. `browserArgs`
Mengizinkan argumen Chrome tambahan per request/session.

Kegunaan:
- eksperimen fingerprint
- proxy flag tertentu
- opsi browser tambahan tanpa edit source tiap kali

### 3. `browserExecutablePath`
Mengizinkan override path Chrome/Chromium.

Kegunaan:
- host yang punya binary non-standar
- eksperimen pakai build browser tertentu

### 4. `userAgent` override lokal
Mengizinkan request/session memaksa UA tertentu pada browser yang diluncurkan FlareSolverr.

Catatan:
- ini patch lokal eksperimen
- upstream FlareSolverr v2 tidak memakai request `userAgent` sebagai perilaku resmi default

### 5. Session menyimpan launch settings
Mode `sessions.create` sekarang menyimpan setting penting ini:
- proxy
- user agent
- user data dir
- browser args
- browser executable path

Artinya session berikutnya tetap konsisten dan tidak balik ke launch default.

### 6. `request.post` tidak lagi merusak nilai form yang mengandung `&`
Patch lokal sekarang mengganti parsing form-post dari split manual berbasis `&` menjadi parser query-string yang benar.

Masalah lama:
- `postData` dipecah pakai `split('&')`
- nilai seperti password `Yuda&4321` ikut kepotong di tengah
- hasil submit bisa tampak seperti kredensial salah padahal value sudah rusak sebelum terkirim

Perbaikan:
- pakai parser query-string yang menjaga encoded delimiter tetap utuh
- isi field HTML diisi dengan nilai mentah yang sudah di-escape untuk atribut, bukan di-URL-encode ulang
- browser yang melakukan URL-encoding saat submit, jadi value asli tetap aman

### 7. pelaporan `userAgent` session sekarang pakai UA browser aktif, bukan cache startup
Masalah lama:
- helper session bisa diluncurkan dengan `userAgent` override yang benar
- tetapi field hasil `solution.userAgent` masih kadang menampilkan UA desktop dari cache startup service
- ini bikin diagnosis salah, seolah override tidak terpakai

Perbaikan:
- kalau `get_user_agent()` dipanggil dengan driver aktif, baca `navigator.userAgent` langsung dari driver itu
- cache global hanya dipakai untuk path bootstrap tanpa driver aktif
- hasil request sekarang lebih cocok dengan UA browser yang benar-benar dipakai session

### 8. `request.dom_submit` untuk submit form hidup dari halaman aktif
Masalah lama:
- `request.post` helper membangun form sintetis berbasis `data:` lalu submit dari sana
- untuk target seperti ClaimCoin faucet, lane ini bikin submit tidak benar-benar terjadi dari halaman form aktif
- itu menyulitkan debugging, dan untuk alur tertentu lebih aman submit langsung dari DOM halaman yang sudah terbuka

Perbaikan:
- ditambah command baru `request.dom_submit`
- command ini mengisi field yang diminta ke form yang sudah ada di halaman aktif lalu submit dari DOM halaman itu sendiri
- cocok untuk kasus seperti ClaimCoin `/faucet/verify`, saat login/clearance sudah terbukti harus tetap di helper session yang sama

### 9. `debuggerAddress` untuk attach ke browser yang sudah hidup
Masalah lama:
- patched FlareSolverr bisa pakai `userDataDir`, tetapi tetap meluncurkan browser baru dari nol
- untuk lane yang clearance-nya hanya stabil di browser yang sudah terlanjur hangat, itu tidak cukup
- sekadar memberi port debug ke `undetected_chromedriver` lokal tidak otomatis berarti attach ke browser existing

Perbaikan:
- ditambah parameter request/session `debuggerAddress` dengan format `127.0.0.1:9222`
- saat field ini diisi, FlareSolverr memakai Selenium attach mode ke browser Chrome yang **sudah hidup** melalui remote debugging port
- mode ini cocok untuk kasus `warm browser handoff`, yaitu browser dibuka dan dipanaskan oleh lane lain lebih dulu lalu FlareSolverr hanya mengambil alih kontrol

Catatan penting:
- mode attach **mengabaikan** `proxy`, `userAgent`, `userDataDir`, dan `browserArgs` sebagai launch option, karena browser target sudah telanjur berjalan
- artinya browser yang ingin di-attach harus memang sudah diluncurkan dengan profile, proxy, dan argumen yang benar dari awal

### 10. `keepAttachedBrowserAlive` agar destroy tidak membunuh browser eksternal
Masalah lama:
- kalau FlareSolverr attach ke browser external lalu memanggil `driver.quit()`, browser itu ikut mati
- itu merusak use-case `lanjut dari browser yang sudah kebuka`

Perbaikan:
- ditambah parameter `keepAttachedBrowserAlive`
- default-nya `true` untuk attach mode
- saat session / request selesai, FlareSolverr hanya menghentikan service chromedriver attach-nya dan membiarkan browser target tetap hidup

### 11. `reuseCurrentPage` untuk lanjut solve dari halaman yang sedang terbuka
Masalah lama:
- meski FlareSolverr sudah bisa attach ke browser existing, `request.get` tetap memanggil `driver.get(req.url)` lagi
- itu membuat lane `warm browser handoff` berisiko reset state atau mengulang request yang sebenarnya sudah sampai boundary penting

Perbaikan:
- ditambah parameter `reuseCurrentPage`
- untuk request `GET`, kalau flag ini `true`, FlareSolverr tidak menavigasi ulang ke `req.url`
- dia langsung menjalankan deteksi challenge, wait loop, click helper, screenshot, dan response capture dari halaman yang **sudah aktif** di browser attach
- ini cocok untuk kasus seperti `xut -> gamescrate`, saat browser custom sudah berhasil sampai gate akhir dan FlareSolverr hanya perlu mengambil alih dari situ

### 12. Fallback klik widget Cloudflare `#GQTnq7`
Temuan lapangan:
- pada lane `gamescrate`, checkbox/iframe Turnstile normal tidak selalu terekspos ke selector Selenium biasa
- tetapi klik pointer di sisi kiri hotspot widget `#GQTnq7` terbukti mengubah state challenge

Perbaikan:
- helper `click_verify()` sekarang punya fallback click ke hotspot kiri `#GQTnq7`
- ini melengkapi strategi lama `TAB + SPACE` dan pencarian tombol biasa

## Hasil yang sudah terbukti

### Gagal
- cookie + UA transplant saja tidak cukup untuk beberapa profile ChatGPT

### Berhasil
- patched FlareSolverr + `userDataDir` profile asli berhasil membersihkan Cloudflare pada beberapa profile ChatGPT
- untuk profile yang lebih bandel, mode session dua langkah juga berhasil:
  - `sessions.create`
  - `request.get`
  - `request.get` lagi sampai `Challenge not detected!`
  - `sessions.destroy`

## Status

Ini masih **patch eksperimen lokal**, belum upstream.
