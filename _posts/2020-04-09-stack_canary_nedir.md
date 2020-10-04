---
layout: post
title:  "Stack Canary Bypass"
author: "Alp Eren Işık"
date:   2020-04-09
categories: [binary-exploitation]
permalink: /:year/:month/:day/:title:output_ext
---

# `Stack Canary Nedir ?`

Progam çalıştığı zaman, henüz başlamadan önce, bir cookie değeri oluşturur (4 byte veya 8 byte uzunluğunda) ve bu değeri `.data` bölümünde saklar. 4 byte uzunluğundaki değerler (yani 32 bit sistemlerde) `GS` , 8 byte uzunluğundaki değerler (yani 64 bit sistemlerde) `FS` flagleri içerisinde saklanır.

> FS , GS  :  İhtiyaca göre ek olarak kullanılan Data Segment Registerlarıdır

Daha sonra oluşturulan bu `stack cookie` değeri Buffer alanı ile EBP arasına yerleştiriliyor. Böylece biz buffer alanını taşırdığımız zaman EBP değerinden önce cookie'nin değerini değiştirmiş oluyoruz. Program bu değer ile kaydettiği değerin farklı olduğunu anlayınca da bir hata vererek exploitimizi çalıştırmıyor.  

> [buffer] [cookie] [EBP] [EIP]

Stack Canary korumasının genel mantığı bu şekilde. Şimdi böyle bir koruma ile karşılaştığımızda ne yapabiliriz onu görelim.

`NOT : ` Oluşturulan cookie değerlerinin her zaman ilk byte'ı `0x00` olacaktır. Yani aslında 32 bit sistemlerde 3 byte uzunluğunda 64 bit sistemlerde 7 byte uzunluğunda bir değerimiz olacak. (`0x00` little-endian olarak düşününce en sondaki byte , big-endian olarak düşününce ilk byte olacaktır)

# `Stack Canary Bypass`

`Stack canary bypass` aşamasında kullanabileceğimiz (en azından benim bildiğim) iki teknik mevcut.

Birincisi `cookie` değerini bruteforce ile bulmaya çalışmak. Bu yöntemle 32bit mimarilerde `cookie` değerini tutturma şansımız biraz daha yüksekken 64bit mimarilerde şansımız düşük olacaktır. Üstelik sürekli olarak karşı tarafa exploit denemesi yapmamızda göze çarpacaktır.

İkinci yöntemde ise karşı tarafta bir `format string zafiyeti` bularak stack içerisinden `cookie` değerini dışarı çıkarabiliriz.

Bu yazıda bruteforce tekniğinden ziyade 64bit mimarilerde format string zafiyetini kullanarak cookie'yi nasıl elde edebiliriz buna değineceğim.


### `Code`


    #include <stdlib.h>
    #include <unistd.h>
    #include <stdio.h>
    #include <string.h>

    void name() {
      char buffer[64];
      printf("What is your name ? ");
      fgets(buffer, sizeof(buffer), stdin);
      printf("\nHello ");
      printf(buffer);
    }

    void vuln(char *string) {
      char buff[64];
      strcpy(buff, string);
      printf("\nFirst argv : ");
      printf(buff);
    }

    int main(int argc, char **argv) {
      name();
      vuln(argv[1]);
    }


***

Kodu derlerken de `stack canary` korumasının aktif olduğundan emin olmak için aşağıdaki gibi derleyebilirsiniz.

    gcc canary.c -o canary -fstack-protector-all

NOT : Herhangi bir programda stack canary koruması açık olduğu zaman çalıştırma anında veya gdb (debugger) ile analiz ederken şu tarz bir hata ile karşılaşırsınız.  

![test](/static/img/posts/stack_canary/0.png)

Veya aşağıdaki komutlardan birini çalıştırarak `stack canary` korumasının aktif olup olmadığını öğrenebilirsiniz.

![test](/static/img/posts/stack_canary/1.png)

Evet şimdi programımızı gdb ile açıp incelemeye başlayalım.

![gdb](/static/img/posts/stack_canary/2.png)

Debugger ile incelediğimiz zaman gördüğünüz gibi `main+15`'de cookie değeri `rax` registerına aktarılmış. Daha sonra `name ve vuln` isimli fonksiyonlar çağrılmış. En sonda da `main+68`'de xor ile cookie değeri kontrol edilmiş. Bu kontrol sonucunda ise eğer değer aynıysa program başarılı bir şekilde devam edip sonlanıyor , aynı değilse program hata mesajını çağırıp kendini sonlandırıyor. (Aynı işlemler her fonksiyon içerisinde tekrarlanıyor.)

Programımızın kaynak kodunu incelediğimizde çalıştırmadan önce `vuln` isimli fonksiyon herhangi bir değer istiyor. Bu değer ile beraber programımızı çalıştırdıktan sonra da bizden adımızın ne olduğunu girmemizi istiyor. En sonda girdiğimiz değerleri ekrana bastırıyor. (Bu kodu blog yazısını yazarken anlatım yapmak için hızlı bir şekilde hazırladığım için kurgusu biraz saçma olabilir :D kusura bakmayın ).

Evet programımızın nasıl çalıştığını anladıktan sonra burada dikkatimizi 2 şey çekiyor. `GCC` ile derlerken hata vermeyen ancak zafiyete yol açan iki tane fonksiyon kullanılmış. `fgets()` fonksiyonu format string zafiyetine `strcpy()` fonksiyonu ise buffer overflow zafiyetine yol açmakta. (Bu fonksiyonlar yerine `gets()` fonksiyonunu kullanırsanız muhtemelen ufak bir uyarı alacaksınızdır).

Bunları da kenara not ettikten sonra `cookie` değerini ve offset değerimizi bulalım. Bu aşamada format string zafiyetinden faydalanırken `%lx` kullanırsanız çektiğiniz değerleri daha rahat okursunuz. Buffer overflow tarafında da kullandığınız debugger a göre `pattern create` kullanarak bulabilirsiniz. (Bu aşamaları hızlıca geçiyorum...)

Şimdi programımızda `vuln veya name` fonksiyonu içerisinde `xor` işleminin olduğu yere birer breakpoint koyuyorum. Daha sonra `'A'*72` ve `'%lx'*15` karakterlerini girerek çalıştırıyorum.

![run](/static/img/posts/stack_canary/3.png)

Ve gördğünüz gibi 15. sırada `rax` registerında bulunan değeri okuyarak `stack cookie` değerini elde etmiş oldum.

`name` fonksiyonundan çıkıp `vuln`'daki breakpointimizin olduğu kısma gelip `stack` içerisinde neler olmuş bakalım.

![run](/static/img/posts/stack_canary/4.png)

72 adet A (0x41) karakterimizin ardından gördüğünüz gibi cookie adresi yer almakta. Bu cookie adresinden sonra da base pointer (rbp) adresi geliyor. Böylece yukarda anlatmiş olduğumuz `[buffer] [cookie] [rbp]` sıralamasını stack içerisinde görmüş oluyoruz.

Bu aşamadan sonra açıkcası `exploit` aşamasınıda yazmayı düşünüyordum ancak yazının çok uzayacağını düşündüğüm için (diğer koruma yöntemlerinin de tek tek atlatılması gerekecek) bu yazıda buna değinmeyeceğim (Vaktim olursa başka bir yazıda detaylıca nasıl exploit yazabileceğimize değinirim).

Genel hatlarıyla `stack canary`'nin ne olduğundan ve bu `cookie` değerini nasıl elde edebileceğinizden bahsettim. Hatalı veya eksik olduğum bir konu varsa geri dönüş yaparsanız sevinirim :)
