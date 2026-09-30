# Prompt: Setup Lengkap 9Router + Hermes + Bot (sekali jalan)

Copy-paste seluruh blok di bawah ini sebagai satu prompt. Lampirkan `bridge.py` (v5.1)
bersamaan dengan prompt ini.

---

```
Setup lengkap "9Router + Hermes + bot" di VPS ini, end-to-end dalam satu sesi:

1. Install 9Router via npm (prefix persisten ~/.npm-global), jalankan sebagai
   systemd service (bind 127.0.0.1:20128), verifikasi health + ambil API key.
2. Deploy bridge.py (file terlampir, v5.1) sebagai systemd service
   (bind 127.0.0.1:8765, antrean file), generate user key + worker key,
   verifikasi /health dan /v1/models.
3. Registrasi via API 9Router: provider node "Muse" (openai-compatible →
   http://127.0.0.1:8765/v1), connection, dan combo model publik "muse".
4. Backup semua key (API key 9Router, bridge user key, bridge worker key) ke file
   terpisah permission 600 — jangan pernah tampilkan nilainya di chat.
5. Pasang worker: scheduled task tiap 1 menit (poll /muse/pending, jawab sebagai
   Muse, POST /muse/answer; diam kecuali bridge tak terjangkau / error auth)
   + hook 10 detik yang memantau folder antrean dan langsung membangunkan worker
   begitu ada job baru.
6. Install Hermes Agent, custom provider → http://127.0.0.1:20128/v1
   (API key via env di ~/.hermes/.env, permission 600), model default = "muse".
7. Uji end-to-end dan laporkan waktu tiap uji:
   (a) POST /v1/chat/completions dengan model "muse" via 9Router,
   (b) hermes -z "apakah kamu terhubung via 9Router ke provider Muse?"
8. Setelah semua hijau, tanyakan ke saya: bot mau ke WhatsApp, Telegram, atau
   Discord? Lalu setup Hermes gateway sesuai pilihan — minta token bot dan
   user ID saya, batasi akses hanya ke user ID tersebut.

Laporkan ringkas: lokasi service/config/backup, hasil tiap uji beserta angkanya,
tanpa menampilkan secret apa pun.
```

---

## Yang perlu disiapkan sebelum menjalankan prompt

| Bahan | Dari siapa | Keterangan |
|-------|-------------|------------|
| `bridge.py` (v5.1) | Asisten / repo ini | Lampirkan bersama prompt |
| Token bot | Kamu | Diberikan saat asisten bertanya di langkah 8 (BotFather untuk Telegram, Developer Portal untuk Discord, dst.) |
| User ID yang diizinkan | Kamu | Agar bot hanya melayani kamu |

Worker yang menjawab (Muse) tidak butuh file tambahan — berjalan sebagai
scheduled task di sisi asisten.
