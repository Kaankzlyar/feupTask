# DEI InfoLab: IT varlık ve servis kataloğu

Bir üniversite bölümünün laboratuvarlarını, donanımını, yazılım lisanslarını,
servislerini ve personelini tek bir RDF grafında tutan web sistemi. Veri
GraphDB'de durur, sorgular SPARQL ile yazılır, veriler Excel dosyalarından
yüklenir.

İki arayüzü var. Son kullanıcı tarafı katalogda arama ve inceleme yapar,
backoffice tarafı kayıtları yönetir.

## Bu repo hakkında

Bu bir pratik proje. FEUP/MEIC tez devam çalışmasının dört adımını milestone
olarak taklit ediyor. Amaç gerçek görev başlamadan önce aynı teknolojilerle
aynı sırayı bir kez yürümek.

| Milestone | Issue | Gerçek görevdeki karşılığı |
|---|---|---|
| M0: Hazırlık ve temel | #1 - #4 | (hazırlık, gerçek görevde ayrı adım değil) |
| M1: Sunucu konfigürasyonu | #5 - #9 | InfoLab-DEI sunucusunun kurulumu |
| M2: Sistem deployment | #10 - #15 | GraphDB ve yazılımın yayına alınması |
| M3: Yazılım güncellemesi | #16 - #21 | son kullanıcı ve backoffice arayüzleri |
| M4: Veri oluşturma | #22 - #26 | Excel dosyalarından veri yükleme |

## Çalışma ortamı

İki makine var. Windows bilgisayarın iş istasyonu, uzaktaki bir Ubuntu
sunucuda çalışan Debian 12 konteyneri ise sunucu rolünde. SSH ile bağlanıyorsun,
GraphDB ve backend orada çalışıyor, tarayıcıdan erişim SSH tüneliyle oluyor.

Bu düzen gerçek görevin birinci adımını (uzaktaki InfoLab-DEI sunucusunu
yapılandırmak) birebir taklit ediyor. Ayrıntılar ve port yönlendirme
00-kurulum.md'de.

## Nereden başlanır

Sırasıyla iki dokümanı oku:

1. [docs/00-kurulum.md](docs/00-kurulum.md), makineye ne kurulacak ve hangi
   aşamada kurulacak. Ortamın iki kısıtı burada yazılı, ikisi de sonraki
   adımları etkiliyor.
2. [docs/01-yol-haritasi.md](docs/01-yol-haritasi.md), 26 issue'nun adım adım
   işlenişi, çalışma sırası ve her adımın bitmiş sayılma ölçütü.

Sonra M0'ın ilk adımıyla, issue #1 ile başla.

## Teknoloji

Backend Node.js. Veri deposu GraphDB, JDK 17 üzerinde çalışıyor. Frontend
çatısı, SPARQL istemcisi ve Excel okuma kütüphanesi issue #3'te seçilecek.

Windows tarafına Node, Java veya GraphDB kurulmuyor. Oraya yalnızca bir SSH
istemcisi ve tercihen VS Code'un Remote - SSH eklentisi gerekiyor.

## Issue takibi

```bash
gh issue list --milestone "M0: Hazırlık & Temel"
gh issue view 1
```
