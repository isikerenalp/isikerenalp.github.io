---
layout: post
title:  "CyberTruckChallenge19 Writeup"
author: "Alp Eren Işık"
date:   2021-02-09
categories: [writeup, mobile-security]
permalink: /:year/:month/:day/:title:output_ext
---

# CyberTruckChallenge19 Writeup

https://github.com/nowsecure/cybertruckchallenge19

```
A new mobile remote keyless system "CyberTruck" has been implemented by one of the most well
known car security companies "NowSecure Mobile Vehicles". The car security company has ensured
that the system is entirely uncrackable and therefore attackers will not be able to recover
secrets within the mobile application.

If you are an experienced Android reverser, then enable the tamperproof button to harden the
application before unlocking your cars. Your goal will consist on recovering up to 6 secrets in
the application.
```

Çelıncımızın yapmış olduğu açıklama bu şekilde. 3 statik 3 dinamik flag saklamışlar. Bunları bulmamız beklenmekte. O halde başlayalım...

![Challenge](/static/img/posts/cybertruckchallenge/cybertruckchallenge.png)

## `Challenge1.1 - Static Analysis`

Öncelikle AndroidManifest.xml'i bir açalım

```java
// AndroidManifest
<activity android:name="org.nowsecure.cybertruck.MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
</activity>
```

Programımız `org.nowsecure.cybertruck.MainActivity`'den başlıyormuş.

```java
...
import org.nowsecure.cybertruck.keygenerators.Challenge1;
import org.nowsecure.cybertruck.keygenerators.a;
...
public void k() {
    new Challenge1();
    new a(j);
    init();
}
...
final Button button = (Button) findViewById(R.id.unlock_car);
button.setOnClickListener(new View.OnClickListener() {
    /* class org.nowsecure.cybertruck.MainActivity.AnonymousClass3 */

    public void onClick(View view) {
        if (button != null) {
            Toast.makeText(MainActivity.this.getApplicationContext(), "Unlocking cars...", 0).show();
            MainActivity.this.k();
        }
    }
});
```

Ne demiş `MainActivity.this.k();` . Peki bu fonksiyonun içinde ne yazıyor `new Challenge1();`. Bu çelınç bir nerden gelmiş `org.nowsecure.cybertruck.keygenerators.Challenge1` 'den gelmiş.

Eee o zaman Challenge1'i bulduğumuza göre gidip içine bakalım.

![0](/static/img/posts/cybertruckchallenge/0.png)

Ve statik analizle bulmamızı istediği flag burda :)

`s3cr3t$_n3veR_mUst_bE_h4rdc0d3d_m4t3!`

## `Challenge1.2 - Dynamic Analysis`

```java
public Challenge1() {
    generateKey();
}

/* access modifiers changed from: protected */
public byte[] generateDynamicKey(byte[] bArr) {
    SecretKey generateSecret = SecretKeyFactory.getInstance("DES").generateSecret(new DESKeySpec("s3cr3t$_n3veR_mUst_bE_h4rdc0d3d_m4t3!".getBytes()));
    Cipher instance = Cipher.getInstance("DES");
    instance.init(1, generateSecret);
    return instance.doFinal(bArr);
}

/* access modifiers changed from: protected */
public void generateKey() {
    Log.d(TAG, "KEYLESS CRYPTO [1] - Unlocking carID = 1");
    try {
        generateDynamicKey("CyB3r_tRucK_Ch4113ng3".getBytes());
    } catch (InvalidKeyException | NoSuchAlgorithmException | InvalidKeySpecException | BadPaddingException | IllegalBlockSizeException | NoSuchPaddingException e) {
        e.printStackTrace();
    }
}
```

Kodumuzu satır satır okuyalım. Öncelikle napmış generateKey() isimli fonksiyonu çağırmış demi. Hop gittik hemen o fonksiyona. try catch yapısı var ve try içerisinde generateDynamicKey() isimli fonksiyona bir değer göndermiş (byte cinsinden hemde). Hop zıpladık hemen generateDynamicKey fonksiyonuna. Ve gördük ki burda bir cipher işlemi var.

Demek ki ben eğer generateDynamicKey fonksiyonunu frida ile hooklarsam return edilen değeri öğrenebilirim.

```js
// https://github.com/nowsecure/cybertruckchallenge19/blob/35bfd62de45948cfb16976d14b00ba59c60e3910/student/student.js#L44
function ba2hex(bufArray) {
    var uint8arr = new Uint8Array(bufArray);
    if (!uint8arr) {
        return '';
    }

    var hexStr = '';
    for (var i = 0; i < uint8arr.length; i++) {
        var hex = (uint8arr[i] & 0xff).toString(16);
        hex = (hex.length === 1) ? '0' + hex : hex;
        hexStr += hex;
    }

    return hexStr.toLowerCase();
}

Java.perform(function () {
	var challenge1 = Java.use("org.nowsecure.cybertruck.keygenerators.Challenge1")
	challenge1.generateDynamicKey.implementation = function(barr) {
		var ret = this.generateDynamicKey(barr);
		console.log("Challenge 1 Flag : ", ba2hex(ret));
		return ret;
	}

});
```

NOT : ba2hex fonksiyonunu bizim için hazırlamışlar :D ben de direk onu kullandım. Üstünde de fonksiyonu aldığım github adresi mevcut

Frida ile js kodumuzu çalıştırıp daha sonra da programda butona tıklarsak...

`frida -U -l challenge1.js -f org.nowsecure.cybertruck --no-pause`

![1](/static/img/posts/cybertruckchallenge/1.png)

`046e04ff67535d25dfea022033fcaaf23606b95a5c07a8c6`

## `Challenge2.1 - Static Analysis`

Challenge1'in statik analiz kısmında hatırlayacak olursanız eğer bizim şöyle bir fonksiyonumuz vardı.

```Java
...
import org.nowsecure.cybertruck.keygenerators.Challenge1;
import org.nowsecure.cybertruck.keygenerators.a;
...
public void k() {
    new Challenge1();
    new a(j);
    init();
}
```

Burda Challenge1'e baktık peki ya `new a(j);` dediği yerde ne var ? Bu bizim Challenge2'miz olabilir mi acaba ?

`org.nowsecure.cybertruck.keygenerators.a`

```Java
...
public byte[] a(Context context) {
     InputStream inputStream;
     Log.d("CyberTruckChallenge", "KEYLESS CRYPTO [2] - Unlocking carID = 2");
     StringBuilder sb = new StringBuilder();
     String str = null;
     try {
         inputStream = context.getAssets().open("ch2.key");
     } catch (IOException e) {
         e.printStackTrace();
         inputStream = null;
     }
     BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(inputStream));
     while (true) {
         try {
             String readLine = bufferedReader.readLine();
             if (readLine == null) {
                 return sb.toString().getBytes();
             }
             str = readLine;
             sb.append(str);
         } catch (IOException e2) {
             e2.printStackTrace();
         }
     }
 }
...
```

Evet bu bizim Challenge2'miz. Şimdi dikkatlice şu fonksiyonu okursak eğer `ch2.key` isimli bir dosyayı açtığını görüyoruz. Bu dosyada assets dizininde bulunmakta. (context.getAssets().open yazmış ordan anladık :D )

Hızlıca GitHub üzerinden bakmak isteyenler için => [https://github.com/nowsecure/cybertruckchallenge19/blob/master/src/app/src/main/assets/ch2.key](https://github.com/nowsecure/cybertruckchallenge19/blob/master/src/app/src/main/assets/ch2.key)

![2](/static/img/posts/cybertruckchallenge/2.png)

`d474_47_r357_mu57_pR073C73D700!!`


## `Challenge2.2 - Dynamic Analysis`

Dinamik analiz için kodu iyi anlamamız lazım zira burda tam 3 dene `a()` fonksiyonu var :D Gerçi cipher işlemini yapan fonksiyon belli ama olsun.

```java
public a(Context context) {
    try {
        a("uncr4ck4ble_k3yle$$".getBytes(), a(context));
    } catch (InvalidKeyException | NoSuchAlgorithmException | BadPaddingException | IllegalBlockSizeException | NoSuchPaddingException e) {
        e.printStackTrace();
    }
}
```

İlk a fonksiyonu `Constructor`. try bloğu içerisinde bir a fonksiyonu daha çalışıyor ve onun içerisinde de ikinci parametre olarak başka bir a fonksiyonu çalışıyor. Şimdi burda ki `a(context)` yazan fonksiyon bizim demin yukarda statik analizde baktığımız a fonksiyonu (bir değer/değişken alıp byte array döndürüyor). İki tane parametre almış olan a fonksiyonu ise asıl şifrelemeyi yapan fonksiyon. (iki parametrede byte cinsinden)

```Java
public byte[] a(byte[] bArr, byte[] bArr2) {
    SecretKeySpec secretKeySpec = new SecretKeySpec(bArr2, "AES");
    Cipher instance = Cipher.getInstance("AES/ECB/PKCS7Padding");
    instance.init(1, secretKeySpec);
    return instance.doFinal(bArr);
}
```

O zaman neyi hooklayacağımızı biliyoruz. Kolları sıvayalım ve işe koyulalım...

```js
// https://github.com/nowsecure/cybertruckchallenge19/blob/35bfd62de45948cfb16976d14b00ba59c60e3910/student/student.js#L44
function ba2hex(bufArray) {
    var uint8arr = new Uint8Array(bufArray);
    if (!uint8arr) {
        return '';
    }

    var hexStr = '';
    for (var i = 0; i < uint8arr.length; i++) {
        var hex = (uint8arr[i] & 0xff).toString(16);
        hex = (hex.length === 1) ? '0' + hex : hex;
        hexStr += hex;
    }

    return hexStr.toLowerCase();
}


Java.perform(function() {

	var challenge2 = Java.use("org.nowsecure.cybertruck.keygenerators.a");
	challenge2.a.overload("[B", "[B").implementation = function(bArr1, bArr2) {
		var ret = this.a(bArr1, bArr2);
		console.log("Challenge 2 Flag : ", ba2hex(ret));
		return ret;
	}

});
```

`frida -U -l challenge2.js -f org.nowsecure.cybertruck --no-pause`

Kodu çalıştıralım, uygulamamızda butona basalım ve...

![3](/static/img/posts/cybertruckchallenge/3.png)

Bingo ! Flag => `512100f7cc50c76906d23181aff63f0d642b3d947f75d360b6b15447540e4f16`

## `Challenge3.1 - Static Analysis`

Son kısım için tekrardan AndroidManifest içerisine bakacak olursak bir kütüphane yüklendiğini görüyoruz

```Java
static {
    System.loadLibrary("native-lib");
}
...
public void k() {
    new Challenge1();
    new a(j);
    init();
}
...
```

    strings libnative-lib.so

![4](/static/img/posts/cybertruckchallenge/4.png)

Flag `Native_c0d3_1s_h4rd3r_To_r3vers3,`

## `Challenge3.2 - Dynamic Analysis`

`libnative-lib.so` dosyamızı ghidra üzerinde açtığımız zaman fonksiyonlar kısmında direk `init()` fonksiyonu gözümüze çarpıyor.

![5](/static/img/posts/cybertruckchallenge/5.png)

Biraz daha detaylı incelersek...

![6](/static/img/posts/cybertruckchallenge/6.png)

Bir xor işlemi var. Disassemble haline bakalım ve XOR işleminin olduğu adresi bulalım.

![7](/static/img/posts/cybertruckchallenge/7.png)

Peki bunlar ne işe yarayacak ?

Şöyle ki eğer XOR işlemi bittikten sonra `eax` registerında yazan değeri okursak aradığımız değeri bulmuş oluruz :)

Frida'nın nimetleri saymakla bitmiyor :D O zaman let's go...

```js
Java.perform(function() {
	var main = Java.use("org.nowsecure.cybertruck.MainActivity");
	main.k.implementation = function() {
		var secret = "";
		var len = 0;

		var hook = Interceptor.attach(Module.findBaseAddress("libnative-lib.so").add(0x6d1), {
			onEnter: function(args) {
				if(len++ < 32) {
					secret += String.fromCharCode(this.context.eax);
				} else {
					console.log("Flag : ", secret);
					hook.detach();
				}
			}
		});

		this.k();
	}
});
```

    frida -U -l challenge3.js -f org.nowsecure.cybertruck --no-pause

Kodumuzu çalıştırıp butona tıklarsak...

![8](/static/img/posts/cybertruckchallenge/8.png)

`backd00r$Mu$tAlw4ysBeF0rb1dd3n$$`


## TamperProof Bypass

Uygulamayı açtığımızda TamperProof diye bir aç-kapa buton var (ona tam olarak ne isim veriyolar bilmiyom :( neyse artık). O özelliği açtığımız zaman frida'yı tespit ediyor ve kapatıyor. Daha öncelerde de bu tarz kontrolleri atlatmıştık o yüzden hızlıca bypass kodumuzu yazalım.

Buton kodu bu..

```java
public void onCheckedChanged(CompoundButton compoundButton, boolean z) {
    if (z && new HookDetector().isFridaServerInDevice()) {
        Toast.makeText(MainActivity.j, "Tampering detected!", 0).show();
        new CountDownTimer(2000, 1000) {
            /* class org.nowsecure.cybertruck.MainActivity.AnonymousClass1.AnonymousClass1 */

            public void onFinish() {
                System.exit(0);
            }

            public void onTick(long j) {
            }
        }.start();
    }
}
```

Ve HookDetector kodu da bu...

```Java
public class HookDetector {
    private static final String TAG = "CyberTruckChallenge";
    private static final String classname = HookDetector.class.getSimpleName();

    public boolean isFridaServerInDevice() {
        if (!new File("/data/local/tmp/frida-server").exists() && !new File("/data/local/tmp/re.frida.server").exists() && !new File("/sdcard/re.frida.server").exists() && !new File("/sdcard/frida-server").exists()) {
            return false;
        }
        Log.d(TAG, "TAMPERPROOF [0] - Hooking detector trigger due to frida-server was found in /data/local/tmp/");
        return true;
    }
}
```

isFridaServerInDevice fonksiyonu false döndüğü sürece bir problem yok gibi.

```js
Java.perform(function() {
	var detector = Java.use("org.nowsecure.cybertruck.detections.HookDetector");
	detector.isFridaServerInDevice.implementation = function() {
		console.log("Hook Detector Bypass !");
		return false;
	}
});
```

![9](/static/img/posts/cybertruckchallenge/9.png)

Ve bu şekilde frida kontrolünü atlatmış olduk :)
