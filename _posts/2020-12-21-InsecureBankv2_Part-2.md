---
layout: post
title:  "InsecureBankv2 Writeup - Part 2"
author: "Alp Eren Işık"
date:   2020-12-21
categories: [writeup, mobile-security]
permalink: /:year/:month/:day/:title:output_ext
---

# Insecure Bank v2 Writeup - Part 2
-------------------------------------

Sınavlar/ödevler/quizler/projeler vs vs derken bir süredir yazmıyordum. Eh tabi Mr. Robot reisin kaçıncı sezonunun bilmem kaçıncı bölümündeki gibi sürekli aynı rutinin içinde sıkışıp kalmak da biraz motivasyon kaybına yol açıyor 😅 Neyse lafı fazla uzatmıyayım ve yarım kalan şu serimizi hızlıca tamamlayalım ki yeni serilerin önü açılsın :)

# `Insecure HTTP Connections`

Bildiğiniz üzere HTTP verileri şifrelemeden göndermektedir. Bu da giriş yapma, kayıt olma, parola sıfırlama, bankacılık aktiviteleri, ödeme sistemleri gibi gibi önemli yerlerde trafiğin ağı dinleyen kişiler tarafından clear-text olarak görülebilmesine neden olur (veya log tutuluyorsa bu loglarda herşey açıkca gözükür). Bizim de burada ki uygulamamızın aslında bir bankacılık uygulaması olduğunu düşünürsek verinin şifrelenerek gitmesini beklemekteyiz.

HTTP isteklerini görmek için Burp Suite aracımızı sanal telefonumuza bağlayalım...

Eğer android 7 sürümünden önceki sürümleri kullanıyorsanız burp sertifikasını yüklemek için...

Burp Suite -> Proxy -> Options -> Import/Export CA certificate

![0](/static/img/posts/insecurebankv2/part2/0.png)

Burdan sertifikayı istediğiniz bir dizine kayıt edeceksiniz. Daha sonra bunun uzantısını .cer olarak değiştireceğiz.

![1](/static/img/posts/insecurebankv2/part2/1.png)

```bash
~/InsecureBank
➞  file cert                                                                   
cert: Certificate, Version=3

~/InsecureBank
➞  mv cert cert.cer                                                            

~/InsecureBank
➞  adb push cert.cer /storage/emulated/0/Download/                             
cert.cer: 1 file pushed, 0 skipped. 3.6 MB/s (939 bytes in 0.000s)

```

Daha sonra sanal telefonumuzda Settings -> Security -> Install from SD card bölümünden sertifikamızı yükleyebiliriz.

![2](/static/img/posts/insecurebankv2/part2/2.png)

![3](/static/img/posts/insecurebankv2/part2/3.png)

Daha sonra gene Settings -> Security altında bulunan Trusted credentials kısmında sertifikamızın yüklendiğini görebiliriz.

![4](/static/img/posts/insecurebankv2/part2/4.png)

Evet şimdi de sıra Burp ile bağlama kısmında. Genymotion bildiğiniz üzere arka planda virtualbox kullanmakta ve siz bir sanal telefon oluşturduğunuz zaman bir ağını NAT diğer ağını Host-only Adapter yapmakta. Bundan dolayı telefonun bulunduğu ağda ki ip adresinizi öğrenmeniz gerekiyor. Eğer linux kullanıyorsanız ip addr show veya ifconfig gibi komutlar ile vboxnet0: vboxnet1: vboxnet2: gibi host-only ağlardaki ip adreslerinizi görebilirsiniz. Telefonun hangi ağda olduğunu öğrenmek içinde virtualbox'da telefonunuzu bulun (Genymotion'da girdiğiniz isim ile açılmış Other Linux (64bit) olarak ayarlanmış makine). Daha sonra bu makinenin Settings -> Network kısmına giderseniz orada hangi ağda olduğunu öğrenmiş olursunuz.

Artık telefonumuzun hangi ağda olduğunu ve bizim o ağdaki adresimizin ne olduğunu öğrendiğimize göre gerekli ayarlamaları yapalım... Telefonumuzda Settings -> Wi-Fi kısmında WiredSSID yazan kısma biraz uzun bastıktan sonra açılan ekranda Modify network diyelim ve kendi cihazımızın ip adresini ve belirlediğimiz bir port numarasını girelim.

![5](/static/img/posts/insecurebankv2/part2/5.png)

Kaydettikten sonra burp arkadaşımızı açalım Proxy -> Options kısmına gidelim. Add butonuna basalım ve girmiş olduğumuz ip adresi ve port numarasını ekleyelim. (Tabi burda ne uğraşçam tek tek direk All interfaces derim kafam rahat eder de diyebilirsiniz o size kalmış)

![6](/static/img/posts/insecurebankv2/part2/6.png)

Evet bunu da listeye ekledikten sonra artık cihazımızdaki HTTP(S) istekleri yakalayabiliriz. Ben bu anlatımdan hiç bişi anlamadım (bi b*k anlamadım kardeş) derseniz eğer aşşağıya bir iki link bırakayım bir de orlara bakın :)

    https://eybisi.run/Mobil-Zararli-Analizi-Bolum-1-Ortami-Kuralim/

    https://portswigger.net/support/configuring-an-android-device-to-work-with-burp

    https://portswigger.net/support/installing-burp-suites-ca-certificate-in-an-android-device

Evet android 7 sürümünden önce böyle. Peki sizin cihazınız 7 sürümü veya daha yüksek sürüme sahip ise eğer o da şu şekilde...

```bash
$ openssl x509 -inform DER -in cacert.der -out cacert.pem
$ openssl x509 -inform PEM -subject_hash_old -in cacert.pem | head -1
9a5ba575
$ mv cacert.pem 9a5ba575.0
$ adb root
$ adb remount
$ adb push 9a5ba575.0 /sdcard/
$ adb shell
$ mv /sdcard/9a5ba575.0 /system/etc/security/cacerts/
$ chmod 644 /system/etc/security/cacerts/9a5ba575.0
$ reboot
Telephone -> Settings -> Wi-Fi CLICK -> Modify Network -> Proxy Manual
Burp Suite -> Proxy -> Options -> Add (Add a new proxy listener)
```

Şimdi bir giriş işlemi yapalım bakalım. (dinesh/Dinesh@123$ || jack/Jack@123$)

![7](/static/img/posts/insecurebankv2/part2/7.png)

Evet login işlemi sırasında girilen bilgileri açık bir şekilde görebiliyoruz...

# `Parameter Manipulation`

Bu kısımda da `jack` kullanıcısının parolasını sıfırlamasını beklerken isteği yakalayıp dinesh kullanıcısının parolasını değiştirebiliyoruz (girilen parolayı değiştirip kendi istediğimiz parolayı da verebiliriz). Veya bu istedği burp suite üzerinde repeater'a atarak da istediğimiz kullanıcının parolasını değiştirebiliriz.

![8](/static/img/posts/insecurebankv2/part2/8.png)

# `Username Enumeration issue`

Bu aşamada da /changepassword isteğini Intruder'a atalım.

![9](/static/img/posts/insecurebankv2/part2/9.png)

![10](/static/img/posts/insecurebankv2/part2/10.png)

Ufak bir wordlist vererek otomatik bir şekilde sistemde kayıtlı kullanıcıların parolalarını değiştirebilecek miyiz ona bir bakalım :)

![11](/static/img/posts/insecurebankv2/part2/11.png)

Ve gördüğümüz üzere sistemde kayıtlı olan kullanıcıların parolaları bizim belirlemiş olduğumuz bir parola ile değiştirilmiş oldu. İsterseniz bir de test edelim...

![12](/static/img/posts/insecurebankv2/part2/12.png)

Belirlemiş olduğumuz yeni parola ile başarılı bir şekilde giriş yapmış oldu 😇

Bundan sonraki kısımlarda Burp'e ihtiyacımız olmayacağı için Wi-Fi kısmından Proxy ayarını tekrardan eski haline getirelim (None yapalım). Arka tarafta dursun bir zararı olmaz derseniz de siz bilirsiniz :)

# `Flawed Broadcast Receivers`

Öncelikli olarak Broadcast konusunda fikir sahibi olmayan arkadaşlar için şöyle bir kaç link bırakayım. Neyin ne olduğunu bilmeden zafiyeti anlamak ve zafiyeti sömürmek pek mümkün olmaz. Olsa bile sağlıklı olmaz :D

    https://developer.android.com/guide/components/broadcasts

    https://gelecegiyazanlar.turkcell.com.tr/konu/android/egitim/android-301/broadcastreceiver-kullanimi

    https://resources.infosecinstitute.com/topic/android-hacking-security-part-3-exploiting-broadcast-receivers/

Evet bu yazılara şöyle bir göz gezdirince zaten neyin ne olduğunu anlamış oluyoruz. Şimdi uygulamamızın AndroidManifest.xml dosyasına bir bakalım acaba herhangi bir broadcast var mı ?

![13](/static/img/posts/insecurebankv2/part2/13.png)

Bir tane broadcast receiver olduğunu görüyoruz. Aynı zamanda exported="true" olarak verilmiş ki bu da bizim dışardan müdahale edebileceğimiz anlamına geliyor :)

![14](/static/img/posts/insecurebankv2/part2/14.png)

```java
String phn = intent.getStringExtra("phonenumber");
String newpass = intent.getStringExtra("newpass");
```

Şimdi bu yayın alıcısının tetiklenmesi için iki adet parametre olduğunu görüyoruz.

```java
public static final String MYPREFS = "mySharedPreferences";
...
...
SharedPreferences settings = context.getSharedPreferences("mySharedPreferences", 1);
```

Bu kısımda ise bazı bilgilerin cihaz içerisinde depolanan bir yerden geldiğini görebiliriz. Burdan ilgili bilgiler alındıktan sonra bilgilendirme mesajı SMS olarak gelmiş olacak.

Artık herşeyi öğrendiğimize göre yapacağımız şey çok basit...

```bash
adb shell am broadcast -a theBroadcast -n com.android.insecurebankv2/.MyBroadCastReceiver --es phonenumber 1337 --es newpass yeniparola123
```

activity manager aracının içerisinde bulunan broadcast komutunu kullanıyoruz. -a parametresi action anlamına gelmekte. Eğer AndroidManifest içerisinde receiver'ın olduğu kısma bakarsanız ` <action android:name="theBroadcast" >` olarak görebiliriz. -n parametresi component anlamına gelmekte. Burda da paket_adi/.Receiver_Name şeklinde birşey girmemiz lazım. Tekrardan AndroidManifest dosyasına bakacak olursak `package="com.android.insecurebankv2" >` ve `<receiver android:name=".MyBroadCastReceiver"` olduğunu görebiliriz. En sonda yer alan --es parametresi ise ekstra anahtar kelime veya ekstra string değer gibi bir anlamı bulunmakta. MyBroadCastReceiver.java içerisinde bulunan `intent.getStringExtra` ile alınan/beklenen parametreleride bu parametre sayesinde belirtmiş oluyoruz.

Komut ve parametreler hakkında daha fazla bilgi için şuraya bakabilirsiniz => https://developer.android.com/studio/command-line/adb#am

![15](/static/img/posts/insecurebankv2/part2/15.png)

![16](/static/img/posts/insecurebankv2/part2/16.png)

Evet dilediğimiz numaradan bu şekilde bir mesaj almasını sağlayabiliyoruz.

# `Insecure Content Provider access`

Nedir bu content provider ? Yenilir mi içilir mi ? gibi sorulara cevap olması açısından gene şöyle bir kaç link bırakayım :)

    https://developer.android.com/guide/topics/providers/content-provider-basics

    https://gelecegiyazanlar.turkcell.com.tr/konu/android/egitim/android-401/bir-araci-icerik-saglayicisi-olusturmak

    https://resources.infosecinstitute.com/topic/android-hacking-security-part-2-content-provider-leakage/

Şimdi gelelim tekrardan AndroidManifest dosyamıza

![17](/static/img/posts/insecurebankv2/part2/17.png)

Burada bir provider görüyoruz. exported="true" olarak verilmiş. O halde bunun üstünden ilerleyebiliriz.

![18](/static/img/posts/insecurebankv2/part2/18.png)

Şu kısımda da URL adresini görebiliriz. O halde yapmamız gereken tek şey ufak bir komut çalıştırarak tüm verileri listelemek

```bash
$ adb shell content query --uri content://com.android.insecurebankv2.TrackUserContentProvider/trackerusers
Row: 0 id=3, name=dinesh
Row: 1 id=1, name=jack
Row: 2 id=2, name=jack
Row: 3 id=4, name=jack
Row: 4 id=5, name=jack
Row: 5 id=6, name=jack
```

# `Android Backup vulnerability`

![19](/static/img/posts/insecurebankv2/part2/19.png)

Uygulamanın backup'ının/yedeğinin alınmasına izin verildiğini görüyoruz. Backup almak için aşşağıdaki komutu kullanıyoruz.

    adb backup com.android.insecurebankv2

Bulunduğumuz dizinde backup.ab isimli bir dosya oluşmuş olacak. Bunu açabilmek ve verilere erişebilmek için [Android backup extractor](https://github.com/nelenkov/android-backup-extractor) aracından faydalanabilirsiniz. Başka bir araç veya yolu var mı diye bakındım ancak işe yarar bişi çıkmadı.

Backup içerisinden dosyaları çıkardığımız zaman telefonda depolanan tüm verilere erişmiş olacağız.

# `Android keyboard cache issues`

Android'de girdiğimiz kelimeler daha sonra otomatik düzeltme amacıyla kullanılmak üzere bir yere kayıt ediliyormuş. Bu kelimelerin tutulduğu dosyayada dışardan erişmek mümkünmüş. (öyleymiş böyleymiş şöyleymiş :D )

Her neyse bu dosyayanın yolu şurada...

```
root@vbox86p:/data/data/com.android.providers.userdictionary/databases $  ls
user_dict.db
user_dict.db-journal
root@vbox86p:/data/data/com.android.providers.userdictionary/databases $ sqlite3 user_dict.db
sqlite> .tables
android_metadata  words           
sqlite> select * from words;
```

Benim cihaz hiçbişi kaydetmemiş o yüzden tam emin olamadım :D Acaba cihaz da mı bir sıkıntı var yoksa ben de mi bir sıkıntı var diye de düşünmüyor değil insan 🤔

# `Insecure Webview implementation`

Uygulamamızın viewStatment kısmında `setJavaScriptEnabled(true)` diye bir ifade bulunmakta.

![23](/static/img/posts/insecurebankv2/part2/23.png)

Onun hemen üstünde ne görüyoruz. `file//` demiş, `.html` demiş... O zaman bu bir dosya olarak geliyorsa bizim bu dosyayı bulmamız lazım. Dosyayı bulduktan sonra da içerisinde javascript yazmamız lazım. Neden ? Çünki javascript enabled demiş :D

```bash
$ adb logcat | grep "$(adb shell ps | grep "com.android.insecurebankv2" | awk '{print $2}')"
12-22 07:16:00.329  4006  4006 I System.out: /storage/emulated/0/Statements_jack.html
```

Dosyanın yolunu öğrendik şimdi içeriğini değiştirelim.

![24](/static/img/posts/insecurebankv2/part2/24.png)

# `Application Debuggable`

![20](/static/img/posts/insecurebankv2/part2/20.png)

AndroidManifest dosyasına baktığımız zaman uygulamanın debug edilebilir olduğunu görüyoruz.

1.

```bash
$ adb jdwp
644
756
845
...
...
2760
3523
3760
```

2.

```bash
$ adb forward tcp:1337 jdwp:3760
$ jdb -attach 1337
```

Dediğimiz zaman debug etmeye başlayabiliyoruz.

```bash
> methods com.android.insecurebankv2.PostLogin
** methods list **
com.android.insecurebankv2.PostLogin <init>()
com.android.insecurebankv2.PostLogin doesSUexist()
com.android.insecurebankv2.PostLogin doesSuperuserApkExist(java.lang.String)
com.android.insecurebankv2.PostLogin callPreferences()
com.android.insecurebankv2.PostLogin changePasswd()
com.android.insecurebankv2.PostLogin onCreate(android.os.Bundle)
com.android.insecurebankv2.PostLogin onCreateOptionsMenu(android.view.Menu)
com.android.insecurebankv2.PostLogin onOptionsItemSelected(android.view.MenuItem)
com.android.insecurebankv2.PostLogin showRootStatus()
com.android.insecurebankv2.PostLogin viewStatment()
...
...
```

Şu şekilde aktivitelerin methodlarına görebiliyoruz (Tabi bunlara erişebilmek için öncesinde o aktiviteyi çalışmış olması gerekmekte) . help komutu ile başka ne tarz komutlar olduğuna bakabilirsiniz. Bir noktaya breakpoint koymak içinse...

```bash
> stop in package_name.activity_name.method_name
```

step veya locals komutu ile local ve method variable ları görebiliyoruz. Değiştirmek içinse set variable_name = "..." şeklinde bir komut var. Gerekli değişiklikleri yaptıktan sonra da run diyerek devam ettirebiliyoruz programı

Açıkcası böyle değer değiştirmek dışında debugger üzerinden başka ne tarz şeyler yapıldığına dair bir fikrim yok. Biraz bakındım ama güzel bir dökümantasyonda bulamadım :(

# `Android Pasteboard vulnerability`

Android'de birşeyleri kopyaladığımız zaman bunlar panoya yazılır. Bu panonun içeriğini de getSystemService() methodu ile ClipboardManager objesi çağrılabilir. Öncelikli olarak uygulamamızda bir veriyi kopyalayalım

![21](/static/img/posts/insecurebankv2/part2/21.png)

Bunu kopyaladıktan sonra uygulamamızın çalıştığı UID değerini öğrenelim. Daha sonra adb komutu ile clipboard'a erişip içerisindeki veriyi okuyalım

```bash
$ adb shell ps | grep "insecurebankv2"                                    
u0_a52    4006  276   834888 182536    ep_poll f72ced75 S com.android.insecurebankv2

$ adb shell su  u0_a52 service call clipboard 2 s16 com.android.insecurebankv2
...
...
...
```

adb shell su u0_a52 olan kısımda uygulamamızı çalıştıran user ile bir kabuk erişimi elde ettik. service call ile getSystemService() methodunu çağırdık ve clipboard ile de ClipboardManager objesini başlattık. Daha sonra gelen 2 ise setClipboardText anlamına gelmekte. s16 UTF-16 olsun demek ve en sonda da uygulamamızın paket adı yer almakta

![22](/static/img/posts/insecurebankv2/part2/22.png)

# `Intent Sniffing and Injection`

Bu aşamada aşağıda ki komut yardımıyla çalıştırdığım broadcast aktivitelerini bulabildim. Drozer ile biraz daha detaylı bir çıktı sağlanıyor ancak bunu adb ile nasıl yapabiliriz bilmiyorum. Onun dışında `Injection` kısmının nasıl olduğuna dair de hiçbir fikrim yok ne yazık ki :/

![Intent_Sniffing_and_Injection](/static/img/posts/insecurebankv2/part2/intent_sniffing.png)
