# YouTube Music Pipeline — Kurulum ve Sorun Giderme Rehberi

## Sistem Mimarisi

```
n8n (Railway) → GitHub Actions → YouTube
     ↓                ↓
Google Drive    Suno API (müzik)
     ↓          Gemini API (görsel)
Cloudinary      Cloudinary (kapak)
```

### Workflow Versiyonları
- **V3** → Temel pipeline (stabil, dokunma)
- **V4** → V3 + Tracklist + MP3 sıralama (stabil)
- **V5** → V4 + YouTube Shorts (aktif kullanım)

---

## Google Cloud Console Ayarları

### Proje
- Proje adı: **Test** (vivid-tuner-492322-b8)
- URL: console.cloud.google.com

### OAuth Credentials
| İsim | Tip | Kullanım |
|------|-----|----------|
| youtube-playground | Web app | Token almak için |
| youtube-upload | Desktop | Eski (artık kullanılmıyor) |
| n8n-drive | Web app | n8n Google Drive |

### youtube-playground Redirect URI
`https://developers.google.com/oauthplayground` ekli olmalı.

### ⚠️ ÖNEMLİ: Publishing Status
- **APIs & Services → OAuth consent screen → Audience**
- **Publishing status = "In production"** olmalı!
- Testing modunda token'lar 7 günde expire olur
- Production'da expire olmaz

---

## GitHub Secrets (kesikoglu/workflows repo)

| Secret | Değer Kaynağı |
|--------|--------------|
| YT_CLIENT_ID | youtube-playground Client ID |
| YT_CLIENT_SECRET | youtube-playground Client Secret |
| YT_REFRESH_TOKEN | OAuth Playground'dan alınan token |
| GEMINI_API_KEY | Google AI Studio |

---

## YouTube Refresh Token Yenileme (Adım Adım)

Token expire olduğunda bu adımları takip et:

**1. YouTube'da doğru kanala geç**
- youtube.com → profil ikonu → Hesabı değiştir
- **Golden Sunset Lounge Vibes** seç

**2. OAuth Playground'a git**
- https://developers.google.com/oauthplayground

**3. Settings ⚙️ → Use your own OAuth credentials işaretle**
- OAuth Client ID: youtube-playground'un Client ID'si
- OAuth Client Secret: youtube-playground'un Client Secret'ı
- Close

**4. Sol tarafta YouTube Data API v3 seç**
- `https://www.googleapis.com/auth/youtube.upload` işaretle
- **Authorize APIs** bas

**5. Hesap seçim ekranında**
- **Golden Sunset Lounge Vibes** kanalını seç (önemli!)
- Devam Et

**6. Exchange authorization code for tokens bas**
- `refresh_token` değerini kopyala

**7. GitHub Secrets güncelle**
- repo → Settings → Secrets → YT_REFRESH_TOKEN → Update

---

## Kanal Bilgileri

- Kanal adı: **Golden Sunset Lounge Vibes**
- Kanal ID: `UCl5pCJC7R-VEXmh6EjCbJkA`
- Handle: @GoldenSunsetLoungeVibes

---

## n8n Workflow V5 Node Yapısı

### Ana akış
```
Form Trigger → Build GPT Prompt → Generate AI Content → Init State
→ Prep Session Data → Log Session → Restore State
→ Execute Song Generator (döngü)
→ Duration Reached? (true branch)
  ├── Search files and folders (youtube.jpg Drive'da var mı?)
  ├── Search shorts.jpg (shorts.jpg Drive'da var mı?)
  └── Fetch Tracklist from Sheets
→ If (youtube.jpg var mı?)
  ├── true → Prep Drive Cover
  └── false → Build Gemini Prompt → Generate Cover Image → Extract Base64
→ Upload Cover to Cloudinary → Assemble Payload
→ Delete youtube.jpg → Delete shorts.jpg → Update Oturumlar
→ Trigger GitHub Actions → Telegram Notify
```

### Shorts akışı (paralel)
```
Search shorts.jpg → If Shorts Exists
  ├── true → Prep Shorts Cover (shortsDriveUrl dolu)
  └── false → No Shorts Cover (shortsDriveUrl boş)
```

### ⚠️ Önemli Bağlantı Kuralları
- **Assemble Payload'a sadece Upload Cover to Cloudinary bağlı olmalı**
- Prep Shorts Cover ve No Shorts Cover çıkışları bağlantısız bırak
- Assemble Payload kodu try/catch ile node referansı okur

### Assemble Payload — coverUrl sorunu
```javascript
// $json direkt Upload Cover to Cloudinary'den geliyor (V3 mantığı)
const coverUrl = $json.secure_url;
```

---

## render_v5.yml Akışı

1. Audio dosyaları indir
2. Kapak görselini indir (cover_raw.jpg)
3. 1920x1080 landscape → cover_hd.jpg
4. Shorts kapak:
   - `shorts_cover_url` doluysa → Drive'dan indir
   - Boşsa → Gemini API ile 9:16 portrait üret
   - Gemini başarısız → cover_raw.jpg crop (fallback)
5. Audio birleştir
6. Tracklist oluştur
7. Landscape video render
8. Shorts video render (Ken Burns efekti, 59 sn)
9. YouTube long form upload
10. YouTube Shorts upload (açıklamada ana video linki)
11. n8n callback (youtube_id + shorts_id)

---

## Sık Karşılaşılan Hatalar

### "Required input 'cover_image_url' not provided"
- **Sebep:** Assemble Payload'a birden fazla node bağlı, $json yanlış geliyor
- **Çözüm:** Assemble Payload'a sadece Upload Cover to Cloudinary bağlı olmalı

### "Node 'Upload Cover to Cloudinary' hasn't been executed"
- **Sebep:** Node referansı execution context dışında
- **Çözüm:** $json.secure_url kullan (direkt input), node referansı değil

### "invalid_grant: Token has been expired or revoked"
- **Sebep:** YT_REFRESH_TOKEN süresi dolmuş
- **Çözüm:** Yukarıdaki token yenileme adımlarını takip et

### "invalid_client: The provided client secret is invalid"
- **Sebep:** YT_CLIENT_SECRET yanlış credential'a ait
- **Çözüm:** youtube-playground credential'ının secret'ını kullan

### "Sheet with ID [object Object] not found"
- **Sebep:** Google Sheets node'unda Sheet alanı expression modunda
- **Çözüm:** Sheet → From list → Parçalar seç (By ID değil)

### Video yanlış YouTube kanalına yüklendi
- **Sebep:** Token yanlış kanal için alınmış
- **Çözüm:** YouTube'da Golden Sunset Lounge Vibes kanalına geçip token yenile

### "redirect_uri_mismatch"
- **Sebep:** Desktop app credential OAuth Playground'da çalışmaz
- **Çözüm:** youtube-playground (Web app) credential kullan

---

## MP3 Sıralama Mantığı (V4+)

Suno her prompt için 2 versiyon üretiyor.
Sheets'e yazılma sırası: 1v1, 1v2, 2v1, 2v2...
İstenen render sırası: 1v1, 2v1, 3v1... sonra 1v2, 2v2, 3v2...

Suno workflow Build Return Data'da:
```javascript
const newTracksV1 = [...(p.tracks_v1 || []), p.audioUrl0];
const newTracksV2 = [...(p.tracks_v2 || []), p.audioUrl1];
const newTracks = [...newTracksV1, ...newTracksV2];
```

---

## Google Drive Klasör ID'leri

- Ana klasör (youtube.jpg, shorts.jpg): `1W876Os3EH6LMfJp0G6Vk6bEZcv8JTEZZ`

## Google Sheets

- Spreadsheet ID: `10zv_8dwXpDbYrEmG1JM34ywOmQ7VEY20IAxIOVIxQyQ`
- Sheet adı: Parçalar
