# MeetMind — Yapay Zeka Destekli Toplantı Asistanı

> Toplantı notlarını otomatik olarak takip edilen, atanan ve iletilen eyleme dönüştürülebilir görevlere dönüştürün.

**Canlı Dashboard:** https://meetmind-hackathon-takim11.github.io/meetmind/

**Tanıtım Videosu:** https://canva.link/takim11hackathonmeetmind

---

## MeetMind Nedir?

MeetMind, ham toplantı notlarını Google Gemini yapay zekası kullanarak yapılandırılmış görevlere dönüştüren bir toplantı verimliliği aracıdır. Görevler otomatik olarak Google Sheets'e senkronize edilir, Gmail üzerinden ekip üyelerine atanır ve Google Takvim'e işlenir — tek bir yapıştırma işlemiyle.

GitHub Pages üzerinde barındırılan tam statik bir frontend (vanilla HTML/CSS/JS) ile oluşturulmuştur; backendde n8n otomasyon iş akışları çalışmaktadır.

---

## Özellikler

- **Yapay Zeka ile Görev Çıkarımı** — Toplantı notlarını yapıştırın, Gemini görevleri, sorumlu kişileri, son tarihleri ve öncelikleri otomatik olarak çıkarır
- **İnsan Onaylı Akış** — Herhangi bir şey kaydedilmeden veya gönderilmeden önce yapay zekanın çıkardığı görevleri inceleyin ve onaylayın
- **Canlı Dashboard** — Google Sheets'ten senkronize edilen KPI kartları, risk ölçeri, duygu durum analizi ve kişi dağılımı
- **Görev Yönetimi** — Görev durumlarını doğrudan gösterge panelinden filtreleyin, sıralayın ve güncelleyin
- **Takvim Görünümü** — Etkileşimli aylık takvim ile son tarih takibi
- **Toplantı Arşivi** — Geçmiş toplantılara ve ilgili görevlere göz atın
- **Sesli Notlar** — Yazmak yerine Telegram'a sesli mesaj gönderin; Whisper yapay zekası metne otomatik dönüştürür
- **Haftalık Özet** — Her Pazartesi sabahı tamamlanan ve bekleyen görevleri özetleyen otomatik e-posta
- **Durum Senkronizasyonu** — Gösterge panelindeki görev durumunu güncelleyin, gerçek zamanlı olarak Google Sheets'e yansısın

---

## Tech Stack

| Katman | Teknoloji |
|---|---|
| Frontend | Vanilla HTML, CSS, JavaScript |
| Hosting | GitHub Pages |
| Otomasyon | n8n (Cloud) |
| Yapay Zeka — Görev Çıkarımı | Google Gemini API |
| Yapay Zeka — Ses Transkripsiyonu | OpenAI Whisper API |
| Veritabanı | Google Sheets + SheetDB |
| Email | n8n üzerinden Gmail |
| Takvim | n8n üzerinden Google Takvim |
| Sesli Arayüz | Telegram Bot |

---

## Mimari

```
┌─────────────────────────────────────────────────────┐
│                   GitHub Pages                       │
│                  (index.html)                        │
│                                                      │
│  ┌─────────┐  ┌─────────┐  ┌──────────┐  ┌──────┐  │
│  │ Panel   │  │ Gönder  │  │ Görevler │  │Diğer │  │
│  └────┬────┘  └────┬────┘  └────┬─────┘  └──────┘  │
└───────┼────────────┼────────────┼───────────────────┘
        │            │            │
        │ SheetDB    │ Webhook    │ Webhook
        │ (GET)      │ (POST)     │ (POST)
        ▼            ▼            ▼
┌───────────────────────────────────────────────────┐
│                    n8n Cloud                       │
│                                                   │
│  İş Akışı 1         İş Akışı 2   İş Akışı 3      │
│  Ana Akış           Haftalık     Telegram Bot     │
│                     E-posta                       │
│  İş Akışı 4                                       │
│  Durum Güncelleme                                 │
└───────┬───────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────┐
│  Google Sheets (SheetDB)      │
│  Google Takvim                │
│  Gmail                        │
│  Telegram                     │
└───────────────────────────────┘
```

---

## n8n İş Akışları

### İş Akışı 1 — Ana Toplantı Akışı

Temel iş akışı. `action` alanıyla birbirinden ayrılan analiz ve onay aşamalarını tek bir webhook üzerinden yönetir.

**Analiz aşaması** (`action: "analyze"`):
```
Webhook → Metin Temizle → IF (action == confirm?)
                               ↓ FALSE
                          Gemini Body Hazırla → HTTP Request (Gemini API)
                          → JSON Çözümle → Veri Hazırla
                          → Respond to Webhook (çıkarılan görevleri ön yüze döndürür)
```

**Onay aşaması** (`action: "confirm"`):
```
Webhook → Metin Temizle → IF (action == confirm?)
                               ↓ TRUE
                          Confirm Hazırla → Sheets (satır ekle)
                                         → E-posta Şablonu → IF E-posta Varsa → Gmail
                                         → IF Son Tarih Varsa → Renk Haritası → Takvim Etkinliği
                                         → Takip Tarihi Al → Takvim Etkinliği
                          → Respond to Webhook
```

**Tetikleyici:** Ön yüzden POST webhook
**Girdiler:** `meeting_title`, `meeting_notes`, `meeting_date`, `action`, `action_items`
**Çıktılar:** Gemini JSON (analiz) / Sheets + Gmail + Takvim (onay)

---

### İş Akışı 2 — Haftalık Pazartesi E-postası

Her Pazartesi sabahı otomatik olarak çalışır. Google Sheets'teki tüm görevleri çeker, tamamlanan ve bekleyen maddeleri Gemini AI ile özetler ve ekibe motivasyonel bir özet e-postası gönderir.

```
Cron (Her Pazartesi) → HTTP Request (SheetDB — tüm görevleri getir)
→ Kod (tamamlanan / bekleyen / süresi geçmiş özetle)
→ Gmail (haftalık özeti tüm ekibe gönder)
```

**Tetikleyici:** Cron zamanlaması (Pazartesi 08:30)
**Çıktılar:** Tüm ekip üyelerine haftalık özet e-postası

---

### İş Akışı 3 — Telegram Sesli Not

Toplantı notlarını yazmak yerine Telegram üzerinden sesli mesaj olarak göndermeye olanak tanır. Whisper yapay zekası sesi metne çevirir, ardından aynı Gemini boru hattı metni işler.

```
Telegram Tetikleyici (sesli mesaj alındı)
→ Ses Dosyasını İndir
→ HTTP Request (OpenAI Whisper API — transkribe et)
→ Metin Temizle → Gemini Body Hazırla → HTTP Request (Gemini API)
→ JSON Çözümle → Veri Hazırla
→ Telegram (kullanıcıya onay mesajı gönder)
→ [Kullanıcı onaylamak için "Evet" yazar]
→ Sheets + Gmail + Takvim
```

**Tetikleyici:** Telegram bot sesli mesajı
**Çıktılar:** Transkripsiyon → görev çıkarımı → İş Akışı 1 onayıyla aynı

---

### İş Akışı 4 — Durum Güncelleme

Kullanıcı gösterge panelindeki görev durumu açılır menüsünü değiştirdiğinde tetiklenir. İlgili satırı Google Sheets'te gerçek zamanlı olarak günceller.

```
Webhook (POST) → Google Sheets (Görev sütununa göre satırı güncelle)
→ Respond to Webhook ({"success": true})
```

**Tetikleyici:** Ön yüz durum açılır menüsünden POST webhook
**Girdiler:** `id` (görev adı), `status` (yeni durum değeri)
**Çıktılar:** Google Sheets'teki `Durum` sütunu güncellenir

---

## Google Sheets Yapısı

Sayfada `Log` adında bir sekme bulunmalı ve aşağıdaki sütun başlıkları tam olarak bu şekilde yer almalıdır:

| Sütun | Açıklama |
|---|---|
| `Toplantı Adı` | Toplantı adı |
| `Sorumlu` | Atanan kişi |
| `Görev` | Görev açıklaması |
| `Deadline` | Son tarih (YYYY-AA-GG) |
| `Öncelik` | Öncelik (Yüksek / Orta / Düşük) |
| `Durum` | Durum (Bekliyor / Devam Ediyor / Tamamlandı) |
| `Tarih` | Toplantı tarihi |
| `E-Posta` | Sorumlu kişinin e-posta adresi |

---

## Kurulum

### Ön Koşullar

- n8n Cloud hesabı (ücretsiz plan yeterlidir)
- Google hesabı (Sheets, Gmail, Takvim)
- Gemini API anahtarı ([aistudio.google.com](https://aistudio.google.com))
- SheetDB hesabı ([sheetdb.io](https://sheetdb.io))
- Telegram Bot (isteğe bağlı, sesli notlar için)
- OpenAI API anahtarı (isteğe bağlı, ses transkripsiyonu için)

### Adımlar

**1. Google Sheets**
- Yeni bir Google Sheets dosyası oluşturun
- `Log` adında bir sekme ekleyin
- 1. satıra yukarıda listelenen sütun başlıklarını ekleyin
- SheetDB'ye bağlayın → API URL'sini kopyalayın

**2. n8n**
- `/workflows` klasöründeki 4 iş akışı JSON dosyasını içe aktarın
- Kimlik bilgilerinizi ekleyin (Google, Gemini, Gmail, Telegram)
- Tüm iş akışlarını etkinleştirin
- İş Akışı 1 ve İş Akışı 4'teki webhook URL'lerini kopyalayın

**3. Ön Yüz Yapılandırması**
- `index.html` dosyasını açın
- Üstteki `CONFIG` nesnesini bulun ve değerleri girin:

```javascript
const CONFIG = {
  webhookUrl:        'N8N_IS_AKISI_1_WEBHOOK_URL',
  confirmWebhookUrl: 'N8N_IS_AKISI_1_WEBHOOK_URL', // aynı URL, action alanı ayırt eder
  sheetDbUrl:        'SHEETDB_API_URL',
  updateWebhookUrl:  'N8N_IS_AKISI_4_WEBHOOK_URL',
  voiceWebhookUrl:   'N8N_IS_AKISI_3_WEBHOOK_URL',
  refreshInterval:   30000,
};
```

**4. Yayına Alma**
- `index.html` dosyasını GitHub reponuza push edin
- Repo Ayarları → Pages bölümünden GitHub Pages'i etkinleştirin
- Kaynak olarak main dalını ve kök klasörü seçin

---

## Nasıl Çalışır — Kullanıcı Akışı

```
1. Dashboardu açın → meetmind-hackathon-takim11.github.io
2. "Toplantı Ekle"ye tıklayın veya "Toplantı Gönder" sayfasına gidin
3. Toplantı adını ve tarihini girin, toplantı notlarını yapıştırın
4. "Gönder"e tıklayın — Gemini notları analiz eder
5. Onay panelinde çıkarılan görevleri inceleyin
6. "Onayla"ya tıklayın — görevler Sheets'e kaydedilir, e-postalar gönderilir, takvim etkinlikleri oluşturulur
7. Gösterge paneli otomatik olarak yenilenir — görevler tüm görünümlerde belirir
```

Alternatif olarak, Telegram botuna sesli mesaj gönderin — geri kalan akış aynıdır.

---

## Proje Yapısı

```
/
├── index.html          # Tam ön yüz — panel, gönder, görevler, takvim, notlar, ses
├── README.md
└── workflows/
    ├── workflow1-main.json
    ├── workflow2-weekly-email.json
    ├── workflow3-telegram-voice.json
    └── workflow4-status-update.json
```

---

## Ekip

**Takım 11** tarafından Yapay Zeka ve Teknoloji Akademisi Low Code / No Code Hackathon için geliştirilmiştir.

Kerziban Sicim | Özde Hava Sinecek | Enes Koyuncu | Göktuğ Kocatürk

---

