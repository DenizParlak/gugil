# google.com'a girdiğinizde arka planda neler oluyor?

<!-- wp:paragraph -->
<p>Bu makalenin amacı, IT dünyasındaki mülakatlarda sık kullanılan (en azından bir ara öyleydi) "Tarayıcıyı açıp Google'a girdiniz, bu sırada neler oldu?" sorusunu mümkün mertebe detaylı yanıtlamak.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Genellikle buradaki beklenti, olayı network bazında indirgemektir ki biraz bilgisi olanlar DNS üzerinden açıklama yapmaya çalışır, en azından verdiğim bazı eğitimlerde bu soruyu yönelttiğimde aldığım geri dönüşler bu yöndeydi. Farklı kaynaklardan edindiğim bilgileri de ekleyerek, daha geniş bir perspektiften açıklamaya çalışacağım.</p>
<!-- /wp:paragraph -->

## Tarayıcıyı açtınız...

Windows üzerinden özetlemek gerekirse:

1. Uygulama imajı RAM'e aktarılır.
2. Çekirdek, gerekli olan DLL kütüphanelerini tespit eder ve yükler.
3. Her DLL kütüphanesinin DllMain() fonksiyonu, kendi içerisindeki bağımlılık sırasına göre çalıştırılır.

Linux tarafında işler daha değişik:

1. fork() ile yeni bir süreç ve Shell'in kendisine ait bir klon oluşturulur.
2. Oluşturulan klonun dosya sistemi, blok katmanı, cache gibi bilgileri exec() tarafından okunur.
3. ELF başlatılır.
4. Kernel, .text bölümünü okur ve belleğe yükler.
5. Kernel, sırasıyla .data ve .bss segmentlerini yükler.
6. _start fonksiyonu başlatılır.
7. _init başlatılır.

## "g" tuşuna bastınız ve..

Kullandığınız tarayıcının çalışma prensibine ve ayarlarına bağlı olarak, "g" tuşuna bastığınızda tarayıcı otomatik doldurma fonksiyonu ile size önerileri getirir. Buradaki fonksiyon genellikle cache ve cookie prensibi üzerinden çalışır.

## Enter tuşuna bastınız ve..

Klavyenin USB veya sanal olması buradaki akışı biraz değiştirse de, genel olarak:

1. Klavyenin mantıksal devresine elektrik akımı gider.
2. Bu akım, keyCode değerine dönüştürülür ki "Enter" tuşuna ait değer 13'tür.
3. Klavye, sinyali IRQ'ye gönderir.
4. IRQ, klavye tuşlarına basılması, mouse hareketleri gibi donanımsal eylemlerde işlemciye sinyal gönderir ve işlemci eylemin kimliğini belirten ve çekirdek tarafından sağlanan IDT'ye göre bu sinyali işler.

## Grafik arabiriminin işleyişi

Linux tarafında açıklayacak olursak:

1. Input, "evdev" tarafından alınır ve aygıt dosyalarının bulunduğu /dev/input alt klasörüne iletilir.
2. X Server, kullanılan arayüze göre (KDE, Gnome, i3 vs.) karakter girişini odaklanılan uygulama penceresini iletir ki bu örnekte tarayıcı penceresine iletilecektir.
3. Arabirim, sistem fontunu belirler ve karakterleri adres satırına yazdırır.

Protokol belirtilmediğinde, tarayıcı girilen karakter veya kelimeyi arama motorunda arayacak şekilde çalışır. Host adı belirtildiğinde, karakterlerin geçerliliği tarayıcı tarafından kontrol edilir ki aksi durumda URL formatı Punnycode encoding ile uygulanacaktır.

## HSTS kontrolü

Tarayıcı, sadece HTTPS üzerinden gelen istekleri işleyen web sitelerin listesini HSTS üzerinden denetler. Söz konusu website listede bulunuyorsa, HTTP protokolü yerine HTTPS üzerinden istek gönderilir.

## DNS kontrolü

1. Tarayıcı, adresin cache'de olup olmadığını denetler, şayet bulunmuyorsa birçok majör işletim sisteminde "gethostbyname" fonksiyonu tetiklenir.
2. gethostbyname fonksiyonu, DNS üzerinden kontrol etmeden önce yerel dosya sistemindeki "hosts" dosyasının ilgili host adını çözümleyip çözümlemediğini kontrol eder.
3. Bu aşamada eğer cache üzerinde ve hosts dosyası üzerinde çözümleme yapılamazsa, ISP veya router düzeyinde DNS sunucusuna istek yapılır.
4. DNS sunucusunun bulunduğu subnet'e göre ARP süreci başlatılır.

## ARP

1. Ağ katmanına ARP broadcast'i gönderilmeden önce IP adresi ve MAC adresi kontrol edilir.
2. ARP'nin cache mekanizması hedef IP adresini bulduğunda ilgili kütüphane fonksiyonunu çalıştırır.
3. Bu noktada adres cache içerisinde bulunmuyorsa, yerel Route tablosundaki subnet'ler kontrol edilir.
4. OSI modelinin ikinci katmanı olan "Data Link"e ARP isteği gönderilir.
5. Burada donanım farklılığına göre farklı aşamalar devreye girer, direkt bağlantıda Router "ARP Reply" yanıtını gönderir. 6. Switch bağlantısı varsa, CAM tablosu kontrol edilerek MAC adresi ve port eşlemesi yapılır.

Ağ katmanı, varsayılan gateway veya DNS sunucusu üzerinden IP adresine eriştikten sonra UDP isteği yapılmak üzere 53. port aktif olur.

IP adresi alındıktan sonra, tarayıcı TCP socket stream'i iletmek üzere "socket" isimli sistem fonksiyonunu çağırır. Bu stream isteği ilk olarak 4. katmanda (Transport) işlenir ve bir TCP segmenti hazırlanır.

Segment, ağ katmanına iletilir. IP header bilgisi paket formatına dönüştürülür ve bu paket Link katmanına gönderilir.

Bu aşamadan sonra paketler, 0 ve 1 sinyallerinin analog sinyale dönüştürülmesi üzere modeme iletilir ve yerel subnet'i yöneten Router'a ulaştırılır. İletim boyunca her Router, IP header bölümünden sonraki varış adresi bilgisini alır uygun olan bir sonraki Router'a iletir. Burada TTL devreye girer ve her geçen Router üzerinden TTL azaltılır. Son varılan Router kuyruğunda yer yoksa veya sıfır değerine ulaşılırsa, paket drop edilir.

Bu aşamadaki TCP akışı birden çok gerçekleşir ve Handshake aşaması başlatılır.

1. Sunucu, SYN paketini oluşturmak üzere ISN'i belirler.
2. İstemci ISN'si + 1 formülü ile ACK paketini ekler.
3. İstemci, kendi tarafındaki ISN'yi artırır ve ACK'yi belirler.

## TLS Handshake

1. İstemci, sunucuya bir "Hello" mesajı göndererek sekansı başlatır. Bu mesaj, istemcinin hangi TLS sürümünü desteklediğini, desteklenen şifreleme yöntemlerini de kapsayan bir byte dizisini içerecektir.
2. Sunucu, cevap olarak SSL sertifikasını, seçtiği şifreleme yöntemini ve sunucu tarafından oluşturulan başka bir rastgele byte dizesini içeren mesajı geri gönderir.
3. İstemci, sunucunun SSL sertifikasını "sertifika yetkilisiyle" (Certificate Authority) doğrular. Bu kontrol, sunucunun söylediği "kişi" olduğunu ve istemcinin etki alanının da gerçek sahibi ile etkileşime girdiğini doğrular.
4. İstemci, rastgele bir byte dizisi gönderir. Bu dizi, ortak anahtarla şifrelenmiştir ve sadece sunucu tarafından özel anahtarla şifresi çözülebilir.
5. Sunucu bu diziyi çözer.
6. Hem istemci hem de sunucu, oluşturulan bu dizilerden oturum anahtarları oluşturur.
7. İstemci, bir oturum anahtarıyla şifrelenmiş "tamamlandı" mesajı gönderir.
8. Sunucu, bir oturum anahtarıyla şifrelenmiş "tamamlandı" mesajı gönderir.
9. İstemci ve sunucu arasındaki iletişim, oturum anahtarları üzerinden devam eder ve Handshake süreci tamamlanır.

Eğer istemci HTTP protokolünü ve tarayıcı da HTTP/1.1 kullanıyorsa, Host adının yer aldığı bir bilgi ile GET isteği gönderilir.

Sunucu, gelen HTTP isteğini metoda göre tanımlar ki bu örnekte GET olarak belirtilmiştir. Host adını tanımladıktan sonra dizin yolunu belirler, aksi belirtilmediği sürece "/" üzerinden çalışır.

Web sunucusu, hedef sunucuda google.com adına bağlı bir sanal host konfigürasyonunun doğruluğunu denetler. Şayet web sunucusu tarafında bir "rewrite" modülü kullanılıyorsa, istemciden gelen istek bu kuralı işleyecek şekilde çalıştırılır. Hedef sitenin kullandığı yorumlayıcıya göre (PHP, ASP.NET vs.) ana sayfayı yorumlar.

Sunucu, web sitesine ait kaynakları oluşturduktan sonra Parse ve Render süreci başlar.

Devamı gelecek.
