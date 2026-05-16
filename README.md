# OpenProject on Coolify -- Docker Compose Deployment Guide

> Adapted from the [official OpenProject Docker Compose repository](https://github.com/opf/openproject-docker-compose) for seamless deployment on [Coolify](https://coolify.io).

**OpenProject 17 | Coolify v4.x | PostgreSQL 17**

---

**Language / Dil:** [English](#english) | [Turkce](#turkce)

---

<a id="english"></a>

# English

## Overview

This repository contains a **Coolify-compatible** Docker Compose configuration for [OpenProject](https://www.openproject.org) -- an open-source project management software.

The official OpenProject Docker Compose setup is designed for bare-metal VPS deployments with its own Caddy reverse proxy. This adapted version removes the built-in proxy and lets **Coolify's Traefik** handle SSL termination and domain routing, eliminating redirect loops and port conflicts.

### Architecture

```
Internet
  |
  v
Coolify Traefik (SSL termination + domain routing)
  |
  v
web (OpenProject Puma :8080)
  |
  +---> db (PostgreSQL :5432)
  +---> cache (Memcached :11211)
  +---> hocuspocus (Collaborative editing :1234)
  +---> worker (Background jobs)
  +---> cron (Scheduled tasks)
  +---> seeder (Database seeding, runs once)
  +---> autoheal (Container health monitoring)
```

### Key Differences from Official Setup

| Feature | Official Setup | Coolify Setup |
|---|---|---|
| Reverse Proxy | Built-in Caddy container | Coolify's Traefik (removed Caddy) |
| SSL Certificates | Manual or Caddy auto-SSL | Coolify auto-provisions via Let's Encrypt |
| Port Binding | Exposes port 8080 to host | No host port binding (Traefik routes internally) |
| YAML Anchors | Uses `<<: *app` aliases | Fully expanded (better Coolify compatibility) |
| Configuration | `.env` file | Coolify Environment Variables panel |

---

## Prerequisites

- A server with [Coolify v4.x](https://coolify.io) installed and running
- A domain name pointed to your server's IP address (A record)
- Access to your DNS provider

---

## Step-by-Step Deployment

### Step 1: Prepare DNS

Create an **A record** at your DNS provider:

| Type | Name | Value |
|---|---|---|
| A | `your-subdomain` | `your-server-ip` |

**Example:** `A | project | 203.0.113.50` for `project.example.com`

> **Cloudflare Users:** Set the proxy toggle to **DNS only** (grey cloud). If the orange cloud (proxy) is enabled, Coolify cannot obtain a Let's Encrypt certificate and you will get SSL errors.

### Step 2: Generate Secrets

You need two secrets before starting. Generate them on Linux/macOS:

```bash
# Generate SECRET_KEY_BASE
openssl rand -hex 64

# Generate COLLABORATIVE_SERVER_SECRET
openssl rand -hex 32

# Generate POSTGRES_PASSWORD
openssl rand -hex 16
```

Save these values -- you will need them in Step 4.

### Step 3: Create Application in Coolify

1. Log into your Coolify dashboard
2. Navigate to **Projects** > select your project (or create a new one)
3. Click **+ New Resource**
4. Select **Docker Compose**
5. Copy the entire contents of [`docker-compose.yml`](./docker-compose.yml) and paste it into the editor
6. Click **Save**

### Step 4: Configure Environment Variables

Go to the application's **Environment Variables** section and add:

| Variable | Value | Description |
|---|---|---|
| `SECRET_KEY_BASE` | *(from Step 2)* | Rails secret key for sessions and encryption |
| `POSTGRES_PASSWORD` | *(from Step 2)* | PostgreSQL database password |
| `COLLABORATIVE_SERVER_SECRET` | *(from Step 2)* | Secret for the Hocuspocus collaboration server |
| `OPENPROJECT_HOST__NAME` | `your-domain.com` | Your domain without protocol (e.g. `project.example.com`) |

> **Important:** Do not include `https://` in `OPENPROJECT_HOST__NAME`. Just the bare domain.

### Step 5: Assign Domain to Web Service

1. In the application view, click on the **web** service from the service list
2. In the **Domains** field, enter: `https://your-domain.com`
3. In the **Port** field, enter: `8080`
4. Click **Save**

> **Note:** Only the `web` service needs a domain. All other services communicate internally.

### Step 6: Deploy

1. Click the **Deploy** button
2. Monitor the logs in the **Logs** tab
3. Wait for the `seeder` service to complete (first run takes 2-5 minutes)
4. Wait for the `web` service health checks to show `status=200`

### Step 7: First Login

Once deployment is complete, visit `https://your-domain.com`

- **Username:** `admin`
- **Password:** `admin`

> **Security:** Change the admin password immediately after first login.

---

## Troubleshooting

### "Too many redirects" / Redirect Loop

**Cause:** A proxy layer between Coolify's Traefik and OpenProject is interfering with `X-Forwarded-Proto` headers.

**Solution:** Make sure there is no additional proxy service (like Caddy or Nginx) between Traefik and the `web` service. The domain should be assigned directly to the `web` service on port `8080`.

### "Port is already allocated"

**Cause:** The docker-compose exposes a port that is already in use on the host.

**Solution:** Remove any `ports:` sections from the docker-compose. Coolify's Traefik routes traffic internally -- no host port binding is needed.

### "Degraded (unhealthy)" Status

**Cause:** The `web` service health check might fail during initial setup while the database is being seeded.

**Solution:** Wait 3-5 minutes for the `seeder` to finish. Check `seeder` logs to confirm it completed. The `web` service will become healthy after the seeder finishes and the application boots.

### SSL Certificate Errors

**Possible causes:**
1. DNS is not pointing to the correct server IP
2. Cloudflare proxy (orange cloud) is enabled -- set to DNS only (grey cloud)
3. Ports 80 and 443 are blocked by a firewall

**Verify DNS:**
```bash
dig +short your-domain.com
# Should return your server's IP
```

### Coolify Redirects to Its Own Dashboard

**Cause:** The domain is not properly assigned to any service, so Traefik falls back to the default route.

**Solution:** Ensure the domain is set on the `web` service with the correct port (`8080`), and that the format is `https://your-domain.com` (with protocol, without port).

---

## Email / SMTP Configuration (Optional)

To enable email notifications (work package updates, mentions, password resets), configure SMTP via Coolify's Environment Variables panel.

### Required SMTP Variables

| Variable | Example | Description |
|---|---|---|
| `OPENPROJECT_EMAIL__DELIVERY__METHOD` | `smtp` | Must be `smtp` to enable email |
| `OPENPROJECT_SMTP__ADDRESS` | `smtp.gmail.com` | SMTP server hostname |
| `OPENPROJECT_SMTP__PORT` | `587` | SMTP port (587 for STARTTLS, 465 for SSL) |
| `OPENPROJECT_SMTP__DOMAIN` | `example.com` | Your email domain |
| `OPENPROJECT_SMTP__AUTHENTICATION` | `plain` | Auth method: `plain`, `login`, or `cram_md5` |
| `OPENPROJECT_SMTP__USER__NAME` | `user@example.com` | SMTP username |
| `OPENPROJECT_SMTP__PASSWORD` | `app-password` | SMTP password or app-specific password |
| `OPENPROJECT_SMTP__ENABLE__STARTTLS__AUTO` | `true` | Enable STARTTLS encryption |
| `OPENPROJECT_MAIL__FROM` | `openproject@example.com` | Sender address for outgoing emails |

### Gmail Example

> **Note:** Gmail requires an [App Password](https://support.google.com/accounts/answer/185833) -- your regular password will not work.

| Variable | Value |
|---|---|
| `OPENPROJECT_EMAIL__DELIVERY__METHOD` | `smtp` |
| `OPENPROJECT_SMTP__ADDRESS` | `smtp.gmail.com` |
| `OPENPROJECT_SMTP__PORT` | `587` |
| `OPENPROJECT_SMTP__DOMAIN` | `gmail.com` |
| `OPENPROJECT_SMTP__AUTHENTICATION` | `plain` |
| `OPENPROJECT_SMTP__USER__NAME` | `your-email@gmail.com` |
| `OPENPROJECT_SMTP__PASSWORD` | `your-app-password` |
| `OPENPROJECT_SMTP__ENABLE__STARTTLS__AUTO` | `true` |
| `OPENPROJECT_MAIL__FROM` | `your-email@gmail.com` |

### Verify Email Delivery

After deploying with SMTP variables:

1. Log into OpenProject as admin
2. Go to **Administration** > **Emails and notifications** > **Email notifications**
3. Click **Send a test email**
4. Check your inbox for the test email

If the email does not arrive, check the `worker` service logs in Coolify for SMTP errors.

---

## Services Reference

| Service | Image | Purpose |
|---|---|---|
| `db` | `postgres:17` | PostgreSQL database |
| `cache` | `memcached` | In-memory caching |
| `web` | `openproject/openproject:17-slim` | Main application server (Puma) |
| `worker` | `openproject/openproject:17-slim` | Background job processing |
| `cron` | `openproject/openproject:17-slim` | Scheduled tasks (emails, cleanup) |
| `seeder` | `openproject/openproject:17-slim` | Database schema and seed data (runs once) |
| `hocuspocus` | `openproject/hocuspocus:17.4.0` | Real-time collaborative editing |
| `autoheal` | `willfarrell/autoheal:1.2.0` | Restarts unhealthy containers |

---

## Backup & Restore

### Backup

To back up your OpenProject data, you need to save two things:

1. **PostgreSQL database** -- the `pgdata` volume
2. **Uploaded assets** -- the `opdata` volume

You can use Coolify's built-in backup features or manually back up the Docker volumes.

### Upgrading OpenProject

When a new version is released:

1. Update the image tags in `docker-compose.yml` (e.g. `17-slim` to `18-slim`)
2. Update the `hocuspocus` image tag to match
3. Redeploy from Coolify

> **Warning:** Always back up your database before upgrading.

---

## License

OpenProject is licensed under the [GNU General Public License v3.0](https://www.gnu.org/licenses/gpl-3.0.html).
This deployment guide is provided as-is for community use.

---
---
---

<a id="turkce"></a>

# Turkce

## Genel Bakis

Bu depo, [OpenProject](https://www.openproject.org) acik kaynak proje yonetim yaziliminin **Coolify uyumlu** Docker Compose konfigurasyonunu icermektedir.

Resmi OpenProject Docker Compose kurulumu, kendi Caddy reverse proxy'si ile VPS uzerinde calisacak sekilde tasarlanmistir. Bu uyarlanmis surum, yerlesik proxy'yi kaldirir ve **Coolify'in Traefik'inin** SSL sonlandirma ve domain yonlendirmesini yapmasina izin verir. Boylece yonlendirme donguleri ve port cakismalari onlenir.

### Mimari

```
Internet
  |
  v
Coolify Traefik (SSL sonlandirma + domain yonlendirme)
  |
  v
web (OpenProject Puma :8080)
  |
  +---> db (PostgreSQL :5432)
  +---> cache (Memcached :11211)
  +---> hocuspocus (Isbirlikci duzenleme :1234)
  +---> worker (Arka plan isleri)
  +---> cron (Zamanlanmis gorevler)
  +---> seeder (Veritabani tohulama, bir kez calisir)
  +---> autoheal (Container saglik izleme)
```

### Resmi Kurulumdan Farklar

| Ozellik | Resmi Kurulum | Coolify Kurulumu |
|---|---|---|
| Reverse Proxy | Dahili Caddy container | Coolify'in Traefik'i (Caddy kaldirildi) |
| SSL Sertifikalari | Manuel veya Caddy otomatik | Coolify, Let's Encrypt ile otomatik saglar |
| Port Baglama | 8080 portunu host'a acar | Host port baglama yok (Traefik dahili yonlendirir) |
| YAML Referanslari | `<<: *app` alias kullanir | Tamamen acik yazilmis (daha iyi Coolify uyumlulugu) |
| Konfigrasyon | `.env` dosyasi | Coolify Ortam Degiskenleri paneli |

---

## On Kosullar

- [Coolify v4.x](https://coolify.io) kurulu ve calisan bir sunucu
- Sunucunuzun IP adresine yonlendirilmis bir domain adi (A kaydi)
- DNS saglayiciniza erisim

---

## Adim Adim Kurulum

### Adim 1: DNS'i Hazirla

DNS saglayicinizda bir **A kaydi** olusturun:

| Tip | Ad | Deger |
|---|---|---|
| A | `alt-domain-adiniz` | `sunucu-ip-adresiniz` |

**Ornek:** `A | proje | 203.0.113.50` -> `proje.example.com`

> **Cloudflare Kullanicilari:** Proxy duzenleme dugmesini **Yalniz DNS** (gri bulut) olarak ayarlayin. Turuncu bulut (proxy) etkinse, Coolify Let's Encrypt sertifikasi alamaz ve SSL hatalari alirsiniz.

### Adim 2: Gizli Anahtarlari Olustur

Baslamadan once iki gizli anahtar gerekiyor. Linux/macOS'ta olusturun:

```bash
# SECRET_KEY_BASE olustur
openssl rand -hex 64

# COLLABORATIVE_SERVER_SECRET olustur
openssl rand -hex 32

# POSTGRES_PASSWORD olustur
openssl rand -hex 16
```

Bu degerleri kaydedin -- Adim 4'te ihtiyaciniz olacak.

### Adim 3: Coolify'da Uygulama Olustur

1. Coolify kontrol paneline giris yapin
2. **Projects** > projenizi secin (veya yeni bir tane olusturun)
3. **+ New Resource** butonuna tiklayin
4. **Docker Compose** secin
5. [`docker-compose.yml`](./docker-compose.yml) dosyasinin tum icerigini kopyalayip editore yapistirin
6. **Save** butonuna tiklayin

### Adim 4: Ortam Degiskenlerini Yapilandir

Uygulamanin **Environment Variables** bolumune gidin ve ekleyin:

| Degisken | Deger | Aciklama |
|---|---|---|
| `SECRET_KEY_BASE` | *(Adim 2'den)* | Oturumlar ve sifreleme icin Rails gizli anahtari |
| `POSTGRES_PASSWORD` | *(Adim 2'den)* | PostgreSQL veritabani sifresi |
| `COLLABORATIVE_SERVER_SECRET` | *(Adim 2'den)* | Hocuspocus isbirligi sunucusu icin gizli anahtar |
| `OPENPROJECT_HOST__NAME` | `domain-adiniz.com` | Protokol olmadan domain adiniz (orn. `proje.example.com`) |

> **Onemli:** `OPENPROJECT_HOST__NAME` degerine `https://` eklemeyin. Sadece domain adini yazin.

### Adim 5: Domain'i Web Servisine Ata

1. Uygulama gorunumunde, servis listesinden **web** servisine tiklayin
2. **Domains** alanina girin: `https://domain-adiniz.com`
3. **Port** alanina girin: `8080`
4. **Save** butonuna tiklayin

> **Not:** Yalnizca `web` servisinin domain'e ihtiyaci var. Diger tum servisler dahili olarak iletisim kurar.

### Adim 6: Deploy Et

1. **Deploy** butonuna tiklayin
2. **Logs** sekmesinden loglari takip edin
3. `seeder` servisinin tamamlanmasini bekleyin (ilk calisma 2-5 dakika surer)
4. `web` servisinin saglik kontrollerinin `status=200` gostermesini bekleyin

### Adim 7: Ilk Giris

Deployment tamamlandiginda `https://domain-adiniz.com` adresini ziyaret edin

- **Kullanici adi:** `admin`
- **Sifre:** `admin`

> **Guvenlik:** Ilk giristen hemen sonra admin sifresini degistirin.

---

## Sorun Giderme

### "Cok fazla kez yonlendirdi" / Yonlendirme Dongusu

**Sebep:** Coolify'in Traefik'i ile OpenProject arasindaki bir proxy katmani `X-Forwarded-Proto` basliklarini bozuyor.

**Cozum:** Traefik ile `web` servisi arasinda ek bir proxy servisi (Caddy veya Nginx gibi) olmadigindan emin olun. Domain dogrudan `web` servisine `8080` portunda atanmalidir.

### "Port is already allocated" (Port zaten kullaniliyor)

**Sebep:** Docker-compose, host'ta zaten kullanimda olan bir portu aciyor.

**Cozum:** Docker-compose'dan tum `ports:` bolumlerini kaldirin. Coolify'in Traefik'i trafigi dahili olarak yonlendirir -- host port baglamaya gerek yoktur.

### "Degraded (unhealthy)" Durumu

**Sebep:** Veritabani tohumlanirken `web` servisinin saglik kontrolu baslangicta basarisiz olabilir.

**Cozum:** `seeder`'in bitmesi icin 3-5 dakika bekleyin. Tamamlandigini dogrulamak icin `seeder` loglarini kontrol edin. Seeder bittikten ve uygulama baslatildiktan sonra `web` servisi saglikli hale gelecektir.

### SSL Sertifika Hatalari

**Olasi sebepler:**
1. DNS dogru sunucu IP'sine isaret etmiyor
2. Cloudflare proxy'si (turuncu bulut) etkin -- Yalniz DNS'e (gri bulut) ayarlayin
3. 80 ve 443 portlari guvenlik duvari tarafindan engellenmis

**DNS'i dogrulayin:**
```bash
dig +short domain-adiniz.com
# Sunucunuzun IP'sini dondurmelidir
```

### Coolify Kendi Paneline Yonlendiriyor

**Sebep:** Domain hicbir servise duzgun atanmamis, bu yuzden Traefik varsayilan rotaya duser.

**Cozum:** Domain'in `web` servisinde dogru port (`8080`) ile ayarlandigindan ve formatin `https://domain-adiniz.com` (protokol ile, port olmadan) oldugundan emin olun.

---

## E-posta / SMTP Yapilandirmasi (Istege Bagli)

E-posta bildirimlerini (is paketi guncellemeleri, bahsetmeler, sifre sifirlama) etkinlestirmek icin Coolify'in Ortam Degiskenleri panelinden SMTP yapilandirilmalidir.

### Gerekli SMTP Degiskenleri

| Degisken | Ornek | Aciklama |
|---|---|---|
| `OPENPROJECT_EMAIL__DELIVERY__METHOD` | `smtp` | E-postayi etkinlestirmek icin `smtp` olmali |
| `OPENPROJECT_SMTP__ADDRESS` | `smtp.gmail.com` | SMTP sunucu adresi |
| `OPENPROJECT_SMTP__PORT` | `587` | SMTP portu (STARTTLS icin 587, SSL icin 465) |
| `OPENPROJECT_SMTP__DOMAIN` | `example.com` | E-posta domain'iniz |
| `OPENPROJECT_SMTP__AUTHENTICATION` | `plain` | Kimlik dogrulama: `plain`, `login` veya `cram_md5` |
| `OPENPROJECT_SMTP__USER__NAME` | `user@example.com` | SMTP kullanici adi |
| `OPENPROJECT_SMTP__PASSWORD` | `uygulama-sifresi` | SMTP sifresi veya uygulamaya ozel sifre |
| `OPENPROJECT_SMTP__ENABLE__STARTTLS__AUTO` | `true` | STARTTLS sifrelemeyi etkinlestir |
| `OPENPROJECT_MAIL__FROM` | `openproject@example.com` | Giden e-postalar icin gonderici adresi |

### Gmail Ornegi

> **Not:** Gmail bir [Uygulama Sifresi](https://support.google.com/accounts/answer/185833) gerektirir -- normal sifreniz calismaz.

| Degisken | Deger |
|---|---|
| `OPENPROJECT_EMAIL__DELIVERY__METHOD` | `smtp` |
| `OPENPROJECT_SMTP__ADDRESS` | `smtp.gmail.com` |
| `OPENPROJECT_SMTP__PORT` | `587` |
| `OPENPROJECT_SMTP__DOMAIN` | `gmail.com` |
| `OPENPROJECT_SMTP__AUTHENTICATION` | `plain` |
| `OPENPROJECT_SMTP__USER__NAME` | `e-postaniz@gmail.com` |
| `OPENPROJECT_SMTP__PASSWORD` | `uygulama-sifreniz` |
| `OPENPROJECT_SMTP__ENABLE__STARTTLS__AUTO` | `true` |
| `OPENPROJECT_MAIL__FROM` | `e-postaniz@gmail.com` |

### E-posta Teslimini Dogrula

SMTP degiskenleriyle deploy ettikten sonra:

1. OpenProject'e admin olarak giris yapin
2. **Administration** > **Emails and notifications** > **Email notifications** yoluna gidin
3. **Send a test email** butonuna tiklayin
4. Gelen kutunuzu test e-postasi icin kontrol edin

E-posta ulasmadiysa, Coolify'da `worker` servis loglarini SMTP hatalari icin kontrol edin.

---

## Servis Referansi

| Servis | Imaj | Amac |
|---|---|---|
| `db` | `postgres:17` | PostgreSQL veritabani |
| `cache` | `memcached` | Bellek ici onbellekleme |
| `web` | `openproject/openproject:17-slim` | Ana uygulama sunucusu (Puma) |
| `worker` | `openproject/openproject:17-slim` | Arka plan is isleme |
| `cron` | `openproject/openproject:17-slim` | Zamanlanmis gorevler (e-postalar, temizlik) |
| `seeder` | `openproject/openproject:17-slim` | Veritabani semasi ve tohum verileri (bir kez calisir) |
| `hocuspocus` | `openproject/hocuspocus:17.4.0` | Gercek zamanli isbirlikci duzenleme |
| `autoheal` | `willfarrell/autoheal:1.2.0` | Sagliksiz container'lari yeniden baslatir |

---

## Yedekleme ve Geri Yukleme

### Yedekleme

OpenProject verilerinizi yedeklemek icin iki seyi kaydetmeniz gerekir:

1. **PostgreSQL veritabani** -- `pgdata` birimi
2. **Yuklenen dosyalar** -- `opdata` birimi

Coolify'in dahili yedekleme ozelliklerini kullanabilir veya Docker birimlerini manuel olarak yedekleyebilirsiniz.

### OpenProject Guncelleme

Yeni bir surum yayinlandiginda:

1. `docker-compose.yml` dosyasindaki imaj etiketlerini guncelleyin (orn. `17-slim` -> `18-slim`)
2. `hocuspocus` imaj etiketini eslestirin
3. Coolify'dan yeniden deploy edin

> **Uyari:** Guncelleme yapmadan once her zaman veritabanini yedekleyin.

---

## Lisans

OpenProject, [GNU Genel Kamu Lisansi v3.0](https://www.gnu.org/licenses/gpl-3.0.html) ile lisanslanmistir.
Bu kurulum rehberi topluluk kullanimi icin oldugu gibi sunulmaktadir.
