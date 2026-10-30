# Katalog Skill

Dokumen ini merangkum 24 skill yang dipakai di OpenCode, Claude Code, Kiro (CLI dan IDE), Zed (lewat agen ACP), serta Cursor, Gemini, Hermes, dan Pi (khusus keluarga antislop).

Terakhir diperbarui: 30 Oktober 2026.

## Daftar isi

1. [Lokasi dan sinkronisasi](#lokasi-dan-sinkronisasi)
2. [Cara memakai skill](#cara-memakai-skill)
3. [Diskusi dan perencanaan](#1-diskusi-dan-perencanaan)
4. [Menulis dan mengubah kode](#2-menulis-dan-mengubah-kode)
5. [Review, keamanan, performa, git](#3-review-keamanan-performa-git)
6. [UI dan desain (antislop)](#4-ui-dan-desain-antislop)
7. [Menulis skill](#5-menulis-skill)
8. [Alur kerja yang disarankan](#alur-kerja-yang-disarankan)
9. [Pemeliharaan](#pemeliharaan)

---

## Lokasi dan sinkronisasi

Sumber utama: `~/.agents/skills`. Edit hanya di sana.

| Lokasi | Dipakai oleh | Cara |
|---|---|---|
| `~/.agents/skills` | OpenCode (dan sumber utama) | sumber |
| `~/.claude/skills` | Claude Code, Zed lewat `claude-acp` | symlink ke sumber |
| `~/.kiro/skills` | Kiro CLI, Kiro IDE, Zed lewat `kiro-cli acp` | salinan (rsync) |
| `~/.cursor/skills`, `~/.gemini/skills`, `~/.gemini/config/skills`, `~/.hermes/skills`, `~/.pi/agent/skills` | Cursor, Gemini, Hermes, Pi | salinan, hanya keluarga antislop |

Setelah mengedit skill:

```bash
~/.agents/sync-skills.sh
```

Catatan:
- Zed sendiri tidak memuat skill. Ia memakai skill dari agen di baliknya.
- Custom agent di Kiro tidak memuat skill otomatis. Tambahkan ke `resources`: `"skill://~/.kiro/skills/*/SKILL.md"`.
- Skill di dalam repo (`.kiro/skills/`, `.claude/skills/`) bisa menimpa skill global dengan nama sama. Belum diperiksa untuk repo-repomu.

## Cara memakai skill

1. **Otomatis**: agen mencocokkan prompt dengan `description` tiap skill (berdasarkan makna, bukan kata persis). Semua description sudah punya kata kunci Indonesia.
2. **Manual (paling pasti)**: sebut nama skill ("pakai skill debugging"), atau ketik `/nama-skill` di Kiro.
3. Agen membalas dalam bahasa yang kamu pakai. Kode, nama fungsi, dan label tetap bahasa Inggris.

Aturan umum yang berlaku di banyak skill:
- Commit, push, tag, merge, dan menghapus hanya dilakukan kalau diminta.
- Perintah build dan tes diambil dari repo, tidak diasumsikan `npm test`.
- Nilai rahasia (API key, token) tidak dicetak ke output.
- Hal yang tidak bisa diverifikasi harus dinyatakan, bukan diklaim lolos.

---

## 1. Diskusi dan perencanaan

### interview-me

**Tujuan:** menggali apa yang sebenarnya diinginkan, dan menjaga diskusi agar tidak berputar.

**Kapan dipakai:** permintaan kurang jelas ("buatkan X" tanpa konteks), brainstorming terbuka, diskusi yang berulang atau melenceng.

**Cara kerja:**
- **Step 0:** baca kode dan konteks dulu; jangan tanya yang sudah dijawab proyek. Kalau semua sudah jelas, langsung kerjakan.
- Hipotesis satu kalimat dengan angka keyakinan, lalu satu pertanyaan per giliran dengan tebakan.
- **Decision Log** tampil tiap sekitar 4 pertanyaan: `DECIDED` / `OPEN` / `PARKED`. Keputusan yang sudah ada tidak dibuka lagi tanpa dicatat.
- **Parking Lot** untuk topik sampingan.
- **Sinyal berputar** (pengulangan, keputusan dibalik dua kali, jawaban makin kabur): berhenti, paksa pilihan antara dua opsi.
- Anggaran pertanyaan: cek-ulang setelah 5, maksimal 8 per topik.
- **Mode eksplorasi** untuk brainstorming tanpa hasil yang dibangun: Goal / Decided / Still open / Next step.
- Penutup: restate (Outcome, User, Why now, Success, Constraint, Out of scope) yang harus dijawab "ya" secara eksplisit.
- **Step 6:** 2–3 pendekatan dengan trade-off dan rekomendasi, user memilih; lalu ukuran pekerjaan (kecil atau proyek). Bisa menawarkan spike buang-pakai.
- Opt-out ringan bila skill terpicu otomatis; default yang dipilih agen diucapkan dan dicatat sebagai keputusan agen.

**Contoh prompt:**
```
Pakai interview-me, saya mau bahas arsitektur notifikasi.
```
```
Kita muter-muter terus, pakai interview-me buat klarifikasi tujuan kita.
```
```
Saya mau bikin fitur export data — belum tahu mau mulai dari mana, bantu gali dulu.
```
```
Interview saya dulu sebelum kamu mulai nulis kode apapun.
```

**Jangan dipakai untuk:** instruksi yang sudah jelas ("ganti nama variabel ini"), pertanyaan informasi, konteks non-interaktif.

---

### doubt-driven-development

**Tujuan:** menguji keputusan sebelum menjadi pegangan.

**Dua bobot:**
- **Ringan** (keputusan non-kode): cek diri tiga pertanyaan, asumsi apa yang tidak kamu nyatakan, cara paling mungkin salah, bukti apa yang mengubah pikiran.
- **Penuh** (kode, tidak bisa dibatalkan, keamanan): siklus 5 langkah dengan reviewer berkonteks baru (OpenCode: tool `task` dengan agent `general`), hanya diberi artefak dan kontrak, bukan klaim.

**Detail:** maksimal 3 siklus lalu eskalasi ke kamu. Penawaran review lintas-model hanya untuk keputusan berisiko tinggi, sekali per keputusan.

**Contoh prompt:**
```
Ragukan keputusan ini sebelum kita lanjut.
```
```
Uji asumsi desain database ini — apa yang bisa salah?
```
```
Apakah pendekatan ini aman untuk production? Coba dipertanyakan dulu.
```
```
Kita mau deploy migrasi yang tidak bisa di-rollback — review dulu dengan konteks baru.
```

---

### writing-plans

**Tujuan:** mengubah maksud yang sudah dikonfirmasi menjadi rencana checklist per task.

**Isi utama:**
- Input: restate dan pendekatan dari `interview-me`, atau file spec.
- Disimpan di `docs/plans/YYYY-MM-DD-<fitur>.md`, **tidak di-commit**.
- Tiap task: file yang disentuh, Interfaces (signature yang dipakai/dihasilkan), langkah RED-GREEN dengan perintah dan `Expected:`, lalu **Checkpoint** (jalankan tes, centang, usulan pesan commit).
- Bagian Global Constraints, Review Focus (kasus yang tidak dites tapi berisiko), dan Progress kosong untuk `executing-plans`.
- Self-review: cakupan spec, tiap langkah tidak ambigu, konsistensi tipe, proporsi.

**Contoh prompt:**
```
Buat rencana implementasi untuk fitur notifikasi ini.
```
```
Sudah tahu mau bikin apa — sekarang tulis plan checklist-nya di docs/plans/.
```
```
Pecah fitur multi-tenant ini jadi task-task kecil yang bisa dikerjakan per slice.
```
```
Plan dulu sebelum ngoding, saya mau review rencana ini sebelum dijalankan.
```

**Jangan dipakai untuk:** perubahan kecil satu-dua file (langsung `incremental-implementation`).

---

### context-engineering

**Tujuan:** menjaga konteks agen tetap tajam.

**Isi utama:**
- Hierarki konteks: file aturan, spec, file sumber, output error, riwayat percakapan.
- **Template ringkasan sesi:** `GOAL / DECIDED / REJECTED / OPEN / PARKED / STATE / NEXT`.
- **Kapan reset:** pengulangan, keputusan dibalik, melenceng, agen lupa batasan.
- Panduan sesi diskusi, dan catatan `AGENTS.md` dan `opencode.json` untuk OpenCode.

**Contoh prompt:**
```
Sesi ini sudah panjang, ringkas dan siapkan konteks untuk sesi baru.
```
```
Jawabanmu mulai ngawur dan salah pola — reset konteks dulu.
```
```
Setup AGENTS.md untuk proyek ini supaya agen selalu tahu konvensi kita.
```
```
Kita mau ganti topik dari auth ke fitur export — atur ulang konteks.
```

---

## 2. Menulis dan mengubah kode

### incremental-implementation

**Tujuan:** fitur multi-file dikerjakan per slice kecil yang bisa diuji.

**Isi utama:**
- Siklus slice: implement, tes, verifikasi, titik simpan (commit hanya jika diminta).
- Strategi: vertikal (utama), contract-first, risk-first.
- Contoh slice untuk Next.js + Supabase, Firebase, Go, Android, AWS.
- Aturan: simpel dulu, disiplin cakupan (catat "NOTICED BUT NOT TOUCHING"), satu hal per langkah, selalu bisa di-compile, feature flag, mudah di-rollback (migrasi dipisah dari perubahan destruktif).

**Contoh prompt:**
```
Kerjakan fitur CRUD ini bertahap — mulai dari migrasi dan RLS dulu.
```
```
Fitur ini besar, pecah jadi slice-slice kecil yang bisa diuji satu per satu.
```
```
Implementasi per langkah ya — tiap step harus bisa di-compile dan tes dulu sebelum lanjut.
```
```
Mulai dari slice paling berisiko dulu (integrasi payment gateway), baru yang lain.
```
```
Build fitur notifikasi ini: Slice 1 = schema DB, Slice 2 = API endpoint, Slice 3 = UI.
```

**Jangan dipakai untuk:** perubahan satu fungsi.

---

### executing-plans

**Tujuan:** menjalankan rencana dari `writing-plans` task demi task, tanpa commit.

**Isi utama:**
- Progres = checkbox yang dicentang + baris di bagian Progress (bisa dilanjutkan setelah compaction atau di sesi baru).
- Baseline dicatat di awal (`HEAD` + `git status`). Kalau ada perubahan belum di-commit, agen memberi tahu dan menyarankan kamu commit/stash dulu.
- Tiap langkah: TDD, bandingkan dengan `Expected:`. Kode salah: `debugging-and-error-recovery`. Rencana salah: catat sebagai **Ruling**.
- Berjalan terus tanpa tanya "lanjut?", kecuali 5 hal: destruktif, keamanan, perubahan git, efek di luar repo, rencana rusak total.
- Review akhir oleh subagent berkonteks bersih dengan `code-review-and-quality`; satu putaran perbaikan dengan tes RED→GREEN.
- Pesan akhir: perubahan, daftar Ruling, yang ditunda, verifikasi, **usulan pesan commit**.

**Contoh prompt:**
```
Jalankan rencana di docs/plans/2026-10-30-notifikasi.md.
```
```
Lanjutkan plan kemarin — kita sudah sampai Task 3.
```
```
Eksekusi plan ini task demi task, stop di Task 5 dulu buat saya review.
```
```
Kerjakan plan-nya, kalau ada yang tidak bisa jalan catat sebagai Ruling jangan stop.
```

---

### dispatching-parallel-agents

**Tujuan:** beberapa masalah independen dikerjakan subagent secara paralel.

**Aturan:** satu subagent per domain masalah, prompt mandiri dan spesifik, set file yang tidak tumpang tindih, tanpa commit atau perubahan git. Setelah selesai: cek konflik di `git diff`, jalankan suite penuh.

**Contoh prompt:**
```
Tiga file tes ini gagal karena sebab berbeda — kerjakan paralel.
```
```
Ada 4 bug independen di modul berbeda, bagi ke beberapa agen biar selesai lebih cepat.
```
```
Refactor auth.ts, api.ts, dan utils.ts secara bersamaan — tidak ada dependensi antar file.
```
```
Kerjakan paralel: satu agen fix bug di backend, satu lagi perbaiki UI yang rusak.
```

**Jangan dipakai untuk:** kegagalan yang saling terkait atau debugging yang belum jelas sumbernya.

---

### test-driven-development

**Tujuan:** tes sebelum kode, reproduksi bug dengan tes.

**Isi utama:**
- Siklus RED / GREEN / REFACTOR dan pola **Prove-It** untuk bug.
- **Name the break** sebelum menulis tes; expected pakai literal; tanpa change detector; jangan assert ke mock.
- **Mutation check** sebelum selesai; laporkan semua tes gagal di suite.
- Piramida tes (unit, integrasi, E2E) dan panduan memilih level.
- **Playbook per stack:**
  - Next.js/React: Vitest atau Jest, React Testing Library, MSW, Playwright. Server Component async sulit diuji unit.
  - Firebase: emulator, `@firebase/rules-unit-testing`, tes kasus yang harus ditolak.
  - Supabase: `supabase test db` (pgTAP), tes RLS dengan beberapa role.
  - AWS: logika handler murni, mock SDK v3, LocalStack atau SAM.
  - Go: table-driven, `-race`, `httptest`, testcontainers, fuzz.
  - Kotlin/Android: JUnit, MockK, `runTest`, Turbine, Compose UI test, Room in-memory.
- Dilonggarkan untuk konfigurasi, spike, dan UI murni. Tes karakterisasi untuk kode lama.

**Contoh prompt:**
```
Buat unit test untuk fungsi calculateDiscount ini.
```
```
Ada bug di fungsi parseDate — reproduksi dulu dengan tes yang gagal, baru fix.
```
```
Tulis tes RLS untuk tabel orders — cek akses owner vs user lain.
```
```
Tambahkan integration test untuk endpoint POST /api/users ini.
```
```
Saya mau ubah perilaku fungsi ini — buat tes dulu sebelum refactor.
```

---

### source-driven-development

**Tujuan:** kode yang sensitif versi mengikuti dokumentasi resmi, bukan ingatan.

**Isi utama:**
- Deteksi versi dari lockfile (juga `build.gradle`, `libs.versions.toml`, `pubspec.yaml`, `firebase.json`, `supabase/config.toml`).
- Sumber resmi per stack (nextjs.org, react.dev, firebase, supabase, AWS, go.dev, developer.android.com, kotlinlang.org). Docs MCP resmi (misalnya Context7) dihitung resmi.
- Konflik dokumentasi dengan kode yang ada ditunjukkan, tidak dipilih diam-diam.
- Sitasi di balasan; di kode hanya untuk workaround khusus versi.
- Tanpa akses web: tandai `UNVERIFIED`.

**Contoh prompt:**
```
Cek dokumentasi resmi Next.js 15 untuk cache behavior sebelum nulis ini.
```
```
Pakai source-driven-development — saya tidak mau kamu nulis dari ingatan untuk Firebase SDK v10.
```
```
Verifikasi dulu API yang benar untuk React Server Components sebelum implementasi.
```
```
Apakah cara ini masih valid di Supabase versi yang kita pakai? Cek docs-nya dulu.
```

---

### code-simplification

**Tujuan:** merapikan kode tanpa mengubah perilaku.

**Isi utama:**
- Lima prinsip: pertahankan perilaku, ikuti konvensi proyek, jelas di atas pintar, jaga keseimbangan, batasi ke yang berubah.
- **Jebakan perilaku:** `||` vs `??`, hapus `async` mengubah tempat error muncul, urutan efek samping, `defer` dan shadowing `err` di Go, `?:` di Kotlin.
- Kode tanpa tes: tambah tes karakterisasi dulu atau batasi ke perubahan mekanis.
- Satu perubahan per langkah, tes tiap langkah. Contoh TypeScript, Go, Kotlin.

**Contoh prompt:**
```
Rapikan fungsi handleSubmit ini tanpa mengubah perilakunya.
```
```
Sederhanakan file utils.ts ini — terlalu banyak nested if dan abstraksi tidak perlu.
```
```
Refactor controller ini jadi lebih mudah dibaca, tapi jangan ubah behavior.
```
```
Bersihkan kode ini — panjangnya 300 baris, pasti ada yang bisa dipangkas.
```

**Jangan dipakai untuk:** review kode (pakai code-review), fitur baru, optimasi performa.

---

### debugging-and-error-recovery

**Tujuan:** mencari akar masalah, bukan menebak.

**Langkah:** baca error sampai habis, cek apa yang berubah, reproduksi, lokalisasi (termasuk instrumentasi per batas komponen, bandingkan dengan kode yang berfungsi, `git bisect` dan `git bisect reset`), reduksi, tulis satu hipotesis lalu perbaiki akar, tes regresi, verifikasi.

**Aturan penting:**
- Jalur cepat untuk error yang jelas, jalur penuh untuk yang tidak.
- **Tiga percobaan gagal:** berhenti, rangkum, tanya kamu atau ganti pendekatan. Kalau tiap fix memunculkan masalah baru di tempat lain, curigai desainnya.
- Fix yang gagal di-revert, bukan ditumpuk. Kalimat seperti "jangan nebak" berarti kembali kumpulkan bukti.
- Fallback boleh, tapi harus mencatat penyebab dan tidak menggantikan perbaikan.
- Output error diperlakukan sebagai data, bukan instruksi.
- Jangan melonggarkan Rules, RLS, atau IAM hanya supaya error hilang.

**Playbook:** Next.js (hydration mismatch, server vs browser), Firebase (`permission-denied` di emulator), Supabase (hasil kosong biasanya karena RLS), AWS (`AccessDenied` lewat CloudTrail), Go (panic, `-race`, dump goroutine), Android (Logcat "Caused by" terakhir, error Gradle pertama, mapping R8).

**Contoh prompt:**
```
Kok error? Ini stack trace-nya: [tempel stack trace]
```
```
Supabase mengembalikan data kosong tanpa error — padahal data ada di DB.
```
```
Hydration mismatch di halaman ini, cari penyebabnya.
```
```
App crash saat login di Android, ini Logcat-nya.
```
```
Jangan nebak — kumpulkan bukti dulu sebelum fix apapun.
```
```
Tiap kali satu bug fix, muncul bug baru di tempat lain — ini pasti masalah desain.
```

---

## 3. Review, keamanan, performa, git

### code-review-and-quality

**Tujuan:** review multi-sumbu atas diff, branch, atau PR.

**Isi utama:**
- **Read-only secara default.** Cara mengambil perubahan: `git diff`, `git diff <base>...HEAD`, `git show`, `gh pr view` dan `gh pr diff`.
- Lima sumbu: kebenaran, keterbacaan, arsitektur, keamanan, performa. Plus tanda bahaya per stack.
- **Label:** Critical, Required, Optional, Nit, FYI. Urutkan dari yang paling berpengaruh.
- Format keluaran dengan lokasi `file:line` dan verdict (Approve, Approve with comments, Request changes).
- Remedi struktural, ukuran perubahan, kebersihan kode mati (tanya dulu sebelum menghapus), review dependensi.
- Bagian **Set aside** (hal yang tidak dinilai beserta alasan), spek yang diam dinilai dari harapan pengguna wajar, tidak mengubah state git (pakai `git worktree` di `/tmp` untuk revisi lain).

**Contoh prompt:**
```
Review perubahan di branch feature/payment ini.
```
```
Cek PR #42 — fokus ke keamanan dan arsitektur.
```
```
Review diff ini sebelum saya merge ke main.
```
```
Ada yang aneh di kode ini — bisa di-review dengan 5 sumbu standar?
```
```
Review perubahan sejak commit abc123 sampai HEAD.
```

---

### receiving-code-review

**Tujuan:** menanggapi review (dari kamu, reviewer lain, atau bot) dengan verifikasi, bukan asal setuju.

**Isi utama:** baca semua dulu, pahami, cek ke codebase, baru kerjakan satu per satu dengan tes. Tanpa "You're absolutely right!". Tanya kalau ada poin yang tidak jelas sebelum mengerjakan apa pun. Tolak dengan alasan teknis bila saran salah atau YAGNI.

**Contoh prompt:**
```
Tanggapi komentar review di PR ini — jangan langsung setuju, cek dulu ke codebase.
```
```
Perbaiki sesuai review ini, tapi kalau ada yang kamu rasa salah, tolak dengan alasan.
```
```
Ada feedback dari reviewer — baca dulu semua, baru kerjakan satu per satu.
```
```
Bot review bilang ada security issue di sini — verifikasi dulu bener apa tidak.
```

---

### verification-before-completion

**Tujuan:** tidak ada klaim "selesai / fix / lulus" tanpa bukti.

**Aturan:** sebelum klaim, jalankan perintah yang membuktikannya di sesi itu, baca output-nya, baru nyatakan hasil sesuai bukti. Berlaku juga untuk laporan subagent.

**Contoh prompt:**
```
Cek dulu sebelum bilang selesai — jalankan tesnya dan tunjukkan outputnya.
```
```
Jangan klaim fix tanpa bukti — verifikasi dulu.
```
```
Sebelum bilang tes lulus, jalankan dan paste hasilnya.
```
```
[biasanya terpicu otomatis setiap agen hendak mengklaim pekerjaan selesai]
```

---

### security-and-hardening

**Tujuan:** kode yang aman untuk stack-mu.

**Aturan agen:** jangan cetak nilai rahasia; laporkan dan sarankan rotasi.

**Cakupan:**
- Model ancaman (STRIDE) dan batas Always / Ask first / Never.
- **Next.js/React:** Server Action dan Route Handler adalah endpoint publik, middleware bukan satu-satunya penjaga, batas `NEXT_PUBLIC_`, XSS, open redirect, SSRF.
- **Firebase:** Rules default menolak, role lewat custom claims, App Check, tes Rules di emulator.
- **Supabase:** RLS di semua tabel, `service_role` hanya server, jangan pakai `user_metadata` di policy, `security_invoker`, `getUser()`.
- **AWS:** IAM least privilege, S3 Block Public Access, presigned URL, Secrets Manager, IMDSv2.
- **Go:** timeout server, `MaxBytesReader`, `html/template`, JWT, `govulncheck`.
- **Android:** tanpa rahasia di APK, `usesCleartextTraffic`, `android:exported`, WebView, Keystore, `BiometricPrompt`.
- Rantai pasok dependensi, rate limiting, manajemen rahasia, fitur LLM.

**Contoh prompt:**
```
Cek keamanan Firestore Rules ini — apakah ada yang bisa dibobol?
```
```
Apakah RLS tabel orders ini sudah aman? User lain tidak boleh bisa baca data orang lain.
```
```
Hardening endpoint ini — ini menerima input dari user dan langsung ke DB.
```
```
Review IAM policy ini — apakah sudah least privilege?
```
```
Ada API key yang mungkin tertanam di kode Android — cek dan perbaiki.
```

---

### performance-optimization

**Tujuan:** ukur dulu, baru perbaiki.

**Alur:** ukur, identifikasi, perbaiki satu hal, verifikasi (simpan jika membaik melewati noise, **revert jika netral**), jaga dengan budget.

**Cakupan:** Next.js/React (Server Components, waterfall, `next/image`, cache per versi), Firestore (biaya per baca, listener, paginasi cursor), Supabase/Postgres (`EXPLAIN ANALYZE`, indeks termasuk kolom RLS, keyset pagination), AWS (Lambda, RDS Proxy, CloudFront), Go (`pprof`, benchmark, `benchstat`, pool DB), Android (main thread, Compose, Baseline Profiles, Macrobenchmark).

**Contoh prompt:**
```
Halaman dashboard ini lambat — cari penyebabnya, ukur dulu sebelum fix apapun.
```
```
Query ini lambat di production — analisis dengan EXPLAIN ANALYZE.
```
```
App Android ini lag saat scroll — profil dulu dengan metode yang benar.
```
```
Bundle size Next.js-ku 4MB, kurangi tanpa pecah fitur.
```
```
Firestore read-nya mahal banget — audit dan optimalkan query dan listener.
```

---

### git-workflow-and-versioning

**Tujuan:** praktik git yang aman dan rapi.

**Aturan keselamatan:** commit, push, dan tag hanya jika diminta. `reset --hard`, `clean -fd`, force push, dan `--no-verify` butuh persetujuan. Ikuti gaya `git log` repo.

**Isi:** commit atomik dan pesan yang menjelaskan alasan, cabang pendek, resolusi merge conflict, tabel undo aman, kebersihan sebelum commit, `.gitignore` per stack, `git bisect`, SemVer, tag, changelog.

**Contoh prompt:**
```
Commit perubahan ini dengan pesan yang menjelaskan alasannya.
```
```
Ada merge conflict di file ini — bantu selesaikan.
```
```
Siapkan changelog dan tag untuk release v1.2.0.
```
```
Saya mau batalkan commit terakhir tapi tidak mau hilangkan perubahannya.
```
```
Buat branch dari main, naming convention sesuai repo ini.
```

---

## 4. UI dan desain (antislop)

Hanya termuat saat membangun, restyling, atau mereview tampilan. Tidak aktif untuk brainstorming, backend, atau obrolan umum.

### antislop (inti)

**Tujuan:** filter agar UI tidak terlihat buatan AI, tanpa menghasilkan tampilan hambar.

**Mode:** **Build** (default, tanpa pertanyaan) dan **Audit** (daftar temuan bernomor, tidak mengubah apa pun sebelum kamu memilih nomor).

**Isi:**
- **Arah desain (R-37):** muat `DESIGN.md` atau arahan brand; tanpa itu, tandai "draft without direction" dengan dial 1/1/1.
- **Dial:** ENERGY, RHYTHM, MOTION (1 sampai 3), dideklarasikan lewat satu baris "Reading this as...".
- **Hard Gate:** tanpa em dash, mobile bekerja, tanpa data atau testimoni palsu, konfirmasi aset (R-23), kontras WCAG AA, kontrol berfungsi, state kosong/loading/error, keyboard dan fokus, tema berfungsi, verifikasi sebelum menyerahkan, tanpa klaim fiktif.
- **Strong defaults:** navigasi hanya yang ada (R-24), FAQ tidak generik (R-28), tema dengan alasan (R-21), alasan satu baris (R-31).
- **Purpose-Gate:** gradien, ikon, tipografi, grid, badge, glass, bayangan, glow, kartu, animasi, ilustrasi boleh dengan alasan tertulis dan batas dosis.
- **Quality Locks:** layout dari konten, radius, CTA spesifik, tanpa buzzword, palet 2 sampai 3 warna + 1 aksen, tidak meniru produk populer.
- **Delivery Gate:** tujuh kelompok, laporan PASS/FAIL dengan bukti dan pengakuan atas yang tidak bisa diverifikasi.

**Contoh prompt:**
```
Buatkan landing page untuk produk manajemen tugas ini.
```
```
Audit tampilan dashboard ini — tunjukkan apa yang terlihat seperti buatan AI.
```
```
Desain ulang halaman pricing ini, brand kita: bold, teknikal, bukan startup generik.
```
```
Ini tampilan yang ada [screenshot] — rapikan tampilan tanpa ubah struktur informasi.
```

---

### antislop-ui

Pola visual dan struktural beserta perbaikannya: warna, layout, dekorasi (ikon, badge, emoji, panah), struktur, **App & Dashboard** (dashboard default, angka karangan, feed palsu, grafik tanpa pertanyaan, kolom generik, data isian palsu, state kosong yang tidak informatif), dan motion.

**Contoh prompt:**
```
Desain ulang halaman ini supaya tidak generik — sekarang kelihatan seperti template AI.
```
```
Warna dan layout-nya terasa generic SaaS banget — perbaiki dengan identitas yang lebih kuat.
```
```
Dashboard ini isinya angka-angka placeholder — buat state kosong yang informatif.
```
```
Terlalu banyak gradien dan glow — pangkas dan kasih alasan yang tersisa.
```

---

### antislop-copywriting

Teks UI dan marketing yang tidak terdengar seperti AI: kosakata kosong, inflasi makna, klaim tanpa bukti, pola ritme (trio paksa, paralelisme negatif, staccato), larangan em dash dengan protokol voice sample, kebersihan markdown, dan loop draft, audit, final.

**Referensi Indonesia:** `antislop-copywriting/references/indonesian.md`, berisi register `Anda` atau `kamu`, buzzword dan CTA Indonesia, pola kalimat AI, format `Rp 1.500.000` dan tanggal, serta pesan error dan state kosong.

**Contoh prompt:**
```
Tulis copy bahasa Indonesia untuk halaman harga ini.
```
```
Teks onboarding ini kedengarannya AI banget — tulis ulang dengan suara yang lebih manusiawi.
```
```
Buat 3 variasi tagline untuk produk ini — bukan yang generik.
```
```
Audit copy landing page ini — mana yang buzzword kosong, mana yang substantif?
```

---

### antislop-human

Aksesibilitas: kontras (termasuk pemeriksa `contrast-check.py`), keyboard, fokus, state, zoom, dan keyboard mobile.

**Contoh prompt:**
```
Cek kontras warna dan aksesibilitas form login ini.
```
```
Apakah tombol-tombol ini bisa diakses dengan keyboard? Cek focus state-nya.
```
```
Jalankan contrast-check.py untuk palet warna ini — harus lulus WCAG AA.
```
```
State loading dan error di halaman ini belum ada — tambahkan yang aksesibel.
```

---

### antislop-layoutmobile

Layout responsif: breakpoint, skala dan ukuran, grid, overflow, tap target, navigasi mobile.

**Contoh prompt:**
```
Perbaiki tampilan mobile halaman ini — tombolnya terlalu kecil dan ada yang overflow.
```
```
Layout ini berantakan di layar kecil — fix responsifnya dengan breakpoint yang benar.
```
```
Tap target di list item ini terlalu kecil untuk jari — perbesar sesuai standar mobile.
```
```
Grid ini tidak collapse dengan benar di mobile — perbaiki tanpa ubah tampilan desktop.
```

---

### antislop-code

Hanya merapikan komentar kode yang bernada AI (pembatas dekoratif, mengulang yang jelas, narasi alur). Tidak menyentuh kode.

**Contoh prompt:**
```
Hapus komentar yang tidak perlu di file ini — yang cuma mengulang apa yang sudah jelas dari kode.
```
```
Bersihkan komentar dekoratif di controller ini, jangan ubah kode sama sekali.
```
```
File ini penuh komentar seperti // ==================== SECTION ==================== — rapikan.
```

---

## 5. Menulis skill

### writing-skills

**Tujuan:** membuat atau mengedit skill di `~/.agents/skills`, dan mengujinya.

**Isi utama:** menulis skill sebagai TDD untuk dokumentasi (uji perilaku agen tanpa skill dulu, tulis skill, uji lagi, tutup celah). Referensi: format SKILL.md dan discovery, pengujian dengan subagent, bulletproofing, best practice Anthropic. Setelah selesai: jalankan `sync-skills.sh` dan update katalog ini.

**Contoh prompt:**
```
Buat skill baru untuk workflow deploy ke Supabase.
```
```
Perbaiki description skill interview-me supaya lebih sering terpicu otomatis.
```
```
Skill debugging-and-error-recovery-ku perlu diperbarui — tambahkan playbook untuk Bun runtime.
```
```
Buat skill dari workflow yang baru kita lakukan tadi supaya bisa dipakai lagi.
```

---

## Alur kerja yang disarankan

**Fitur kecil:**
1. `interview-me`: baca konteks dulu, pastikan maksud dan batasan, lalu pilih pendekatan (2–3 opsi beserta trade-off) dan ukuran pekerjaan.
2. `incremental-implementation` + `test-driven-development`: bangun per slice dengan tes.
3. `source-driven-development`: untuk API yang sensitif versi.
4. `security-and-hardening`: bila menyentuh auth, data, Rules, atau RLS.
5. `code-review-and-quality`: review sebelum merge.
6. `git-workflow-and-versioning`: commit dan rilis (jika diminta).

**Fitur besar (proyek):**
1. `interview-me`: maksud, pendekatan, ukuran.
2. `writing-plans`: rencana checklist di `docs/plans/`. Kamu review dulu.
3. `executing-plans`: dikerjakan task demi task dengan TDD, tanpa commit, review akhir berkonteks bersih.
4. Kamu review diff dan commit sendiri (pesan commit sudah diusulkan).

**Menanggapi review:** `receiving-code-review`. **Sebelum bilang selesai:** `verification-before-completion`.

**Bug:** `debugging-and-error-recovery`, lalu `test-driven-development` (tes regresi), lalu `code-review-and-quality`.

**UI:** `antislop` (+ `antislop-ui`, `-copywriting`, `-human`, `-layoutmobile` sesuai kebutuhan).

**Keputusan besar:** `interview-me` lalu `doubt-driven-development`.

**Sesi panjang atau berputar:** `context-engineering` untuk ringkasan dan reset.

---

## Pemeliharaan

- **Aturan format frontmatter (Kiro dan standar Agent Skills):**
  - `name` harus sama dengan nama folder (huruf kecil, angka, tanda hubung, maks 64).
  - `description` maksimal 1024 karakter.
  - Description yang mengandung titik dua harus diberi tanda kutip, kalau tidak YAML rusak.
- **Backup:**
  - `~/skills-backup-2026-10-28.tar.gz`: `.agents`, `.claude`, `.kiro`, `.config/opencode` sebelum perubahan.
  - `~/skills-backup-others-2026-10-28.tar.gz`: Cursor, Gemini, Hermes, Pi.
  - `SKILL.md.bak` per skill untuk `interview-me`, `doubt-driven-development`, `context-engineering`, `antislop`, `antislop-ui`, `antislop-copywriting`.
  - `SKILLS.md.pre-superpowers.bak`: katalog ini sebelum skill superpowers ditambahkan.
  - Skill dari [obra/superpowers](https://github.com/obra/superpowers) (MIT): `writing-plans`, `executing-plans` (keduanya disesuaikan ke model checklist tanpa commit), `verification-before-completion`, `receiving-code-review`, `dispatching-parallel-agents`, `writing-skills` (rujukan disesuaikan). Tidak dipasang: `using-superpowers`, `brainstorming`, `systematic-debugging`, `test-driven-development`, `requesting-code-review`, `subagent-driven-development`, `using-git-worktrees`, `finishing-a-development-branch`, `diagnosing-superpowers`.
  - `SKILL.md.pre-superpowers.bak` di `interview-me`, `debugging-and-error-recovery`, `test-driven-development`, `code-review-and-quality`: versi sebelum ditambah bagian dari skill superpowers yang setara (`brainstorming`, `systematic-debugging`, `test-driven-development`, `requesting-code-review`), 30 Oktober 2026.
- **Keamanan konfigurasi:** token di `~/.config/opencode/opencode.jsonc`, `~/.config/zed/settings.json`, dan `~/.kiro/settings/mcp.json` masih berupa teks biasa. Pindahkan ke environment variable dan rotasi token yang pernah tercetak.
- **Belum diuji di tugas nyata:** skill yang ditulis ulang belum dicoba. Pantau aturan yang terasa terlalu ketat atau longgar, lalu sesuaikan.
- **Perlu dicek manual:** perilaku cache `fetch` Next.js per versi, status deprecation `androidx.security:security-crypto`, dan detail `os.Root` di Go 1.24+.
