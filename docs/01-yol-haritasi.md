# Yol haritası

26 issue, beş milestone. Bu doküman issue'ları hangi sırayla ele alacağını,
her adımın neyi ürettiğini ve neyin bittiğini nasıl anlayacağını yazar.

Issue numaraları GitHub'daki sırayı takip eder ama çalışma sırası her zaman
numara sırası değil. Sırayı değiştirdiğim yerlerde gerekçesini yazdım.

## Çalışma ritmi

Aynı anda tek issue aç. Her issue için `feature/<numara>-kısa-ad` gibi bir dal
aç, işi bitir, PR aç, kendi PR'ını oku, birleştir. Bu proje pratik amaçlı
olduğu için PR sürecini atlamak cazip gelecek, atlama. Gerçek tez çalışmasında
denetlenecek olan şey de bu iz.

Her milestone'un sonunda repo çalışır durumda olmalı. M1 bittiğinde elinde
prosedür dokümanları olur, M2 bittiğinde ayakta bir sistem, M3 bittiğinde
kullanılabilir bir arayüz, M4 bittiğinde içi gerçek veriyle dolu bir graf.

Kodu Windows makinenden değil, konteynerin içinden yazacaksın. VS Code'un
Remote - SSH eklentisiyle bağlanırsan dosyalar doğrudan konteynerde açılır ve
kopyalama derdin olmaz. Kurulum, port yönlendirme ve iki makine arasındaki iş
bölümü ayrı dokümanda: [00-kurulum.md](00-kurulum.md).

## M0: Hazırlık ve temel (#1 - #4)

Kod yazmadan önce neyi modellediğini bilmen gerekiyor. Bu milestone'un tamamı
yazı işi ve muhtemelen en çok atlamak isteyeceğin kısım. Atlarsan M3'te
ontolojiyi baştan yazmak zorunda kalırsın.

### Adım 1, issue #1: proje vizyonu, kapsam ve mimari

Ne yapacağını ve ne yapmayacağını yaz. Kapsam dışı bıraktıklarını açıkça
listele, çünkü bu proje kolayca büyüyüp bir ITSM ürününe dönüşebilir.

Mimari bölümünde en az şunlar olsun: bileşenler (GraphDB, backend API,
frontend, ETL), aralarındaki oklar, ve verinin Excel'den grafa nasıl aktığı.

Çıktı: `docs/02-mimari.md`.
Bitti sayılır: bir yabancı okuyup sistemin ne yaptığını anlatabiliyorsa.

### Adım 2, issue #2: domain ontolojisi

RDF/RDFS şeması. Modelleyeceğin varlıklar kabaca şunlar: laboratuvar, donanım
varlığı, yazılım, lisans, servis, personel. Aralarındaki ilişkiler ve her
sınıfın özellikleri.

Namespace'i baştan seç ve bir daha değiştirme. Lisans son kullanma tarihi gibi
tarih alanlarını `xsd:date` olarak tiple, çünkü #18'deki son kullanma panosu
bunlar üzerinde karşılaştırma yapacak.

Çıktı: `ontology/dei-infolab.ttl` ve şemayı açıklayan kısa bir okuma notu.
Bitti sayılır: Turtle dosyası sözdizimi hatasız yükleniyorsa.

### Adım 3, issue #3: teknoloji yığını kararı ve repo iskeleti

Backend Node.js olacak, bu belli. Karar vermen gerekenler: HTTP çatısı, SPARQL
istemcisi, frontend çatısı, Excel okuma kütüphanesi, RDF üretme kütüphanesi.

Her seçim için bir cümle gerekçe yaz. Alternatifini de yaz, çünkü M4'te ETL
kütüphanesi seni yarı yolda bırakırsa geri dönüp neden onu seçtiğini
hatırlaman gerekecek.

Aynı issue'da dizin yapısını da sabitle. Sonradan taşımak PR geçmişini
okunmaz hale getirir.

Çıktı: `docs/03-teknoloji-secimi.md`, boş `package.json` ve dizin iskeleti.
Bitti sayılır: `npm install` hatasız çalışıyorsa.

### Adım 4, issue #4: SPARQL sorgu kataloğu

Ontoloji hazır olmadan bu adıma başlama. M3'teki her ekran için bir sorgu yaz:
varlık listesi, filtreli arama, varlık detayı, lab detayı, servis kataloğu,
süresi dolan lisanslar.

Sorguları dosya olarak sakla, koda gömme. Backend bunları dosyadan okusun.
Böylece #21'deki SPARQL konsolunu yazarken aynı dosyaları tekrar
kullanabilirsin.

Çıktı: `queries/*.rq` ve her sorgunun ne döndürdüğünü anlatan bir dizin
dosyası.
Bitti sayılır: her sorgu boş bir grafta sözdizimi hatası vermeden çalışıyorsa.

## M1: Sunucu konfigürasyonu (#5 - #9)

Gerçek görevin birinci adımının karşılığı. Burada ürettiğin şey çalışan sistem
değil, tekrarlanabilir prosedür. Bir sunucu sıfırlansa bu dokümanlarla tekrar
kurulabilmeli.

### Adım 5, issue #5: sunucu temel kurulumu

İşletim sistemi seçimi, kullanıcı ve grup düzeni, SSH yapılandırması, güvenlik
duvarı, güncelleme politikası. Servisleri root ile çalıştırma, uygulama için
ayrı bir sistem kullanıcısı tanımla.

Bu issue'nun güzel tarafı şu: hâlihazırda bağlandığın konteyner tam olarak bu
işin öznesi. Mevcut `sshd_config` durumu `PermitRootLogin no` (iyi) ve
`PasswordAuthentication yes` (sertleştirilecek). Windows tarafında
`ssh-keygen` ile anahtar üret, açık anahtarı konteynerdeki
`~/.ssh/authorized_keys` dosyasına ekle, anahtarla girebildiğini doğrula, sonra
parola girişini kapat. Bu prosedürü yazarken kendi bağlantını kesme riskin var,
o yüzden değişikliği yapmadan önce ikinci bir SSH oturumu açık tut.

systemd çalışmadığı için prosedürün bir kısmını burada doğrulayamazsın.
Yazdığın komutların hangilerini gerçekten test ettiğini, hangilerini yalnızca
yazdığını dokümanda ayır.

Çıktı: `docs/runbook/01-sunucu-kurulum.md`.

### Adım 6, issue #6: Java çalışma zamanı ve GraphDB kurulum prosedürü

JDK 17 bu makinede zaten kurulu, dolayısıyla Java kısmı doğrulama ve
sürüm sabitleme işi. GraphDB kısmı elle indirme gerektiriyor, adımları
00-kurulum.md'de yazdım, buraya sunucu bağlamıyla taşı: hangi dizine kurulacak,
hangi kullanıcı çalıştıracak, bellek ayarları ne olacak.

Çıktı: `docs/runbook/02-graphdb-kurulum.md`.

### Adım 7, issue #7: reverse proxy, portlar ve TLS

nginx GraphDB'yi (7200), backend API'yi (3000) ve frontend'i tek bir port
arkasına alacak. Port haritası 00-kurulum.md'de sabitlendi, burada ona sadık
kal ve `.ssh/config` dosyandaki `LocalForward` satırlarıyla aynı tuttuğundan
emin ol.

Konteynerin dışarıya açık tek portu SSH olduğu için tarayıcıdan yaptığın her
test SSH tüneli üzerinden geçecek. Bir sayfa açılmadığında önce tünelin ayakta
olup olmadığına bak, nginx'i kurcalamadan önce.

GraphDB'nin workbench arayüzünü dışarıya açma. Yalnızca backend'in eriştiği
bir iç servis olarak kalsın, dışarıya yalnızca kendi API'ni aç.

TLS'i planla ama gerçek sertifika alamayacaksın: konteynerin dışarıya bakan bir
alan adı yok, yalnızca SSH portu açık. Sertifika üretme ve yenileme adımlarını
prosedür olarak yaz, testi kendi imzaladığın bir sertifikayla yap.

Çıktı: `docs/runbook/03-reverse-proxy.md` ve `deploy/nginx/` altında örnek
yapılandırma.

### Adım 8, issue #8: ortam değişkenleri, secret ve config yönetimi

Bu issue `priority:high` etiketli ve haklı olarak öyle. GraphDB kimlik bilgisi,
backoffice oturum anahtarı, API adresleri: hepsi ortam değişkeninden gelsin.

Repoda `.env.example` tut, `.env` tutma. `.gitignore`'a `.env` satırını bu
adımda ekle, sonra değil. Bir kez commit'lenmiş secret'ı geçmişten silmek
ayrı bir uğraş.

Çıktı: `.env.example`, `.gitignore` güncellemesi, `docs/runbook/04-config.md`.
Bitti sayılır: `git log -p` çıktısında hiçbir gerçek parola geçmiyorsa.

### Adım 9, issue #9: yedekleme ve geri yükleme

Yedeklenecek iki şey var: GraphDB repository içeriği ve yapılandırma dosyaları.
Yedek almak kolay kısım, asıl iş geri yükleyip çalıştığını göstermek.

cron bu makinede systemd altında çalışmadığı için zamanlanmış görevi
`service cron start` ile elle ayağa kaldırman gerekecek. Alternatif olarak
yedek betiğini elle çalıştırıp geri yükleme tatbikatını yap, zamanlama kısmını
prosedür olarak yaz.

Çıktı: `scripts/backup.sh`, `scripts/restore.sh`,
`docs/runbook/05-yedekleme.md`.
Bitti sayılır: repository'yi silip yedekten geri yükleyebiliyorsan.

## M2: Sistem deployment (#10 - #15)

Gerçek görevin ikinci adımı. M1'de yazdığın prosedürleri uygula ve sistemi
ayağa kaldır.

### Adım 10, issue #10: GraphDB deployment ve repository

GraphDB'yi kur, çalıştır, bir repository oluştur ve M0'daki ontolojiyi yükle.

Repository adını ve ayarlarını dokümana yaz. Backend'in bağlanacağı endpoint
adresi buradan çıkacak.

Bitti sayılır: `curl` ile bir SPARQL sorgusu atıp ontolojiden sınıf listesini
alabiliyorsan.

### Adım 11, issue #11: backend API deployment

M0'daki sorgu kataloğunu okuyup HTTP üzerinden sunan servis. Bu adımda tüm
uç noktaları yazman gerekmiyor, bir sağlık kontrolü ve bir gerçek sorgu
yeterli. Asıl uç noktalar M3'te gelecek.

Bitti sayılır: servis nginx arkasından yanıt veriyorsa.

### Adım 12, issue #12: frontend deployment

Statik derleme çıktısını nginx'ten sun. Bu aşamada arayüzün içeriği önemli
değil, backend'e ulaşabildiğini gösteren tek bir sayfa yeterli.

Bitti sayılır: SSH tüneli açıkken Windows tarayıcında `http://localhost:8080`
açılıyor ve sayfa backend'den veri çekebiliyorsa.

### Adım 13, issue #13: Omega-S analoğu

Bu issue `needs-info` etiketli ve öyle kalması doğru. Gerçek görevdeki
"Omega S"in ne olduğu bilinmiyor. Pratikte ayrı bir veri ve entegrasyon
platformu olarak modellendi.

Tavsiyem: bu adımı M2'nin sonuna bırak ve dar tut. Dışarıdan veri alan tek bir
entegrasyon noktası kur, örneğin bir CSV veya JSON kaynağından periyodik
çekim. Omega S'in gerçek tanımı öğrenildiğinde bu adımı yeniden yazarsın.
Şimdiden büyük bir entegrasyon katmanı tasarlarsan büyük ihtimalle boşa
gidecek.

Bu issue'yu kapatırken hangi varsayımla ilerlediğini yorum olarak yaz.

### Adım 14, issue #14: süreç yönetimi ve sağlık kontrolü

Burada bir sapma var, dokümanla. Issue "systemd/pm2" diyor ama bu makinede
systemd çalışmıyor. pm2 kullan.

pm2 ile backend'i ve gerekirse GraphDB'yi yönet, makine yeniden başladığında
ayağa kalkacak şekilde ayarla, `/health` uç noktası ekle ve pm2'nin bunu
izlemesini sağla.

systemd unit dosyasını yine de yaz ve `deploy/systemd/` altında tut. Gerçek
InfoLab-DEI sunucusunda kullanılacak olan o.

Bitti sayılır: `pm2 restart` sonrası sistem kendini toparlıyorsa.

### Adım 15, issue #15: smoke test ve deployment doğrulama

M2'nin kapanış adımı. Sıfırdan kurulumdan sonra sırayla kontrol edilecek
maddelerin listesi ve bu listeyi çalıştıran bir betik.

Kontrol listesine en az şunları koy: beklenen portlar dinleniyor mu (`ss -tlnp`,
iproute2 kurulduktan sonra), GraphDB repository'si yanıt veriyor mu, backend
sağlık kontrolü geçiyor mu, nginx her iki yolu da doğru yere yönlendiriyor mu.

Çıktı: `docs/runbook/06-smoke-test.md` ve `scripts/smoke-test.sh`.
Bitti sayılır: betik tek komutla çalışıp bütün kontrolleri geçiyorsa.

## M3: Yazılım güncellemesi (#16 - #21)

Gerçek görevin üçüncü adımı ve projenin en uzun kısmı. İki ayrı arayüz var:
son kullanıcı tarafı okuma odaklı, backoffice tarafı yazma odaklı.

Numara sırasını burada bozuyorum. Backoffice kimlik doğrulaması (#19) CRUD'dan
(#20) önce gelmeli, yoksa yetkisiz yazma yapan bir arayüz ortaya çıkar ve
sonradan güvenlik eklemek her uç noktayı tekrar elden geçirmek demektir.

Önerilen sıra: #16, #17, #18, #19, #20, #21.

### Adım 16, issue #16: varlık listeleme, filtre ve arama

Son kullanıcı tarafının ilk ekranı. Sayfalama, tipe ve laboratuvara göre
filtre, metin araması.

SPARQL tarafında sayfalama `LIMIT` ve `OFFSET` ile yapılır ama toplam sayı
için ayrı bir `COUNT` sorgusu gerekir. Bunu baştan planla.

### Adım 17, issue #17: varlık ve laboratuvar detay sayfası

Listeden tıklanınca açılan detay. Varlığın özellikleri, bağlı olduğu
laboratuvar, üzerindeki yazılımlar ve lisanslar.

Detay sorgusu ilişkileri de getirdiği için #16'daki liste sorgusundan ayrı bir
sorgu olacak. İkisini tek sorguda birleştirmeye çalışma.

### Adım 18, issue #18: servis kataloğu ve lisans son kullanma panosu

Servislerin listesi ve süresi dolmuş veya dolmak üzere olan lisansların
panosu. Eşik değeri (örneğin 30 gün) yapılandırılabilir olsun.

Bu ekran ontolojideki tarih tiplemesine bağımlı. `xsd:date` olarak
tiplenmemişse karşılaştırma çalışmaz ve #2'ye geri dönmen gerekir.

### Adım 19, issue #19: backoffice kimlik doğrulama ve yetkilendirme

`priority:high`. En az iki rol tanımla: okuyabilen ve yazabilen. Oturum
yönetimi ve parola saklama kararlarını burada ver.

Kimlik bilgilerini #8'deki config mekanizmasından oku.

Bitti sayılır: yetkisiz istek 401 veya 403 dönüyorsa ve bunu gösteren bir test
varsa.

### Adım 20, issue #20: backoffice CRUD

Varlık, yazılım, lisans ve personel kayıtları için ekleme, düzenleme, silme.

SPARQL'da güncelleme `DELETE` ve `INSERT` çiftiyle yapılır ve yanlış yazılmış
bir `DELETE WHERE` beklediğinden fazlasını siler. Her yazma işleminden önce
etkilenecek üçlü sayısını sayan bir sorgu çalıştır, sonra sil. #9'daki
yedekleme prosedürü ilk kez burada işine yarayacak.

### Adım 21, issue #21: SPARQL konsolu ve audit log

Yönetici için serbest sorgu ekranı ve yapılan değişikliklerin kaydı.

Konsolu yalnızca okuma sorgularına izin verecek şekilde sınırla. `INSERT`,
`DELETE`, `DROP`, `CLEAR` içeren istekleri reddet. Yazma işlemleri #20'deki
denetlenen uç noktalardan geçsin.

Audit log kim, ne zaman, neyi değiştirdi bilgisini tutsun. Log'u grafın
içinde ayrı bir named graph'ta tutmak da mümkün, ayrı bir dosyada tutmak da.
Kararı yaz.

## M4: Veri oluşturma (#22 - #26)

Gerçek görevin dördüncü adımı. Bölümün elindeki Excel dosyalarından grafı
doldurmak.

### Adım 22, issue #22: Excel şablonları ve ontoloji eşlemesi

Kod yazmadan önce şablonu tanımla. Her sütun ontolojideki hangi sınıfa veya
özelliğe karşılık geliyor, hangi sütunlar zorunlu, kimlikler nasıl üretilecek.

Kimlik üretimi kritik. Aynı satır iki kez yüklendiğinde aynı URI üretilmeli,
yoksa graf kopyalarla dolar. Doğal bir anahtar varsa (envanter numarası gibi)
onu kullan.

Çıktı: `docs/04-excel-semasi.md` ve `templates/` altında örnek dosyalar.

### Adım 23, issue #23: Excel'den RDF'e ETL

Şablonu okuyup Turtle üreten Node betiği. Dönüşümü ayrı, yüklemeyi ayrı tut:
önce dosyaya yaz, gözle kontrol et, sonra GraphDB'ye gönder. Tek adımda
doğrudan yüklersen hatayı ancak graf bozulduktan sonra fark edersin.

Bitti sayılır: örnek Excel dosyasından üretilen Turtle sözdizimi hatasız
yükleniyorsa.

### Adım 24, issue #24: backoffice'ten tetikleme ve tekilleştirme

ETL'i komut satırından çıkarıp arayüze taşı. Dosya yükleme, çalıştırma,
sonucu gösterme.

Tekilleştirme #22'deki kimlik üretimine dayanır. Aynı dosya iki kez
yüklendiğinde graftaki üçlü sayısı artmamalı. Bunu bir testle sabitle.

### Adım 25, issue #25: veri kalite kontrolleri ve yükleme raporu

Yüklemeden önce çalışan kontroller: zorunlu alan boş mu, tarih formatı doğru
mu, referans verilen laboratuvar grafta var mı.

Kontrolden geçmeyen satırları yüklemeyi durdurmadan atla ve raporda listele.
Yükleme sonunda kaç satır işlendi, kaçı atlandı, kaç üçlü eklendi bilgisini
göster.

### Adım 26, issue #26: uçtan uca test

Gerçekçi hacimde örnek veriyle bütün akışı çalıştır: Excel yükle, ETL çalışsın,
son kullanıcı arayüzünde veriyi gör, backoffice'ten bir kayıt düzenle,
değişikliğin audit log'a düştüğünü doğrula.

Bu adım #15'teki smoke test'ten farklı. Orada sistemin ayakta olduğunu
kontrol ediyordun, burada iş akışının doğru çalıştığını.

Bitti sayılır: senaryonun tamamı elle müdahale olmadan geçiyorsa.

## Açık konular

Omega S'in gerçekte ne olduğu bilinmiyor (#13). Pratikte varsayımla
ilerleniyor. Gerçek tanım öğrenildiğinde #13 ve muhtemelen #11 etkilenecek.

Konteynerde systemd yok, dolayısıyla #14'ün systemd tarafı yazılabilir ama
doğrulanamaz. Gerçek sunucuda ilk yapılacak iş bu prosedürü test etmek.

Konteynerin dışarıya açık tek portu SSH. Bu yüzden #7'deki TLS işi ve #12'deki
frontend erişimi tünel üzerinden test edilecek, gerçek bir alan adıyla değil.
Gerçek InfoLab-DEI sunucusuna geçildiğinde bu iki adım yeniden ele alınmalı.

GraphDB indirmesi kayıt formuna bağlı. Erişim sağlanamazsa Apache Jena Fuseki
ile devam etme planı 00-kurulum.md'de yazılı, ama bu bir sapma ve gerçek
görevle uyuşmuyor.
