Bu klasör n8n 🎼 Music Pipeline - Ana Workflow_V2 ve 🎵 Suno Song Generator_V2 otoamsyonlar için oluşturuldu.
Otomasyonların nasıl çalıştığı aşağıda detaylı yazıyor. 

# 🎼 Music Pipeline — Nasıl Çalışır

**Son güncelleme:** Mayıs 2026  
**Durum:** Production'da aktif ✅

---

## 🧩 Sistem Nedir?

Bir form doldurarak otomatik olarak:
1. Suno AI ile müzik üretir
2. Google Drive'a kaydeder
3. Kapak resmi oluşturur (Gemini AI veya manuel)
4. Cloudinary'e kapağı yükler
5. GitHub Actions üzerinde video render eder (FFmpeg)
6. YouTube'a yükler
7. Tüm süreci Google Sheets'e loglar
8. Telegram'a bildirim gönderir

---

## 🗺️ Bileşenler ve Konumları

| Bileşen | Ne İşe Yarar | Nerede |
|---|---|---|
| **n8n** | Otomasyon motoru | Railway → `n8n-production-eefc3.up.railway.app` |
| **🎼 Ana Workflow** | Tüm süreci yönetir | n8n → Workflows → `🎼 Music Pipeline - Ana Workflow_V2` |
| **🎵 Sub-workflow** | Tek iterasyon müzik üretir | n8n → Workflows → `🎵 Suno Song Generator_V2` |
| **GitHub Actions** | FFmpeg render + YouTube upload | `github.com/kesikoglu/workflows` → `.github/workflows/render.yml` |
| **Google Sheets** | Parçalar ve Oturumlar logu | `n8n_Suno_music` spreadsheet |
| **Google Drive** | MP3 dosyaları | Drive → `suno_music` klasörü (ID: `1W876Os3EH6LMfJp0G6Vk6bEZcv8JTEZZ`) |
| **Cloudinary** | Kapak görseli CDN | Account: `desntuwcc`, klasör: `reals` |
| **KIE AI / Suno** | Müzik üretim API | `api.kie.ai` |
| **Gemini API** | AI kapak görseli üretimi | `generativelanguage.googleapis.com` |

---

## 🔄 Akış — Adım Adım

### AŞAMA 1: Form → GPT → State

```
Form Trigger
    ↓
Build GPT Prompt   (musicType, mood, tempo, extraStyle → OpenAI body hazırlar)
    ↓
Generate AI Content  (GPT-4o → sunoPrompt, sunoStyle, sunoTitle, imagePrompt,
                       youtubeTitle, youtubeDescription, youtubeTags üretir)
    ↓
Init State         (tracks:[], driveFileIds:[], totalDuration:0, sessionId üretir)
    ↓
Prep Session Data  (Sheets için session verisi hazırlar)
    ↓
Log Session to Sheets  (Oturumlar sheet'e satır ekler → durum: not-used)
    ↓
Restore State      (Init State'i geri yükler — Sheets response'u ezmesin diye)
```

### AŞAMA 2: Müzik Üretim Döngüsü

```
Execute Song Generator  ←──────────────────────────┐
    ↓                                               │
  [🎵 Sub-workflow çalışır]                        │
    ↓                                               │
Duration Reached?                                   │
    ├── HAYIR → ────────────────────────────────────┘  (döngü devam)
    └── EVET → Sonraki aşamaya geç
```

**🎵 Sub-workflow içinde ne olur:**
```
Build API Body     (instrumental=true ise vocalGender gönderilmez)
    ↓
Create a Song      (KIE AI → Suno V5_5 model ile 2 parça üretir)
    ↓
Wait 15sn
    ↓
Check the Status   (SUCCESS gelene kadar bekler)
    ↓
Parse Song Data    (sourceStreamAudioUrl alır — Suno orijinal CDN, yüksek kalite)
    ↓
Download Song 0 → Upload Song 0 (Drive) → Sheets'e yaz (Parçalar)
    ↓
Download Song 1 → Upload Song 1 (Drive) → Sheets'e yaz (Parçalar)
    ↓
Build Return Data  (totalDuration güncellenir, tracks[] ve driveFileIds[] büyür)
```

### AŞAMA 3: Kapak Görseli

```
Duration Reached? (EVET)
    ↓
Search files and folders  (Drive'da youtube.jpg arar)
    ↓
If (youtube.jpg var mı?)
    ├── EVET → Prep Drive Cover → Upload to Cloudinary
    └── HAYIR → Build Gemini Prompt → Generate Cover Image
                    → Extract Base64 → Upload to Cloudinary
```

> **Not:** `youtube.jpg` kullanmak istersen Drive klasörüne at. Önemli: gerçek JPG olmalı (sıkıştırılmış, 10MB altı). PNG'yi rename etme, gerçekten JPG olarak kaydet. Yoksa Gemini ile otomatik üretir.

### AŞAMA 4: Render ve YouTube

```
Upload to Cloudinary
    ↓
Assemble Payload   (githubBody hazırlar: session_id, audio_urls, cover_image_url,
                    video_title, youtube_description, youtube_tags)
    ↓
Update Oturumlar   (Sheets → durum: rendering, cover_url, tracks_json, youtube_title yazar)
    ↓
Trigger GitHub Actions  (render.yml'i tetikler)
    ↓
Telegram Notify    ("Pipeline başladı" mesajı)
```

**GitHub Actions render.yml içinde ne olur:**
```
Install dependencies (ffmpeg, python)
    ↓
Download all audio files  (audio_urls JSON array'inden tüm MP3'leri indirir)
    ↓
Download cover image  (Cloudinary URL'den kapağı indirir)
    ↓
Prepare cover 1920x1080  (FFmpeg: increase+crop, siyah çerçeve yok)
    ↓
Build FFmpeg concat list  (tüm MP3'leri sıralı listeler)
    ↓
Concatenate all audio  (-c copy ile kayıpsız birleştirir)
    ↓
Render video  (statik kapak + audio → MP4, AAC 320k, 44100Hz stereo)
    ↓
Upload to YouTube  (OAuth2 chunked upload, public video)
    ↓
Notify n8n  (webhook POST → session_id + youtube_id gönderir)
```

### AŞAMA 5: Callback ve Tamamlanma

```
YouTube Callback Webhook  (GitHub Actions'dan POST alır)
    ↓
Update Oturum Final  (Sheets → durum: used, youtube_id yazar)
    ↓
Telegram Final Notify  ("YouTube'a yüklendi + link" mesajı)
```

---

## 📁 Dosyalar

### Yerel dosyalar (F:\AK\Karşıdan Yüklemeler\)

| Dosya | Açıklama |
|---|---|
| `🎼 Music Pipeline - Ana Workflow_V2 (4).json` | Ana workflow — n8n'e import edilecek |
| `🎵 Suno Song Generator_V2.json` | Sub-workflow — n8n'e import edilecek |
| `render.yml` | GitHub Actions render script |
| `MUSIC_PIPELINE_README.md` | Bu dosya |

### GitHub (kesikoglu/workflows)
- `.github/workflows/render.yml` — FFmpeg + YouTube upload script

### n8n (Railway)
- Ana workflow ID: `qzqKWimt7hBegT9h`
- Sub-workflow ID: `DDTpRrWVO3TTgdS3`

---

## 🔑 Credential'lar ve Secret'lar

### n8n Environment Variables
| Değişken | Nerede Kullanılır |
|---|---|
| `OPENAI_API_KEY` | GPT-4o içerik üretimi |
| `GEMINI_API_KEY` | Kapak görseli üretimi |
| `GITHUB_TOKEN` | GitHub Actions tetikleme |

### n8n Credentials (sol sidebar → Credentials)
| Credential | Kullanıldığı Node'lar |
|---|---|
| `Google Sheets account` (ID: pJsmok21k6tkzA85) | Log Session, Update Oturumlar, Update Oturum Final, Parçalar |
| `Google Drive account 2` (ID: 5lD42L63Rgi5GNDl) | Upload Song 0/1, Search files |

### GitHub Secrets (kesikoglu/workflows → Settings → Secrets)
| Secret | Açıklama |
|---|---|
| `YT_CLIENT_ID` | YouTube OAuth client ID |
| `YT_CLIENT_SECRET` | YouTube OAuth client secret |
| `YT_REFRESH_TOKEN` | YouTube refresh token (expire olursa yenilenmeli) |
| `GITHUB_TOKEN` | Otomatik (Actions tarafından sağlanır) |

### KIE AI API Key
- Sub-workflow → "Create a Song" node → Authorization header içinde
- `8f1fb0167c9d0f575fe0ea95fe097406`

---

## 📊 Google Sheets Yapısı

**Spreadsheet:** `n8n_Suno_music`  
**URL:** `docs.google.com/spreadsheets/d/10zv_8dwXpDbYrEmG1JM34ywOmQ7VEY20IAxIOVIxQyQ`

### Parçalar sheet (her üretilen MP3 için bir satır)
| Kolon | Açıklama |
|---|---|
| session_id | Hangi oturuma ait |
| musicType, mood, tempo | Form değerleri |
| hasLyrics, voiceGender | Sözlü mü, ses cinsiyeti |
| audio_url | Suno CDN URL (cdn1.suno.ai) |
| duration_sn | Parça süresi (saniye) |
| suno_title | Suno'nun verdiği başlık |
| durum | `not-used` (şimdilik güncellenmyor) |
| drive_file_id | Google Drive'daki dosya ID'si |

### Oturumlar sheet (her video için bir satır)
| Kolon | Açıklama |
|---|---|
| session_id | Benzersiz oturum ID'si |
| musicType, mood, tempo | Form değerleri |
| hedef_sn | Hedef süre (saniye) |
| cover_url | Cloudinary kapak URL'i |
| youtube_title | Video başlığı |
| youtube_id | YouTube video ID'si |
| durum | `not-used` → `rendering` → `used` |
| tarih | Oluşturma tarihi |
| tracks_json | Tüm audio URL'lerin JSON dizisi |

---

## ⚠️ Bilinen Sorunlar ve Çözümleri

### YouTube OAuth Token Süresi Dolar
**Belirti:** GitHub Actions'da `KeyError: 'access_token'` veya `invalid_grant`  
**Çözüm:**
1. Google Cloud Console → `youtube-playground` OAuth client → Credentials
2. OAuth 2.0 Playground → `youtube-playground` client ID/Secret ile token al
3. GitHub Secrets → `YT_REFRESH_TOKEN` güncelle

### n8n'e Import Sonrası Webhook GET Geliyor
**Belirti:** YouTube Callback webhook POST yerine GET bekliyor  
**Çözüm:** n8n'de `YouTube Callback` node'unu aç → HTTP Method → POST yap → Save

### Google Sheets EAUTH Hatası
**Belirti:** Sheets node'larında `invalid_grant` veya EAUTH  
**Çözüm:** n8n → Credentials → `Google Sheets account` → Reconnect → Google hesabıyla yeniden giriş

### Cloudinary "File size too large"
**Belirti:** `youtube.jpg` yüklenmiyor, 10MB limiti aşıldı  
**Çözüm:** Dosyayı gerçek JPG olarak kaydet (Paint → Farklı Kaydet → JPEG). 10MB altı olmalı.

### Audio Kalitesi Düşük (Cızırtı)
**Belirti:** Videoda arka planda hissing sesi  
**Nedenler:** `sourceAudioUrl` (KIE CDN) yerine `sourceStreamAudioUrl` (Suno CDN) kullanılmalı  
**Çözüm:** Sub-workflow → Parse Song Data → `sourceStreamAudioUrl` kullan ✅ (düzeltildi)

### Instrumental Seçildi Ama Vokal Var
**Belirti:** Instrumental seçildiğinde kadın/erkek ses duyuluyor  
**Neden:** `vocalGender` parametresi instrumental'da da gönderiliyordu  
**Çözüm:** Build API Body node'u — instrumental=true ise vocalGender gönderilmiyor ✅ (düzeltildi)

---

## 🔧 Bakım Notları

### Ayda bir kontrol et:
- [ ] YouTube refresh token süresi (genellikle 6 ay geçerli)
- [ ] KIE AI kredi bakiyesi
- [ ] Railway n8n instance çalışıyor mu
- [ ] Google Sheets OAuth credential hâlâ bağlı mı

### İki Ayda Bir:
- [ ] Suno model versiyonu (V5_5 → yeni versiyon çıkmış olabilir, sub-workflow'da güncelle)

---

## 🚀 Sıfırdan Kurulum Gerekirse

1. **n8n'e import et:**
   - `🎵 Suno Song Generator_V2.json` → önce import et (sub-workflow)
   - `🎼 Music Pipeline - Ana Workflow_V2 (4).json` → sonra import et (ana workflow)
   - Ana workflow'da Execute Song Generator node'u → sub-workflow ID'yi doğrula (`DDTpRrWVO3TTgdS3`)

2. **Credentials bağla:**
   - Her Sheets node'unda `Google Sheets account` seç
   - Her Drive node'unda `Google Drive account 2` seç

3. **YouTube Callback webhook'u düzelt:**
   - `YouTube Callback` node → HTTP Method → POST yap

4. **Workflow'u aktif et** (sağ üst toggle)

5. **GitHub render.yml'i kontrol et:**
   - `kesikoglu/workflows` → `.github/workflows/render.yml`
   - "Notify n8n" step'inin sonunda olduğunu doğrula

---

## 📌 Hızlı Referans

| Ne | Nerede |
|---|---|
| Form URL | n8n → Ana Workflow → Form Trigger → Test URL (production'da webhook URL) |
| n8n | `https://n8n-production-eefc3.up.railway.app` |
| Google Sheets | `n8n_Suno_music` spreadsheet |
| Drive klasörü | `suno_music` (ID: `1W876Os3EH6LMfJp0G6Vk6bEZcv8JTEZZ`) |
| GitHub Actions | `github.com/kesikoglu/workflows/actions` |
| Cloudinary | `cloudinary.com` → Account: `desntuwcc` → Media Library → `reals` klasörü |
| YouTube kanalı | YouTube Studio → `kesikoglu` |
