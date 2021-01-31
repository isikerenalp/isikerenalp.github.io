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

`Uncrackable Level 3` uygulamamızı telefonumuza yükledikten sonra açalım...

![giris](/static/img/posts/owasp_uncrackable_android/3/0.png)

Diğer seviyelerden farklı olarak sadece root detect değil aynı zamanda tampering detect mesajıda veriyor. Peki ya bu nedir ?

`sg.vantagepoint.uncrackable3.MainActivity`

```java
if (RootDetection.checkRoot1() || RootDetection.checkRoot2() || RootDetection.checkRoot3() || IntegrityCheck.isDebuggable(getApplicationContext()) || tampered != 0) {
    showDialog("Rooting or tampering detected.");
}
```

`tampered` diye birşey var hemen dikkatimi o çekiyor. Hemen kullanıldığı yerlere bakalım.

![1](/static/img/posts/owasp_uncrackable_android/3/1.png)

`private native long baz();`

İki if bloğunda da CRC checksum var. Bunlardan ilki libfoo.so dosyasının değerini strings.xml dosyasında kayıtlı olan değerle karşılaştırıyor. İkincisi ise classes.dex dosyasının değerini libfoo.so dosyası içerisinde bulunan değerle karşılaştırıyor.

İnanmıyorum ben sana nerde karşılaştırıyor (:D olabilir böyle şeyler) derseniz eğer şu şekilde değerlere bakıp not edelim ve daha sonra logcat çıktısındaki değerlerle karşılaştıralım.

![2](/static/img/posts/owasp_uncrackable_android/3/2.png)

`private native long baz();`

![3](/static/img/posts/owasp_uncrackable_android/3/3.png)

`adb shell logcat`

```bash
...
  3103  3103 V UnCrackable3: CRC[lib/arm64-v8a/libfoo.so] = 1608485481
  3103  3103 V UnCrackable3: CRC[lib/x86_64/libfoo.so] = 2856060114
  3103  3103 V UnCrackable3: CRC[lib/armeabi-v7a/libfoo.so] = 881998371
  3103  3103 V UnCrackable3: CRC[lib/x86/libfoo.so] = 1618896864
  3103  3103 V UnCrackable3: CRC[classes.dex] = 25235683
...
```

Her neyse burası bu şekilde işte. Yolumuza devam edelim ve daha önce yazdığımız root detect bypass kodunu çalıştıralım.

```js
Java.perform(function() {
    console.log("[+]Root detect bypass !");
    var detect = Java.use("java.lang.System");
    detect.exit.implementation = function() {
        console.log("system.exit func was called");
    };
});
```

![4](/static/img/posts/owasp_uncrackable_android/3/4.png)

Noldu processes crash oldu. Neden crash oldu ? Çünki program bizim frida çalıştırdığımızı çaktı ve crash ettirdi. İyide nerden anladı bunu ?  

![5](/static/img/posts/owasp_uncrackable_android/3/5.png)

NOT : `_INIT_0` isimli fonksiyona giderseniz eğer `pthread_create(&local_24,(pthread_attr_t *)0x0,FUN_00013080,(void *)0x0);` şöyle bir satırla karşılaşacaksınız.

İşte tam olarak böyle bir fonksiyon sayesinde anladı. İyi güzel hoş da bunu nasıl atlatacaz 🤔 Öncelikle kısaca bunun nasıl çalıştığını anlamak lazım.

`/proc/self/maps` okuyor ve bir değişkene (array) atıyor (0x200 yani 512). Daha sonra strstr() fonksiyonu ile bu dizenin içerisinde `frida` ve `xposed` var mı ona bakıyor. Eğer bulursa naptını zaten biliyoruz :D. Peki strstr fonksiyonunda ne var bi de ona bakalım.

![6](/static/img/posts/owasp_uncrackable_android/3/6.png)

Evet o zaman elimizdeki bilgileri kullanırsak...

```js
Java.perform(function() {
    console.log("[+]Root detect bypass !");
    var detect = Java.use("java.lang.System");
    detect.exit.implementation = function() {
        console.log("system.exit func was called");
    };
});

Interceptor.attach(Module.findExportByName("libc.so", "strstr"), {
	onEnter: function(args) {
		this.frida = Boolean(0);
		var haystack = Memory.readUtf8String(args[0]);
		var needle = Memory.readUtf8String(args[1]);
		if(haystack.indexOf("frida") != -1) {
			this.frida = Boolean(1);
		}
	},
	onLeave: function(retval) {
			if(this.frida){
				retval.replace(0);
			}
			return retval;
		}
});
```

şöyle bir script ortaya çıkacak.

[https://frida.re/docs/javascript-api/#interceptor](https://frida.re/docs/javascript-api/#interceptor)

![7](/static/img/posts/owasp_uncrackable_android/3/7.png)

Artık uygulamamıza erişim sağlayabiliyoruz. Kodu okumaya devam edelim (bizden istenen değer nerden nasıl gidiyor acaba)

`sg.vantagepoint.uncrackable3.MainActivity`

```java
...
   private static final String xorkey = "pizzapizzapizzapizzapizz";
   private native void init(byte[] bArr);
...
public void onCreate(Bundle bundle) {
    verifyLibs();
    init(xorkey.getBytes());
...
}
...
static {
    System.loadLibrary("foo");
}
```

Main fonksiyonumuzda bizim için önemli olan 3 nokta. Sondaki bizim yüklenen native library miz.  onCreate fonksiyonun içindeki verifyLibs() fonksiyonu az önce baktığımız root, tamper detect işlemini yapan fonksiyon. En üstte bir xorkey tanımlanmış. Adından da anlaşıldığı üzere demek ki bir xor işlemi var :D Ve son olarak da yüklediğimiz native library den gelen bir init fonksiyonu var ki ona da xorkey'i gönderiyoruz. Demek ki burda gene bi işimiz yok tekrardan ghidra'ya dönelim.

![8](/static/img/posts/owasp_uncrackable_android/3/8.png)

Bizden aldığı değeri ise bu sefer yerel bir değişkene kaydediyor. (baştaki o fun_00013250 fonksiyonu sanırım anti_debug için kullanılmakta)

`strncpy((char *)&DAT_0001601c,__src,0x18);`

Bu değişkeni (DAT_0001601c) biz tanımlamadık nerden geldi bu derseniz eğer tabi ki de `Java_sg_vantagepoint_uncrackable3_CodeCheck_bar` fonksiyonundan.

![9](/static/img/posts/owasp_uncrackable_android/3/9.png)

Burda bizim `xorkey` değişkeninde olan değer (pizzapizzapizzapizzapizz (artık puVar5 değişkeninde)) ile başka değer (local_40 değişkeninde olan değer) xor işlemine tutuluyor ve daha sonra bizden aldığı argüman ile karşılaştırıyor. (`FUN_00010fa0` fonksiyonu baya uzun o yüzden o kısmı eklemedim. İkinci xor değeri burdan geliyor). Biz de frida ile xor'da kullandığı ikinci değeri ortaya çıkaracağız. Sonra da manuel olarak kendimiz xor işlemini yapacağız ve istenen argümanı öğreneceğiz.

Bunun için kütüphanemizin içerisindeki fonksiyonu hooklayan bir `hook_libs` fonksiyonu ekleyeceğim kodumuzun başına. Daha sonra native kütüphanemizin doğru şekilde yüklendiğinden emin olman için `Java.perform` içerisinde `sg.vantagepoint.uncrackable3.MainActivity` isimli arkadaşa ufak bir dokunuş yapıp devamında da yazmış olduğumuz fonksiyonu çağıracağım.

```javascript
function hook_libs() {
	console.log("hook_libs func");
	var offset_fun = 0x00000fa0;
	var libfo = Module.findBaseAddress("libfoo.so");
	var secret_fun = libfo.add(offset_fun);

	Interceptor.attach( secret_fun, {
		onEnter: function(args) {
			this.secret = args[0];
			console.log("onEnter()");
		},
		onLeave: function(retval) {
			console.log("onLeave()");
			console.log(Memory.readByteArray(this.secret, 24));
		}
	});
}


Java.perform(function() {

    var detect = Java.use("java.lang.System");
    detect.exit.implementation = function() {
        console.log("system.exit func was called");
    };

    var mainfunc = Java.use("sg.vantagepoint.uncrackable3.MainActivity");
    mainfunc.onStart.overload().implementation = function() {
	console.log("sg.vantagepoint.uncrackable3.MainActivity was called");
	var ret = this.onStart.overload().call(this);
    };

    hook_libs();

});

Interceptor.attach(Module.findExportByName("libc.so", "strstr"), {
	onEnter: function(args) {
		this.frida = Boolean(0);
		var haystack = Memory.readUtf8String(args[0]);
		var needle = Memory.readUtf8String(args[1]);
		if(haystack.indexOf("frida") != -1) {
			this.frida = Boolean(1);
		}

	},
	onLeave: function(retval) {
			if(this.frida){
				retval.replace(0);
			}
			return retval;
		}

});
```

Şimdi burda şöyle bişi var. Bizim offset adresimiz ghidra üzerinde baktığımız zaman `00010fa0` ama bu yanlış fazlalık var. Asıl offset `0x00000fa0` olacak yani o aradaki `1` değerini `0` yapacağız. Eğer sizde adres farklı olursa değiştirmeyi unutmayın. (ben x86 üzerinde yapıyorum belki siz x86_x64 veya arm üzerinde vs yaparsanız benim gibi hata nerde diye kafayı yemeyin 😅)

Konuyla alakalı detaylı bir anlatım mevcut, fonksiyonları vs daha okunaklı hale getirip güzel güzel anlatmış. Orayada bakmanızı tavsiye ederim. [https://enovella.github.io/android/reverse/2017/05/20/android-owasp-crackmes-level-3.html](https://enovella.github.io/android/reverse/2017/05/20/android-owasp-crackmes-level-3.html)

![10](/static/img/posts/owasp_uncrackable_android/3/10.png)


```python
xorkey = "pizzapizzapizzapizzapizz"
secret_key = "1d0811130f1749150d0003195a1d1315080e5a0017081314".decode("hex")

def xorla(x, y):
	return "".join(chr(ord(a) ^ ord(b)) for a,b in zip(x, y))

mesaj = xorla(xorkey, secret_key)
print(mesaj)
```

```bash
$ python2.7 xor.py
making owasp great again
```

![11](/static/img/posts/owasp_uncrackable_android/3/11.png)

Evet böylelikle bu levelide tamamlamış olduk. Alternatif olarak bir yöntem daha söyleyeceğim :)

![11](/static/img/posts/owasp_uncrackable_android/3/12.png)

Hani o uzun diye atmadığım `FUN_00010fa0` fonksiyonu vardıya. Heh işte eğer onun en sonuna gidip param_1'i sırasıyla tersten okursanız eğer 1d081113... şeklinde gittiğini görebiliriz 😂 Yani aslında frida ile hooklamadan da bu şekilde statik analiz ile xor işleminde kullandığı ikinci key'i elde edebiliriz. Diğerini de zaten kodun içine yazmış. Eh telefonda rootlu olmasaydı aslında (o zaman tabi root detect bypass için frida kullanmak ve daha sonra frida detect için tekrar frida kullanmak zorunda kalmazdık) böyle statik analiz ile de çözülebilirmiş.

Evet bu serinin son yazısıydı. Aslında level 4 'de geldi ama o biraz uzun o yüzden onu ayrı bir yazı yapacam (umarım :D).


### Kaynaklar

  - https://github.com/randorisec/workshops/tree/master/BSidesBudapest2020/Android

  - https://github.com/OWASP/owasp-mstg/tree/master/Crackmes

  - https://enovella.github.io/android/reverse/2017/05/20/android-owasp-crackmes-level-3.html
