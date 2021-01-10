---
layout: post
title:  "OWASP UnCrackable Android App Writeup"
author: "Alp Eren Işık"
date:   2020-12-22
categories: [writeup, mobile-security, reverse-engineering]
permalink: /:year/:month/:day/:title:output_ext
---

# OWASP UnCrackable Android Apps Writeup

## Level 1

    adb install UnCrackable-Level1.apk

Uygulamamızı kurduktan sonra açalım bakalım bizi ne karşılayacak (Android 4.4 üstü olması tavsiye ediliyor. Ben android 7 sürümü üzerinde anlatacağım)  

![root detected](/static/img/posts/owasp_uncrackable_android/1/0.png)

Cihazımız rootluydu ve program bunu farkedip bize bir uyarı çıkardı. OK butonuna bastığımız zaman kendini kapatacak. Yani ilk halletmemiz gereken kısım bu `root detect` fonksiyonu olacak. Bunun için ben `jadx` ile açıp inceleyeceğim.

##### AndroidManifest.xml

![AndroidManifest](/static/img/posts/owasp_uncrackable_android/1/1.png)

##### sg.vantagepoint.uncrackable1.MainActivity

![onCreate](/static/img/posts/owasp_uncrackable_android/1/2.png)

##### sg.vantagepoint.a.c

![root detect](/static/img/posts/owasp_uncrackable_android/1/3.png)

Cihazımızın nasıl root olduğunu tespit ettiğine dair yapı şu şekilde kısaca. Burada isterseniz tek tek frida ile 3 fonksiyonun false dönmesini sağlayabilir, isterseniz gene frida ile System.exit() fonksiyonunu manipüle edebilir veya smali kodu içerisindeki bu if bloğunu / fonksiyonu silip tekrar paketleyebilirsiniz.

###### 1. Yöntem

```javascript
Java.perform(function() {
	console.log("[+]Root detect bypass !");

	var detect = Java.use("sg.vantagepoint.a.c");

	detect.a.implementation = function() {
		console.log("a() return false");
		return false;
	};
	detect.b.implementation = function() {
		console.log("b() return false");
		return false;
	};
	detect.c.implementation = function() {
		console.log("c() return false");
		return false();
	};
});
```

    frida -l root-bypass.js -U -f owasp.mstg.uncrackable1 --no-pause

![1. yontem](/static/img/posts/owasp_uncrackable_android/1/4.png)

###### 2. Yöntem

```javascript
Java.perform(function() {
	console.log("[+]Root detect bypass !");

	var detect = Java.use("java.lang.System");

	detect.exit.implementation = function() {
		console.log("system.exit nerdesin ?");
	};
});
```

    frida -l root-bypass.js -U -f owasp.mstg.uncrackable1 --no-pause

![2. yontem](/static/img/posts/owasp_uncrackable_android/1/5.png)

Karşımıza gene `root detected!` yazısı gelecek. Ancak `OK` butonuna bastıktan sonra çıkış işlemi çalışmayacaktır.  

###### 3. Yöntem

![3. yontem](/static/img/posts/owasp_uncrackable_android/1/6.png)

apktool ile apk dosyasını açtıktan sonra (apktool d UnCrackable-Level1.apk) şu dizine gidelim `UnCrackable-Level1/smali/sg/vantagepoint/uncrackable1` . Bu dizinde bulunan `'MainActivity$1.smali'` dosyasında yukarda jadx ile görmüş olduğumuz satır yer almakta.

![7](/static/img/posts/owasp_uncrackable_android/1/7.png)

Burda bulunan `const/4 pl1, 0x0 invoke-static {p1}, Ljava/lang/System;->exit(I)V` kod parçasından kurtulup tekrardan paketlersek eğer programın çıkış yapmasını engellemiş oluruz.

```bash
$ apktool b UnCrackable-Level1 -o UnCrackable-Level1_edited.apk
$ keytool -genkey -v -keystore ctf.keystore -alias ctfKeystore -keyalg RSA -keysize 2048 -validity 10000
$ jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore ctf.keystore UnCrackable-Level1_edited.apk ctfKeystore
$ jarsigner -verify -verbose -certs UnCrackable-Level1_edited.apk
$ adb uninstall owasp.mstg.uncrackable1
$ adb install UnCrackable-Level1_edited.apk
```  

![8](/static/img/posts/owasp_uncrackable_android/1/8.png)

Eğer soruyu böyle çözmek isterseniz `a.smali` içerisine de şu satırları ekleyip paketlerseniz eğer programı çalıştırdıktan sonra loglardan istenilen şeyi öğrenebilirmişsiniz.

Ben frida ile devam edeceğim...

NOT: Bu yöntemler dışında farklı yöntemler kullanarak da root kontrolünü atlatabilirsiniz.

###### Frida Devam

Bizden istenilen değeri bulalım...

sg.vantagepoint.uncrackable1.MainActivity

![9](/static/img/posts/owasp_uncrackable_android/1/9.png)

sg.vantagepoint.uncrackable1.a

![10](/static/img/posts/owasp_uncrackable_android/1/10.png)

sg.vantagepoint.a.a

![11](/static/img/posts/owasp_uncrackable_android/1/11.png)

Evet gördüğümüz üzere AES kullanılmış. Bu fonksiyondan dönen değer bizim girmiş olduğumuz değer ile karşılaştırılıyor. O halde bu fonksiyonu hooklayıp return ettiği değeri ekrana yazdırırsak aradığımızı bulmuş oluruz.

Not: Burda dikkat etmemiz gereken şey dönen değer string tipinde değil gibi o yüzden ufak bi dönüşüme ihtiyacımız olacak.

```javascript
Java.perform(function() {
	console.log("[+]Root detect bypass !");

	var detect = Java.use("java.lang.System");
	detect.exit.implementation = function() {
		console.log("system.exit nerdesin ?");
	};

	var encrypt = Java.use("sg.vantagepoint.a.a");
	encrypt.a.implementation = function(var0, var1) {
		var ret = this.a(var0, var1);
		var flag = '';
		for(var i = 0; i < ret.length;	i++) {
			flag += String.fromCharCode(ret[i]);
		}
		console.log("[+] Decrypted : " + flag);
		return ret;
	};
});
```

![12](/static/img/posts/owasp_uncrackable_android/1/12.png)

Bizden istenen değer `I want to believe` sanırım. Hemen deneyelim.

![13](/static/img/posts/owasp_uncrackable_android/1/13.png)

Ve evet bu şekilde bizden isteneni bulmuş olduk :)

## Level 2

    adb install UnCrackable-Level2.apk

![0](/static/img/posts/owasp_uncrackable_android/2/0.png)

Root detect gördüğümüze göre o zaman koduna bakıp nasıl bir kontrol yaptığına bakalım.

![1](/static/img/posts/owasp_uncrackable_android/2/1.png)

    sg.vantagepoint.uncrackable2.MainActivity

MainActivity'ye girince şu satırları görüyoruz...

```java
import sg.vantagepoint.a.a;
import sg.vantagepoint.a.b;

...
...

private void a(String str) {
    AlertDialog create = new AlertDialog.Builder(this).create();
    create.setTitle(str);
    create.setMessage("This is unacceptable. The app is now going to exit.");
    create.setButton(-3, "OK", new DialogInterface.OnClickListener() {
        /* class sg.vantagepoint.uncrackable2.MainActivity.AnonymousClass1 */

        public void onClick(DialogInterface dialogInterface, int i) {
            System.exit(0);
        }
    });
    create.setCancelable(false);
    create.show();
}

...

    if (b.a() || b.b() || b.c()) {
        a("Root detected!");
    }
    if (a.a(getApplicationContext())) {
        a("App is debuggable!");
    }
```

Demek ki kontroller başka bir yerde yapılıyor.

    sg.vantagepoint.a.b

![2](/static/img/posts/owasp_uncrackable_android/2/2.png)

Level 1 'de bunlardan bahsetmiştik sanırım o yüzden çok üstünde durmayacağım. Frida ile bu kısmı hooklayıp kontrolü geçebiliriz. Bunun içinde gene en kısa yol olan system.exit() fonksiyonunu hooklayacağız

```javascript
Java.perform(function() {
    console.log("[+]Root detect bypass !");
    var detect = Java.use("java.lang.System");
    detect.exit.implementation = function() {
        console.log("system.exit func was called");
    };
});
```

    frida -l level2.js -U -f owasp.mstg.uncrackable2 --no-pause

![3](/static/img/posts/owasp_uncrackable_android/2/3.png)

Bu kısım zaten bildiğimiz birşeydi şimdi gelin doğrulama işlemini nasıl yapıyor onu anlayalım.

    sg.vantagepoint.uncrackable2.MainActivity

Tekrardan MainActivity içerisine baktığımız zaman...

```java
    static {
        System.loadLibrary("foo");
    }
```

Sisteme bir kütüphane yüklendiğini görüyoruz. Geri kalan kısımlarda göze çarpan birşey olmadığı için bunu incelemekte fayda var.

Peki buna nerden ulaşacağız ? Dilerseniz apktool ile dilerseniz unzip veya sıkıştırılmış dosya açabileceğiniz herhangi bir program ile içindeki dosyaları dışarı çıkartın. `lib` klasörü altında kullandığınız mimarideki klasörü açın. Orada aradığımız `libfoo.so` dosyasını bulacağız.

NOT : Bunlar nerden geldi ? Onun o olduğunu nerden anladık ? Bu şekilde kütüphane kullanmanın mantığı nedir ? gibi soru işaretleriniz varsa şuraya bakmanızı tavsiye ederim oldukça güzel bir kaynak. [https://ragingrock.com/AndroidAppRE/reversing_native_libs.html](https://ragingrock.com/AndroidAppRE/reversing_native_libs.html)

Şimdi bu libfoo.so dosyamızı açalım.

![4](/static/img/posts/owasp_uncrackable_android/2/4.png)

Burda bizim bakacağımız `sym.java_sg_vantagepoint_uncrackable2_CodeCheck_bar` olacak. Bunu hemen decompile edip ne yaptığına anlamaya çalışalım. (Ghidra, radare2, cutter, rizin... bunlardan herhangi birini kullanarak decompile edebilirsiniz.)

![5](/static/img/posts/owasp_uncrackable_android/2/5.png)

Şimdi bu kısımda en baştan little endian olarak düşünerek çevirmeye başlarsak aslında ne ile karşılaştırdığını öğrenebiliyoruz.

```python
>>> chr(0x54)
'T'
>>> chr(0x68)
'h'
>>> chr(0x61)
'a'
>>> chr(0x6e)
'n'
>>> chr(0x6b)
'k'
>>> chr(0x73)
's'
>>> chr(0x20)
' '
>>> chr(0x66)
'f'
>>> ...
```

Ancak bu işin kolay, zahmetsiz ve her zaman işe yaramayacak bir yolu. Biz frida ile yolumuza devam edip kütüphane içerisindeki bir fonksiyonu nasıl hooklayabiliriz ona bakalım.

Öncelikli olarak `if(iVar2 == 0x17)` demiş. 0x17 => 23 sayısına eşittir. Yani girilen metnin 23 karakter uzunluğunda olması beklenmekte. Eğer bu koşul sağlanırsa `strncmp` fonksiyonu çalışarak bizim girdimiz ile beklenen girdiyi karşılaştıracak.

`Interceptor.attach` kullanacağımız, ihtiyacımız olan şey bu. Bunun ne yaptığını da şurdan bakabilirsiniz [https://frida.re/docs/javascript-api/#interceptor](https://frida.re/docs/javascript-api/#interceptor). Ayrıca şuna da için bakmanızı öneririm [https://frida.re/docs/javascript-api/#thread](https://frida.re/docs/javascript-api/#thread). Lafı fazla uzatmadan hemen fonksiyonumuzu yazalım, zaten oldukça basit ve anlaşılır.

```javascript
Interceptor.attach(Module.findExportByName("libc.so", "strncmp"), {
	onEnter: function (args) {
		var param1 = Memory.readUtf8String(args[0]);
		var param2 = Memory.readUtf8String(args[1]);
		if(param1.startsWith("AAAA")) {
			console.log("Secret is : " + param2);
		}
	},
	onLeave: function (retval) {
	}
});

Java.perform(function() {
    console.log("[+]Root detect bypass !");
    var detect = Java.use("java.lang.System");
    detect.exit.implementation = function() {
        console.log("system.exit func was called");
    };
});

```

23 tane `A` karakteri gönderelim...

![6](/static/img/posts/owasp_uncrackable_android/2/6.png)

`Thanks for all the fish` :)

## Level 3

### Kaynaklar

  - https://github.com/randorisec/workshops/tree/master/BSidesBudapest2020/Android

  - https://github.com/OWASP/owasp-mstg/tree/master/Crackmes
