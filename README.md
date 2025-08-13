# X11 Nedir?

**X11 (X Window System)**; Linux/Unix'te GUI'nin "telgraf dili" dir. Uygulama (istemci) ekrana çizmek istediğinde **X sunucusuna** istek paketleri yollar; X sunucusu da ekrana çizer, klavye/fare olaylarını uygulamaya geri yollar.

* **X sunucusu (X server) :** Ekran + klavye + fare neredeyse orada çalışır (genelde masaüstü).
* **X istemci (X client) :** GUI uygulaması (ör. gedit, PyQt programı)
* **DISPLAY :** Uygulamanın hangi X sunucusuna bağlanacağını söyleyen adres, biçimi **host:display.screen** (örn. :0, localhost:10.0)

## Protokol Akışı : 

* **İstemci -> Sunucu  :** Şu boyutta pencere aç, şu koordinata şu metni çiz...
* **Sunucu  -> İstemci :** Klavyede A basıldı, pencere görünür oldu, fare hareket etti...
* **Eklentiler :** GLX (OpenGL), MIT-SHM (paylaşımlı bellek), XRender vb.

### Neden böyle ?
* Ağ şeffaflığı için: uygulama bir makinede, ekran başka bir makinede olabilir. Bu sayede "uzaktan GUI" mümkündür.


# X11 Forwarding Nedir?

**SSH'nin X11 protokolünü şifreli bir tünelden geçirip yerel ekranına ulaştırmasıdır. Uygulama uzak makinede (Xavier NX, Orin vb.) çalışır; pencere **senin** ekranında açılır.


### Adım adım "forwarding" nasıl işler?

1. **ssh -Y uzak_makine** ile bağlanırsın. SSH istemcisi yerel X sunucusundan **gerçek bir yetki çerezi (xauth cookie)** alır, uzak tarafta **sahte bir çerez** yazar ve uzak oturumda **DISPLAY=localhost:10.0** gibi bir değer ayarlar.
2. Uzakta uygulama başlar ve **DISPLAY=localhost:10.0** adresine bağlanır. Bu adres aslında **uzaktaki sshd'nin X proxy'sidir** (X11UseLocalhost=yes ile yalnız loopback).
3. **sshd** bu X11 paketlerini **mevcut SSH bağlantısı üzerinde açılan X11 kanalı** ile **şifreli** olarak yerel makineye yollar.
4. Yereldeki **ssh** istemcisi gelen X11 paketlerini alır ve **yerel X sunucusuna** (bizim ekranımıza) iletir.
5. Yerel X sunucusu çizer. Klavye/fare olayları **ters yönde** aynı tünelden uygulamaya döner.  


### GÜvenlik & Yetkilendirme

* **xauth (MIT-MAGIC-COOKIE-1) :** "Bu X sunucusuna kim pencere açabilir?" sorusunun cevabı. SSH bu çerezleri otomatik yönetir, böylece rastgele süreçlerin ekranına dadanması engellenir.
* **X11UseLocalhost yes :** Uzak taraftaki X proxy sadece 127.0.0.1'de dinler (dışarıdan erişilemez), daha güvenli.



# Ne amaçla kullanılır? 

* **Uzak GUI'yi hızlıca görmek :** Sunucu/Jetson üzerinde çalışan tek bir GUI aracı (ayar, test, kısa kullanım.)
* **Geliştirme/diagnostik :** "O makinede koşsun ama penceresi burada açılsın."
* **Ağ şeffaflığı :** Ekranı olmayan bir cihazın (Xavier NX) GUI'sini masaüstünde kullanmak.

### Avantajları

* Sadece SSH yeter; kurulum hafif, güvenli (şifreli).
* Tek uygulama bazında çalıştırma kolay.

### Sınırlamaları

* Gecikmeye hassas ve "chatty" protokol - yüksek gecikmeli ağlarda yavaş.
* 3D/GL işi genelde **donanım hızlandırmasız** ve ağır olur (MIT-SHM ağda devre dışı).
* Ağ trafiği arttıkça **pencere tepkisi** düşebilir.

### Ağır/3D gerekiyorsa alternatifler

* **Xpra** (uygulama-bazlı ekran aktarımı, sıkıştırma/yeniden çizim optimizasyonları)
* **TurboVNC + VirtualGL, NoMachine (NX), RDP/VNC** (tüm masaüstünü görüntüler; bazıları sunucu tarafında encode eder - ağda daha verimli)


<img width="954" height="401" alt="image" src="https://github.com/user-attachments/assets/36e28c46-13ce-4435-834c-71929a516e3b" />


------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Neden ssh user_name@user_ip hata verirken (GUI'yi çalıştırmazken) ssh -Y user_name@user_ip çalışıyor (GUI'yi çalıştırır) ?

* **ssh (bayraksız) : X11 forwarding İSTEMEZ.** Uzak oturumda **$DISPLAY**ayarlanmaz, X11 cookie hazırlanmaz. PyQt "bağlanacak ekran yok" diye düşer - **Could not to any X display**

* **ssh -Y :** İstemci **X11 forwarding Talep eder (trusted).** Uzakta **DISPLAY=localhost:10.0** ayarlanır, **xauth cookie** yazılır, tünel açılır. Uygulama ekrana bağlanabildiği için pencere gelir.

Sunucudaki **X11Forwarding yes** sadece izin verir; **istemci talep etmedikçe** X11 tüneli kurulmaz. O talebi **-X veya -Y (ya da client config)** verir.


### -X ve -Y farkı (neden QT bazen -X'te sorun çıkarır?)

* **-X = untrusted** (güvenlik kısıtlamaları daha sıkı). Bazı Qt/OpenGL özellikleri kısıtlanır; uygulama ya çalışmaz ya da eksik çalışır.
* **-Y = trusted** Kısıtlar kalkar; Qt/PyQt genelde sorunsuz çalışır. (Güvendiğin hostlarda kullan.)

 ### Her seferinde -Y yazmadan ssh bağlantısı

 1. **İstemci tarafında (senin Ubuntu sisteminde) kalıcı ayar :**

* ~/.ssh/config dosyası oluştur/editle :

```cmd
Host host_name host_ip
  HostName host_ip
  User remote_username
  ForwardX11 yes
  ForwardX11Trusted yes
  Compression yes
```
* Örnek kullanım

```cmd
Host xavier 10.62.2.61
  HostName 10.62.2.61
  User hayati
  ForwardX11 yes
  ForwardX11Trusted yes
  Compression yes
```
Artık ssh host_name yazmak yeterlidir.



2. **Sunucu Tarafı (Xavier NX) kontrolleri (bir kere)**

* **/etc/ssh/sshd_config :**

```cmd
X11Forwarding yes
X11UseLocalhost yes
```

```cmd
sudo systemctl restart ssh
sudo apt install -y xauth x11-apps
```

```cmd
ssh ekin@10.62.2.61
echo $DISPLAY          # dolu olmalı (örn: localhost:10.0)
xclock                 # pencere sende açılmalı
```

3. **PyQt için küçük uyumluluk bayrağı (gerekirse)**

```cmd
export QT_X11_NO_MITSHM=1 
```



# Ekstra (X11 kurulum Xavier NX tarafı)


```cmd
sudo apt update
sudo apt install -y openssh-server xauth x11-apps mesa-utils

sudo nano /etc/ssh/sshd_config
# Bu satırlar böyle olmalı:
# X11Forwarding yes
# X11UseLocalhost yes

sudo systemctl restart ssh
```







































































