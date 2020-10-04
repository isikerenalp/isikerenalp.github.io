---
layout: post
title:  "DIVA (Damn insecure and vulnerable App) Writeup"
author: "Alp Eren Işık"
date:   2020-10-4
categories: [writeup, mobile-security]
permalink: /:year/:month/:day/:title:output_ext
---

# DIVA (Damn insecure and vulnerable App) Writeup

Zafiyetli apk dosyasını kurmak için [GitHub](https://github.com/payatu/diva-android) adresinden indirip kendiniz compile edebilir veya `http://www.payatu.com/wp-content/uploads/2016/01/diva-beta.tar.gz` adresinden direk apk dosyasını indirebilirsiniz.

APK dosyasını indirdikten sonra ihtiyacımız olacak bazı araç gereçler olacak.

  - Android Emulator (genymotion, android studio etc.)

  - adb (Android Debug Bridge)

  - jadx-gui, jd-gui etc. (İnceleyeceğimiz apk dosyası open-source olduğundan dolayı isterseniz bunları kurmak yerine kaynak kodu bilgisayarınıza indirip veya tarayıcıdan github adresine giderekde inceleyebilirsiniz)

Sanırım şimdilik bu kadarı işimizi görür. Ayağa kaldıracağımız `Android` versiyon numarasının çok büyük olmasına gerek yok. Android sürümünü 5,6,7 gibi düşük sürümler seçebilirsiniz.

Gerekli kurulumları yaptıktan sonra çayımızı/kahvemizi/müziğimizi vs de ayarladıysak eğer başlayabiliriz. Öncelikli olarak bu `apk` dosyasını cihazımıza yüklememiz gerekecek.

    adb install diva-beta.apk

![install](/static/img/posts/diva/0.png)

İlgili komutumuzu çalıştırdık ve uygulamamızı yüklemiş olduk.

![app](/static/img/posts/diva/1.png)

Ekran görüntüsüne sığmıyor ancak toplamda 13 tane bölüm bulunmakta

```bash
1. Insecure Logging
2. Hardcoding Issues – Part 1
3. Insecure Data Storage – Part 1
4. Insecure Data Storage – Part 2
5. Insecure Data Storage – Part 3
6. Insecure Data Storage – Part 4
7. Input Validation Issues – Part 1
8. Input Validation Issues – Part 2
9. Access Control Issues – Part 1
10. Access Control Issues – Part 2
11. Access Control Issues – Part 3
12. Hardcoding Issues – Part 2
13. Input Validation Issues – Part 3
```

## Insecure Logging

![Insecure Logging](/static/img/posts/diva/2.png)

İlgili bölüme girdiğimiz zaman bize amacımızın ne olduğunu ve bir adette ipucunun yazdığını görüyoruz. Hem bölümün adından hemde açıklamadan anlayacağımız üzere güvenli olmayan bir şekilde loglama yapılıyormuş. Bunu anlamak için gene `adb` kullanacağız.

    adb logcat | grep diva

![3](/static/img/posts/diva/3.png)

Evet bu şekilde uygulamamızda girmiş olduğumuz input değerinin rahatlıkla loglardan okunabildiğiniz görüyoruz.

Peki bu zafiyet nerden çıktı ?

![source-code](/static/img/posts/diva/4.png)

Görmüş olduğunuz satırdaki işlemi eğer kaldırırsak sorun ortadan kalkacaktır :)

## Hardcoding Issues – Part 1

![Hardcoding Issues – Part 1](/static/img/posts/diva/5.png)

Bize vermiş olduğu bilgilerden anlıyoruz ki burda bizden istenen `key` kodun içerisinde mevcut durumda.

![source-code](/static/img/posts/diva/6.png)

Koda baktığımız zaman ise düşündüğümüz şeyde yanılmadığımızı anlıyoruz

![7](/static/img/posts/diva/7.png)

## Insecure Data Storage – Part 1

![Insecure Data Storage – Part 1](/static/img/posts/diva/8.png)

Girmiş olduğumuz verinin güvensiz bir şekilde depolandığını düşünüyoruz. Öncelikli olarak bu depolama işleminin nasıl yapıldığına bir bakmamız lazım.

![source-code](/static/img/posts/diva/9.png)

`SharedPreferences` fonksiyonu kullanılmış. Açıkcası tam olarak ne işe yaradığını bilmiyorum ancak uygulamanın dizinine gidince `shared_prefs` isimli bir klasör dikkatimizi çekiyor. Demekki herhangi bir username-password girildiği zaman default olarak buraya yazıyor.

![10](/static/img/posts/diva/10.png)

## Insecure Data Storage – Part 2

![Insecure Data Storage – Part 2](/static/img/posts/diva/11.png)

İpucu ve açıklama bir önceki ile aynı sanırım. Hızlıca kaynak kodu inceleyelim.

![source-code](/static/img/posts/diva/12.png)

Bu sefer veriyi yazmak için `SQLite` veritabanı kullanılmış. `ids2` isimli bir database oluşturuyor. Bu veritabanının içerisine `myuser` isimli bir tablo oluşturuluyor. Bu tablonun içerisinede iki adet kolon oluşturuyor. Girmiş olduğumuz verileride buraya yazıyor.

![13](/static/img/posts/diva/13.png)

## Insecure Data Storage – Part 3

![Insecure Data Storage – Part 3](/static/img/posts/diva/14.png)

Gene aynı şeyler yazıyor :( Neyseki kaynak kodu görebiliyoruz

![source-code](/static/img/posts/diva/15.png)

`File ddir = ...` işlemi sanırım bizim uygulamamızın verilerinin vs tutulduğu yeri bulup kaydediyor. Daha sonra `try` işlemi içerisine girdiğimiz zaman gene username-password için alanlarımızın oluştuğunu görüyoruz. `File uinfo = File.createTempFile("uinfo", "tmp", ddir);` bu işlem ile uygulamamızın dosyalarının olduğu dizine ismi uinfo ile başlayan bir dosya oluşturuyor. Daha sonra bu dosyayı okunabilir ve yazılabilir yapıyor. Ve girmiş olduğumuz verileri buraya yazıyor.

![16](/static/img/posts/diva/16.png)

## Insecure Data Storage – Part 4

![Insecure Data Storage – Part 4](/static/img/posts/diva/17.png)

![source-code](/static/img/posts/diva/18.png)

Bu seferde `Environment.getExternalStorageDirectory()` kullanmış. Nedir peki bu diyecek olursanız adındanda anlaşıldığı üzere `ExternalStorage`'a dosyamızı kaydediyor. (/mnt/sdcard/ veya /storage/emulated/0/ olabilir)

`NOT:` Bu fonksiyon android 10 sürümünde kaldırılmış

`NOT-2:` Ayarlardan uygulamamızın `storage` iznini açmayı unutmayın.

![19](/static/img/posts/diva/19.png)

## Input Validation Issues – Part 1

![Input Validation Issues – Part 1](/static/img/posts/diva/20.png)

Burda bizden yapmamızı istediği alışık olduğumuz SQL Injection

![source-code](/static/img/posts/diva/21.png)

Kaynak koda baktığımız zamanda `sqli` ben burdayım diyor :D

![sqli](/static/img/posts/diva/22.png)

## Input Validation Issues – Part 2

![Input Validation Issues – Part 2](/static/img/posts/diva/23.png)

Bir url adresi girdiğimiz zaman bunu alt tarafa yüklüyormuş/gösteriyormuş.

![source-code](/static/img/posts/diva/24.png)

Tabi biz burda amacına uygun kullanacak değiliz :joy: Web zafiyetlerinden aşina olduğumuz `LFI` zafiyetine benzetebiliriz burada oluşan zafiyeti.

![25](/static/img/posts/diva/25.png)

## Access Control Issues – Part 1

![Access Control Issues – Part 1](/static/img/posts/diva/26.png)

`VIEW API CREDENTIALS` butonuna bastığımız zaman bize api bilgilerini getiriyor. Bizden istenen şey ise bu butona basmadan api bilgilerinin gösterildiği ekrana ulaşabilmek :/

![source-code](/static/img/posts/diva/27.png)

Kaynak koda bakacak olursak eğer kullanmış olduğu aktiviteyi görebiliriz. Sıra geldi bunu biz nasıl kullanacaz sorusuna. Neyse ki bu konuda da `adb` bize yardımcı oluyor :)

    adb shell am start -a jakhar.aseem.diva.action.VIEW_CREDS

![28](/static/img/posts/diva/28.png)

## Access Control Issues – Part 2

![Access Control Issues – Part 2](/static/img/posts/diva/29.png)

`Register Now.` dersek eğer bizden bir "PIN" istiyor. `Already Registered.` dersek eğer api bilgilerini getiriyor. Yani anlayacağınız direk açılmasın diye ufak bir kontrol gibi bişi yapmışlar

![source-code](/static/img/posts/diva/30.png)

`https://github.com/payatu/diva-android/blob/master/app/src/main/AndroidManifest.xml`

```java
...
<activity
    android:name=".APICreds2Activity"
    android:label="@string/apic2_label" >
    <intent-filter>
        <action android:name="jakhar.aseem.diva.action.VIEW_CREDS2" />

        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
...
```

`https://github.com/payatu/diva-android/blob/master/app/src/main/res/values/strings.xml`

```java
...
    <string name="chk_pin">check_pin</string>
...
```

Şimdi bu şekilde bakacak olursak eğer `check_pin` değerimiz true olduğu zaman istediğimiz api bilgilerini göremiyoruz. Yani bu değeri false yapmamız gerekecek. Tabi bunun içinde bir parametremiz mevcut :)

    $ adb shell
    root@vbox86p:/ # am --help

![am--help](/static/img/posts/diva/31.png)

    adb shell am start -a jakhar.aseem.diva.action.VIEW_CREDS2 -n jakhar.aseem.diva/.APICreds2Activity --ez check_pin false

![32](/static/img/posts/diva/32.png)

## Access Control Issues – Part 3

![Access Control Issues – Part 3](/static/img/posts/diva/33.png)

![source-code](/static/img/posts/diva/34.png)

Bu kısımda da bizden aldığı pin değerini uygulamanın bulunduğu dizinde `shared_prefs` dizini altına kaydetmekte

![35](/static/img/posts/diva/35.png)

## Hardcoding Issues – Part 2

![Hardcoding Issues – Part 2](/static/img/posts/diva/36.png)

Developer arkadaş gene bir yerlere `key` saklamış onu bulmamızı istiyor.

![source-code](/static/img/posts/diva/37.png)

Burda dikkatimizi `private DivaJni djni;` satırı çekiyor. Çünki `key` oradan geliyor.

Bunun için `DivaJni.java` isimli dosyamıza gidiyoruz.

![38](/static/img/posts/diva/38.png)

`System.loadLibrary(soName);` sistemden bir adet kütüphane ekliyor sisteme. O halde hedefimiz ` diva-android/app/src/main/jni/divajni.c` dosyası olacak. ("soName = divajni")

![39](/static/img/posts/diva/39.png)

O halde aradığımız `key` = `olsdfgad;lh`

![40](/static/img/posts/diva/40.png)

## Input Validation Issues – Part 3

![Input Validation Issues – Part 3](/static/img/posts/diva/41.png)

![source-code](/static/img/posts/diva/42.png)

Eğer çok fazla karakter girerseniz bellek alanı yeterli gelmeyecek ve uygulama crash olacaktır.

![43](/static/img/posts/diva/43.png)

![44](/static/img/posts/diva/44.png)
