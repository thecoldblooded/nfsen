# Pardus ve NfSen ile Yönlendirici Netflow Kayıtlarının İncelenmesi

>Umut DOĞAN 
udogan@etu.edu.tr TOBB ETÜ Üniversitesi
 
###### Ağ yönlendirici cihazlarının oluşturduğu ve üzerinden geçen trafiğe ait izleri barındıran NetFlow kayıtlarının depolanması ve incelenebilmesi için yapılması gerekenleri adım adım anlatacağım. Bu kayıtların tutulması ağımızdan dışarı çıkan trafiğin kaynak ve hedeflerinin belirlenmesi konusundaki hukuki sorumlulukları karşılamada yardımcı olacaktır. Ayrıca kurulumunu anlatacağım NfSen ile de bu kayıtların detaylı analizini bir arayüz yardımıyla yaparak ağ trafiğinizi inceleyebilirsiniz.

### Giriş
___
Adım adım yapılması gerekenlere geçmeden önce NetFlow datasının ne olduğunu açıklayalım:

Cisco tarafından geliştirilmiş açık bir protokol olan NetFlow, IP trafiği kayıtlarının toplanmasını sağlar. 
NetFlow kayıtları 5 temel içerikten oluşur: __Kaynak IP adresi__, __hedef IP adresi__, __kaynak kapısı (PORT)__ ve __hedef kapısı (PORT)__ ve __protokol__.

__`Örnek Kayıt:`__

| Sif | SrcIPaddress |	DIf |	DstIPaddress | Pr	|	SrcP |	DstP |	Pkts |	Octets | B/Pk | Ts |	Fl  |
| :-------------: | :-------------: | :-------------: | :-------------: | :-------------: | :-------------: | :-------------: | :-------------: | :-------------: | :-------------: | :-------------: | :-------------: |
| 00c7 | 193.140.21.18 |	00fd | 88.239.42.161 |	06 | 50 |	446 |	1 |	464 |	464 |	00 | 19  |

Burada *Sif* trafiğin geldiği arayüzün (interface) yönlendirici üzerindeki tanımlayıcısıdır. Aynı şekilde *Dif* de trafiğin yönlendirildiği arayüzün tanımlayıcısıdır. *Pr* trafiğin kullandığı protokolü göstermektedir. ( 6 - > TCP) Diğer alanlarda IP başlık bilgisinde elde edilen bilgilerdir. Ayrıca trafiğin başlangıç ve bitiş zamanlarına da NetFlow kayıtlarından ulaşılabilir.

##### A)	Netflow Kayıtlarının Oluşturulması ve Bir PC'ye Yönlendirilmesi İçin Yapılması Gerekenler
Yönlendirici cihaza bağlanıp enable moduna geçtikten sonra;

```javascript
Router#conf t
Router(config)#
Router(config)#ip flow-export version 5 origin-as
Router(config)#ip flow-export destination 192.168.1.10 9996
```

komutları ile NetFlow beşinci nesil flow kayıtlarının 192.168.1.10 nolu makinanın 9996 nolu kapısına yönlendirilmesini sağladım. Şimdi hangi arayüzlerden geçen trafiğin kayıt altına alınacağı belirtmek için ilgili arayüzün altına gidip, 

```javascript
Router(config)#interface FastEthernet0/0
Router(config-if)#ip route-cache flow
```

__`Önemli Not:`__ Bu analizlerin yapılacağı bir çok ağda yönlendirici ile kayıtların tutulacağı makine (örneğimizde 192.168.1.10 IP'li) arasında ateş duvarı (firewall) olduğunu düşünmekteyiz. Bu durum sizin ağınız için de geçerli ise, yönlendiricinizin iç ağa bakan arayüzünün IP'sinden kaynaklı ve hedefi kayıtların tutulacağı makinanın yönlendirici ayarlarında girilen kapısı olan UDP trafiğine izin vermeniz gerekmektedir. Bu örnek için yönlendiricinin iç ağa bakan arayüz Ipsi kaynaklı __192.168.1.10:9996__ hedefli trafiğin geçişine izin verilmelidir.

##### B)	Netflow Kayıtlarının Saklanması ve Analizi İçin NfSen kurulumu
Bu kısımda NfSen uygulamasını Pardus üzerine kurduk. Detaylarını aşağıda anlatacağım. Benzer şekillerde herhangi bir Linux dağıtımı ya da BSD işletim sisteminde de rahatlıkla kullanabilirsiniz. (FreeBSD'de nfsen bir port olarak bulunmakta).

##### 1-	Pardus Kurulumu

ftp://ftp.pardus.org.tr/pub/pardus/kurulan/2007.1 adresinden edineceğiniz kaynak kod ile Pardus kurumunu standart şekilde yapmanız yeterli olacaktır.
Sorun yaşamanız durumunda http://www.pardus.org.tr/belgeler/kurulum_nasil.html adresindeki belgeye başvurabilirsiniz.

Pardus Kurumsal 5 sürümünü www.pardus.org.tr adresinden temin edebilirsiniz. DVD oluşturmak için K3b, Brasero, USB oluşturmak için Unetbootin, ISO to USB uygulamalarını kullanabilirsiniz. Bu uygulamalar hakkında bilgi için http://www.pardus.org.tr/belgeler adresindeki Pardus İşletim Sistemi kitabından yardım alabilirsiniz. Kitap'ta Uygulamalar ve Tanıtımları altında Disk Yazma Araçları başlığında bu konuya değinilmiştir.

| Başlamak için kurulum ortamını(DVD-USB) bilgisayarınıza yerleştiriniz. Ardından F2-F8-F12 veya Delete tuşlarından biri ile BİOS ayarlarına girerek ( bios ayarlarına giriş tuşu farklılık gösterebilir ) açılış seçeneklerinden yerleştirdiğiniz DVD ya da USB disk'i ilk sıraya getirerek kaydedip çıkınız. |
| :-------------: |
| ![alt text](https://www.pardus.org.tr/documents/10180/830384338/giris.png/6cbe14f3-1720-4e0f-905b-8179eb50d310?t=1429875245537 "Logo Title Text 1")       |

Yeniden başlayan bilgisayarınız canlı sistem üzerinden açılacaktır. Bu özellik sayesinde kurulum öncesinde, sonrasında ve kurulum esnasında Pardus'un bilgisayarınızdaki performansını ve uygulamalarını test edebilirsiniz.

| 1- Kurulum için ekranda ki Pardus'u Yükle simgesine tıklayarak kuruluma başlayabilirsiniz.  |
| :-------------: |
| ![alt text](https://www.pardus.org.tr/documents/10180/830384338/masaustu.png/ccbf5c16-5079-4a43-a700-cc9f96bb491e?t=1429866782000 "Logo Title Text 1")       |

| 2- Sistem dilini seçerek ilerleyiniz.  |
| :-------------: |
| ![alt text](https://www.pardus.org.tr/documents/10180/830384338/dil.png/8abf3a3a-b5f3-409c-8307-b678ac1da1d8?t=1429866717000 "Logo Title Text 1")       |

| 3- Zaman dilimini seçerek devam ediniz.  |
| :-------------: |
| ![alt text](https://www.pardus.org.tr/documents/10180/830384338/zaman.png/d23d835a-2285-4d20-b16f-a6895823dd56?t=1429866812000 "Logo Title Text 1")       |

| 4- Klavye düzeni belirleyerek devam ilerleyiniz. |
| :-------------: |
| ![alt text](https://www.pardus.org.tr/documents/10180/830384338/klavye.png/d46b26b5-560c-4295-8cf6-ac3899be0884?t=1429866761000 "Logo Title Text 1")       |

| 5- Kullanıcı resminizi, kullanıcı ve makine adınızı belirleyiniz. |
| :-------------: |
| ![alt text](https://www.pardus.org.tr/documents/10180/830384338/bilgi.png/a1388f5e-e8c9-4c60-8ab3-990b18037747?t=1429866694000 "Logo Title Text 1")       |

Bu ekranda belirleyeceğiniz parola bilgisayarınızın hem root (Yönetici) parolası hemde tanımlamış olduğunuz kullanıcının parolası olacaktır. 

| 6- Açılan pencerede bilgisayarınızda bulunan fiziksel birimlerin listesi gelecektir.   |
| :-------------: |
| ![alt text](https://www.pardus.org.tr/documents/10180/830384338/fdisk.png/dc450806-1751-4e94-8c00-1442f0d056a2?t=1429866727000 "Logo Title Text 1")       |

Hard disk, USB disk, harici disk ve SSD diskler boyutları ile birlikte bu ekranda listelenir.  

| 7- Seçmiş olduğunuz fiziksel diskin mantıksal alt birimleri bu ekranda görüntülenir. Yeni bir birim(ler) oluşturmak veya var olan birimleri düzenlemek için ”Bölümleri düzenle” seçeneğini kullanabilirsiniz. Fare ile sağ tıklayarak işletim sisteminin kurulacağı “ / ” (Kök) dizini belirleyiniz. (*)    |
| :-------------: |
| ![alt text](https://www.pardus.org.tr/documents/10180/830384338/atama.png/de772020-6757-4bb4-8fb6-27ff324a8793?t=1429866681000 "Logo Title Text 1")       |
| ![alt text](https://www.pardus.org.tr/documents/10180/830384338/kok.png/729ab5e9-99e3-4743-aed2-46a15f900c44?t=1429866772000 "Logo Title Text 1")       |

| Ardından aynı şekilde sağ tıklayarak “home” (Ev Dizini) belirleyiniz.(**)   |
| :-------------: |
| ![alt text](https://www.pardus.org.tr/documents/10180/830384338/home.png/c08c80ee-cfda-4952-a458-76a10ae9ff48?t=1429866747000 "Logo Title Text 1")       |

(!) Sadece “ / “ (kök) dizin belirleme işlemi yaparak da kurulumu tamamlayabilirsiniz.

(!!) Pardus 2013 de kullanmış olduğunuz ev dizininizi bu ekranda seçerek devam edebilir veya yeni bir birimi ev dizini olarak belirleyebilirsiniz. Ev dizini belirtilmediğinde “/” ve “home” aynı birime kurulacaktır, herhangi bir ev dizini belirlemeden de devam edebilirsiniz.

| 8- Her işletim sistemi kendi önyükleyici ayarlarını kurar. Linux sistemlerde önyükleyici uygulaması “Grub”'tır. Grub'ı fiziksel diskin baş kısmına kurulması tavsiye edilir. Bu sayede bilgisayarınızdaki işletim sistemlerini okur ve giriş ekranında görüntüler.|
| :-------------: |
| ![alt text](https://www.pardus.org.tr/documents/10180/830384338/grub.png/cda8b6e0-5502-4f7f-a7d2-eabc703f2f08?t=1429866738000 "Logo Title Text 1")       |

6. adımdaki (/dev/sdb) disk seçimini gözden geçirerek Grub'ı kur onayı veriniz.

| 9- Son olarak bu ana kadar yapılan değişiklikler ekranda görüntülenir. Bu aşamadan sonra geri dönüş olmayacaktır.  |
| :-------------: |
| ![alt text](https://www.pardus.org.tr/documents/10180/830384338/ozet.png/567c5ee7-8165-4df0-a411-776d89e6d361?t=1429866792000 "Logo Title Text 1")       |

Yapılacak bir değişiklik yoksa “Kur” butonuna tıklayarak kuruluma başlayabilirsiniz. Kurulum bilgisayarınızın hızına göre 15 ile 25 dakika aralığında olacaktır.  

| 10- Kurulum tamamlandıktan sonra bilgisayarınızı yeniden başlatabilir veya canlı sistemde devam edebilirsiniz. Fakat unutmayınız ki canlı sistem üzerinde yaptığınız değişiklikler yeniden başlama sonrasında kaybolacaktır.  |
| :-------------: |
| ![alt text](https://www.pardus.org.tr/documents/10180/830384338/ybasla.png/752b67ce-5a31-45dd-9939-8481631a2bb9?t=1429866802000 "Logo Title Text 1")       |

Bilgisayarınız yeniden başlarken kurulum ortamını(DVD-USB) çıkarmayı unutmayınız. Aksi taktirde bilgisayarınız tekrar canlı sistem ile açılacaktır.

| Pardus Kurumsal 5 kurulumu tamamlandı, kullanıma başlayabilirsiniz.  |
| :-------------: |
| ![alt text](https://www.pardus.org.tr/documents/10180/830384338/bitti.png/6b2e3f6d-06e9-4e85-8ca5-86adeed3f215?t=1429866706000 "Logo Title Text 1")       |

##### 2-	Php ve Apache
Root erişimine geçtikten sonra:

```javascript
$ pisi update-repo (depo güncellemesi)
$ sudo pisi it apache mod_php
$service apache on (açılışta apache başlasın)
```

##### 3-	RRd Tools
http://oss.oetiker.ch/rrdtool/download.en.html adresindeki herhangi bir yansıdan rrdtool.tar.gz son sürümünü edinebilirsiniz. (rrdtool-1.2.15)
 
```javascript
$tar -zxvf rrdtool.tar.gz
$cd rrdtool-1.2.15
$./configure
$make
$make install
```

NOT: Pardus 2007.1 üzerinde sorunsuz derleniyor.

##### 4-NfDump

http://sourceforge.net/project/showfiles.php?group_id=119350

```javascript
$tar -zxvf nfdump-1.5.2.tar.gz
$cd nfdump-1.5.2
$./configure
$make
$make install
```

##### 5-Nfsen

Adım adım kurulumunu ve ayarlarını anlatacağımız NfSen ile ilgili belgelere http://nfsen.sourceforge.net/	adresinden ulaşabilirsiniz. http://sourceforge.net/project/showfiles.php?group_id=134525

```javascript
$tar -zxvf nfsen-1.2.4.tar.gz
$cd nfsen-1.2.4/etc
$cp nfsen-dist.conf nfsen.conf
$vi nfsen.conf
```

$HTMLDIR değerini aşağıdaki şekilde değiştirin...

__$HTMLDIR= "/var/www/localhost/htdocs/nfsen/";__

%sources Netflow bilgisini yollayan kaynakları içermemiz gerekiyor. Buradaki port değeri yönlendirici ayarlarında NetFlow datasının yollandığı portu ifade ediyor.

```javascript
%sources = (
    'deneme'	=> { 'port'	=> '9996', 'col' => '#0000ff' },
);
```

:wq (vi dan kaydederek çıkın)

```javascript
$cd..
$groupadd www
$useradd www -g www
$useradd netflow -g www
$vi /etc/apache2/httpd.conf
```

Ayarlar dosyasındaki User ve Group değerlerini apache yerine www ile değiştireniz gerekiyor.

```javascript
User www 
Group www
```
```javascript
$./install.pl etc/nfsen.conf	(perl yolu /usr/bin/perl )
```

Bu komutla birlikte NfSen için gerekli klasörleri oluşturacak, nfsen.conf da belirlediğimiz HTMLDIR altına php/html dosyalarını kopyalayacak, live profile 'ı yaratacaktır. Bu komut sonrasında nfsen.conf dosyasının CONFDIR altında da konduğunu bir kontrol edelim.

```javascript
$ls -la /data/nfsen/etc/nfsen.conf
-rw-r--r-- 1 root root 4451 Mar 28 16:47 /data/nfsen/etc/nfsen.conf
```

```javascript
$cd  /var/www/localhost/htdocs/nfsen
$chmod 755 nfsen.php
$cd /data/nfsen/bin/
$pwd
/data/nfsen/bin
$./nfsen.rc start
```

Kullandığınız profile'ın şu an ki durumunu izlemek için:

```javascript
$./nfsen -l live name	live
tstart  Wed Mar 28 16:55:00 2007
tend	Thu Mar 29 10:15:00 2007
updated Wed Mar 28 16:50:00 2007 filter  <none>
expire  0 hours
size	0
maxsize 0 sources deneme type	live locked 0
status OK
```

Eğer locked değeriniz 1 görünüyor ise aşağıdaki komut ile tekrar analiz prosedürünü başlatabilirsiniz:

```javascript
$./nfsen -m live -U
$pwd
/data/nfsen/bin
```
 
Tüm nfsen komutlarını /data/nfsen/bin altında çalıştırdık. Bu klasörü genel yola eklemek için (PATH):

```javascript
$EXPORT PATH=$PATH:/data/nfsen/bin
```

Nfsen ile analizlerin açılışta başlatılması için local.start dosyasını değiştirmeniz gerekmektedir:

```javascript
$vi /etc/conf.d/local.start
/data/nfsen/bin/nfsen.rc start (Bu satırı dosyanın sonuna eklemeniz yeterlidir)
```

##### C) NfSen ile Analiz

Artık Pardus kurduğumuz makinada Firefox ya da benzeri uygulamalar ile: http://localhost/nfsen/nfsen.php adresini görüntülediğimizde NfSen arayüzü ile karşılaşacağız.

Detail bölümünden ilgilendiğimiz zaman aralığını seçerek (grafik üstünde) ilgili kayıtları ayıklayabiliriz. Bunun için grafiğin altında yer alan _Processing_ bölümünü kullanmanı gerekiyor. Source bölümde ayarlar dosyasında UDP portlarına göre ayırdığımız kaynaklar listeleniyor.

Buradan bir kayna seçtikten sonra Filter bölümde özel olarak ilgilendiğimiz bir eleman için (Ip adresi, arayüz, AS, Port v.s) filtreleme yapabiliriz. Bu alanının kullanım şekli Nfdump datasının kullanım şekli ile aynı. (http://nfdump.sourceforge.net/) adresinden Filter bölümünde detaylara ulaşabilirsiniz. Bazı alanlar için filtreleme şeklini aşağıda     bulabilirsiniz.

##### 1-Filtreler:
Başlıkların altında verilen komutları teker teker ya da bir kaçını birden Filters bölümüne yazarak ilgili başlığa göre filtreleme yapabilirsiniz.
protokol nesli

> Ipv4 için __inet__ yada __ipv4__

>Ipv6 için __inet6__ ya da __ipv6__

###### protokol
___
__TCP, UDP, ICMP, GRE, ESP, AH, RSVP__ yada __PROTO  <protokol_numarası>__

###### IP Adresi
___
Kaynak Ipsi için: __IP a.b.c.d__

Kaynak ya da hedef: __HOST a.b.c.d__

###### Ağ
___
__NET a.b.c.d m.n.r.s__	(m.n.r.s ağ maskesi)

__NET a.b.c.d / num__ (Ya da / gösterimi  ile)

###### Kapı
___
__PORT [operator] port_no__ (operator olarak =,>,< kullanılabilir)
 
###### Arayüz
___
[inout] __IF arayuz_no__ (başına eklenecek in ya da out ile trafiğin yönünü de belirtebilirsiniz)

###### Pakete göre
___
__packets__ [operator] __sayı__ [scale] (scale değeri __k,m,g__ olabilir. Kilo, mega ve giga için)


###### Byte değerine göre
___
__bytes__ [operator] __sayı__ [scale]


###### Saniyelik Paket (Packets per second):
___
__pps__ [operator] __num__ [scale]

###### Flow zamanına göre:
___
__duration__ [operator] __num__

###### Saniyelik Bite Göre (Bits per second):
___
__bps__ [operator] __num__ [scale] .

###### Paketlerine Byte cinsinden büyüklüğüne göre (Bytes per packet):
___
__bpp__ [operator] __num__ [scale]

###### AS numarası
___
[SourceDestination] __AS sayı__

##### 2-Show Bölümü

Kaynağı ve filtreleri hallettikten sonra şimdi NetFlow dökebilir ya da istatistiğini alabilirsiniz. List seçeneğinde dökeceğiniz flowların sayısını ve formatını belirleyebiliyorken, Stat seçeneğinde kaynak IP, hedef Ip, Kapı, AS numarası v.s. için çıkacak istatistikleri byte,paket,pps v.s. için sıralatabilirsiniz.


Önemli Not:
Ağınız ile ilgili tüm trafiğin bilgilerini barındıran Netflow kayıtlarının tutulması ve analiz edilmesi ağ yönetimi için çok önemlidir. Bununla birlikte bu kayıtlar kurum çalışanlarınızın kişisel bilgilerini de içerdiğinden ağ yöneticileri dışında kişilerin erişimine izin verilmemelidir. Bunun için en pratik çözüm olarak
.htaccess dosyası yardımı ile web sunucusuna erişimi kullanıcı tabanlı yapmak sayılabilir. Ayrıca sunucuya ssh erişiminin de çok dikkatli yapılması gerekmektedir.
