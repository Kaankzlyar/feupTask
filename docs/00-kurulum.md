# Kurulum

Bu projede iki makine var ve hangisine ne kuracağın farklı. Karıştırırsan
saatler kaybedersin, o yüzden önce topolojiyi netleştirelim.

Tespit tarihi: 2026-08-24.

## Elindeki iki makine

```
Windows PC                Ubuntu sunucu            Debian 12 konteyner
(iş istasyonu)     SSH    (Docker host)            (sunucu rolü)
88.230.1.9        ----->  kernel 7.0.0-28-generic  172.17.0.2
                                                   hostname 4071f78a27e2
```

Windows makinen iş istasyonu. Kod yazdığın, tarayıcıyı açtığın, SSH ile
bağlandığın yer.

Debian 12 konteyneri sunucu rolünde. GraphDB, backend, nginx burada çalışacak.
Bu bir WSL2 dağıtımı değil ve Docker Desktop da değil: `/mnt/c` yok,
`WSL_DISTRO_NAME` boş, çekirdek Ubuntu derlemesi bir sunucu çekirdeği. Yani
konteyner senin bilgisayarında değil, uzaktaki ayrı bir Linux makinede
çalışıyor ve oraya internet üzerinden SSH ile giriyorsun.

Bu ayrım aslında projeye yarıyor. Gerçek tez görevinin birinci adımı zaten
"InfoLab-DEI sunucusunu yapılandırmak", yani uzaktaki bir makineyi SSH ile
kurmak. Elindeki düzen bunu birebir taklit ediyor. Windows tarafına bir şey
kurup sunucu tarafını atlamak, pratik projenin asıl öğreteceği şeyi atlamak
olur.

## Windows tarafına ne kurulacak

Kısa liste. Windows'a Node, Java veya GraphDB kurmana gerek yok, hepsi
konteynerde çalışacak.

| Araç | Gerekli mi | Not |
|---|---|---|
| OpenSSH istemcisi | evet | Windows 10 (1809+) ve 11'de zaten kurulu |
| VS Code + Remote - SSH eklentisi | önerilir | dosyaları konteynerde açıp düzenlersin |
| Windows Terminal | isteğe bağlı | PowerShell penceresinden rahat |
| Tarayıcı | evet | zaten var |
| Git for Windows | hayır | git konteynerde kurulu, orada çalış |
| Node.js for Windows | hayır | aynı sebep |

OpenSSH'in kurulu olduğunu PowerShell'de doğrula:

```powershell
ssh -V
```

Sürüm bilgisi dönüyorsa hazırsın. Dönmüyorsa Ayarlar altında İsteğe Bağlı
Özellikler bölümünden OpenSSH Client'ı ekle.

VS Code'da yapman gereken tek şey Remote - SSH eklentisini kurmak. Sonra
komut paletinden sunucuya bağlanınca dosyaları doğrudan konteynerde
düzenlersin, kopyalama yok, senkronizasyon yok.

## Port yönlendirme

Bu bölüm atlanmaz. Konteynerin dışarıya açık tek portu SSH. GraphDB'nin
arayüzü, backend'in API'si ve frontend Windows tarayıcından doğrudan
erişilebilir değil. SSH tüneli açman gerekiyor.

Kullanacağımız port haritası (bu haritayı issue #7'de de aynen kullan):

| Port | Ne çalışıyor | Kim erişecek |
|---|---|---|
| 7200 | GraphDB workbench | yalnız sen, yönetim için |
| 3000 | backend API | nginx, ve geliştirme sırasında sen |
| 80 | nginx (frontend ve API'nin önü) | sistemin asıl kapısı |

PowerShell'den tek seferlik bağlantı:

```powershell
ssh -L 7200:localhost:7200 -L 3000:localhost:3000 -L 8080:localhost:80 kutay@SUNUCU -p PORT
```

`SUNUCU` ve `PORT` yerine hâlihazırda bağlanırken kullandığın adres ve portu
yaz. Bu tünel açıkken Windows tarayıcında `http://localhost:7200` GraphDB'yi,
`http://localhost:8080` de nginx'i açar.

Her seferinde bu uzun komutu yazmamak için Windows'ta
`C:\Users\<kullanıcı>\.ssh\config` dosyasına şunu ekle:

```
Host feup
    HostName SUNUCU
    Port PORT
    User kutay
    LocalForward 7200 localhost:7200
    LocalForward 3000 localhost:3000
    LocalForward 8080 localhost:80
```

Sonrası `ssh feup` demekten ibaret. VS Code Remote - SSH de bu config
dosyasını okur, aynı Host girdisini seçtiğinde tünelleri o da kurar.

## Konteynerde şu an hazır olanlar

Ortam: Debian 12 (bookworm), x86_64, 125 GB RAM, `/workspace` altında 648 GB
boş alan, parolasız sudo mevcut.

| Araç | Sürüm | Nerede lazım |
|---|---|---|
| git | 2.39.5 | her yerde |
| Node.js | 22.22.2 | backend, frontend, ETL |
| npm | 10.9.7 | paket yönetimi |
| GitHub CLI (`gh`) | 2.94.0 | issue takibi |
| OpenJDK (JDK, javac dahil) | 17.0.19 | GraphDB'nin çalışma zamanı |
| curl | 7.88.1 | API ve SPARQL endpoint testleri |
| unzip, tar | - | GraphDB dağıtımını açmak, yedek almak |

GraphDB 10.x Java 11 veya 17 ister. Kurulu JDK 17 bu şartı karşılıyor, ayrıca
Java kurmana gerek yok.

## Konteynerde eksik olanlar

| Araç | Ne zaman gerekli | Kurulum |
|---|---|---|
| jq | hemen (SPARQL JSON çıktısını okumak için) | `sudo apt-get install -y jq` |
| iproute2 (`ss`) | M2 (#15, port doğrulama) | `sudo apt-get install -y iproute2` |
| nginx | M1 (#7), M2 (#12) | `sudo apt-get install -y nginx` |
| rsync | M1 (#9) | `sudo apt-get install -y rsync` |
| cron | M1 (#9) | `sudo apt-get install -y cron` |
| pm2 | M2 (#14) | `npm install -g pm2` |
| GraphDB | M1 sonu / M2 başı | aşağıda, elle indirme |

Docker ve Python 3 kurulu değil, ikisine de ihtiyacın olmayacak.

## Konteynerin iki kısıtı

Birincisi: konteynerde systemd çalışmıyor. PID 1 `sshd` ve
`systemctl is-system-running` komutu `offline` dönüyor. Pratik sonucu şu:
issue #14'teki süreç yönetimini systemd unit dosyalarıyla yapamazsın, pm2
kullanacaksın. systemd prosedürünü yine de yaz (gerçek InfoLab-DEI sunucusunda
systemd olacak), ama burada çalıştırıp doğrulayamayacağını issue'ya not düş.

Aynı sebeple servisler `systemctl` yerine `service` ile başlar:

```bash
sudo service nginx start
sudo service cron start
```

İkincisi: Python 3 yok. Excel'den RDF üreten ETL'i (#23, #24) Node ile
yazarsan hiç Python kurman gerekmez. Hangi kütüphaneleri kullanacağına issue
#3'te karar vereceksin.

## Adım adım kurulum

### Şimdi yap

Windows tarafında `ssh -V` çalıştığını doğrula ve `.ssh\config` dosyasına
yukarıdaki `feup` girdisini ekle. Konteynerde ise:

```bash
sudo apt-get update
sudo apt-get install -y jq iproute2
```

`gh` token'ında `project` yetkisi yok (mevcut kapsamlar: `gist`, `read:org`,
`repo`). Issue'ları Projects board'una komut satırından eklemek istiyorsan:

```bash
gh auth refresh -s project
```

Board'u kullanmayacaksan atla, issue listesi tek başına yeterli.

### M1 sırasında

```bash
sudo apt-get install -y nginx rsync cron
npm install -g pm2
sudo service nginx start
```

Tünel açıkken Windows tarayıcından `http://localhost:8080` adresinin nginx
karşılama sayfasını gösterdiğini doğrula. Göstermiyorsa sorun tünelde, nginx'te
değil, önce oraya bak.

### GraphDB kurulumu

GraphDB apt deposunda yok, elle indirmen gerekiyor. Ontotext'in indirme
sayfasında kayıt formu var ve bağlantı e-postayla geliyor.

İndirmeyi Windows'ta yapacaksın, kurulum konteynerde olacak. Dosyayı karşıya
geçirmek için PowerShell'den:

```powershell
scp -P PORT graphdb-dist.zip kutay@SUNUCU:/workspace/
```

`.ssh\config` girdisini eklediysen `scp graphdb-dist.zip feup:/workspace/`
yeterli. Sonra konteynerde:

```bash
mkdir -p /workspace/opt
unzip /workspace/graphdb-*-dist.zip -d /workspace/opt/
/workspace/opt/graphdb-*/bin/graphdb -d
curl -sS http://localhost:7200/rest/repositories | jq .
```

Son komut boş bir dizi (`[]`) döndürüyorsa GraphDB ayakta ve henüz repository
yok demektir. Repository oluşturmak issue #10'un işi.

Tarayıcıdan workbench arayüzünü görmek istiyorsan tünel açıkken
`http://localhost:7200` adresini aç.

Kayıt formu veya indirme sana kapalıysa yedek plan Apache Jena Fuseki. Kayıt
istemez, doğrudan indirilir, aynı JDK 17 üzerinde çalışır ve SPARQL 1.1
protokolünü konuşur. Backend kodun SPARQL endpoint'ine HTTP ile konuştuğu
sürece ikisi arasında geçiş birkaç satırlık config değişikliği olur. Gerçek
tez görevi GraphDB'yi şart koştuğu için Fuseki'yi yalnızca tıkanırsan kullan
ve bu sapmayı issue #10'a not et.

### M3 ve M4 sırasında

Bu aşamalarda kuracakların npm paketleri, hepsi proje dizinine iner ve
`package.json` kaydı tutar. Hangi paketler olduğu issue #3'teki teknoloji
yığını kararına bağlı, o karar verilmeden liste yazma.

## Doğrulama

Kurulumun bittiğini şu dört kontrolle anlarsın. İlk üçü konteynerde, sonuncusu
Windows tarayıcında:

```bash
node -v                                        # 22.x
java -version                                  # 17.x
curl -sS http://localhost:7200/rest/repositories | jq .
```

Sonra tünel açıkken tarayıcıdan `http://localhost:7200` GraphDB workbench'i
açıyorsa M2'ye geçebilirsin.
