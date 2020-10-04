---
layout: post
title:  "Address Space Layout Randomization (ASLR)"
author: "Alp Eren Işık"
date:   2020-02-26
categories: [binary-exploitation]
permalink: /:year/:month/:day/:title:output_ext
---

# `ASLR Nedir ?`

Adından da anlaşılacağı üzere program her çalıştırıldığında `sanal memory` üzerinde stack ve kütüphane adreslerini değiştiren bir koruma yöntemidir. (sanırım sadece .text bölümüne ait adresler sabit kalıyormuş)

Yani şu şekilde...

![1](/static/img/posts/ASLR/1.png)  

![2](/static/img/posts/ASLR/2.png)  

Her defasında adreslerin değiştiğini görebilirsiniz. Bu durumda exploitimizi yazarken elde etmiş olduğumuz adres programı kapattığımızda değişeceği için pek bir işe yaramayacaktır.

`NOT :`

Sistemimizde (linux için) `ASLR` korumasının açık olup olmadığına bakmak için aşağıdaki komutu girebiliriz. Eğer 0 ise kapalı, 1 ise orta, 2 ise full+full açık demektir

    cat /proc/sys/kernel/randomize_va_space


`NOT-2 :`

ASLR sistemimizde aktif olmasına rağmen bazen uygulama bunu desteklemeybilir ve her seferinde bize aynı adresi verebilir. Bu durumda da gcc ile derlerken birkaç parametre kullanabiliriz.

![3](/static/img/posts/ASLR/3.png)  

![4](/static/img/posts/ASLR/4.png)  

Burada dikkat ederseniz adresin belli bir kısmı değişirken belirli bir kısmı sabit kalmakta. Bu da `ASLR Bypass` yaparken işimize yarayabilecek bir detay.

# `Bypass ? `

ASLR korumasını bypass etmek için çeşitli yöntemler mevcut.

- `ret2libc + Bruteforcing`

- `Return Oriented Programming (ROP)`

- `return to PLT (ret2PLT)`

Şimdilik sadece bunları gördüm ilerde başka yöntemler ile karşılaşırsam eklerim.

# `DEMO`

Sanırım az biraz `ASLR`'nin ne olduğunu anladık. Şimdi de kısa bir örnek yapalım ki kafamızda ki soru işaretlerini giderelim. (Bu yazıda yukarda bahsettiğim tekniklerin hepsini anlatmayacağım)

Kodumuz şu şekilde...

    #include <stdio.h>
    #include <string.h>

    void shell() {
    	system("/bin/sh");
    }

    void vuln(char *str) {
    	char buffer[64];
    	strcpy(buffer, str);
    }

    void main(int argc, char *argv[]) {
    	vuln(argv[1]);
    	printf("%s\n","Exec Normal");
    }

**

    gcc vuln.c -o vuln

Derlediğimize göre gdb üzerinden açıp incelemeye başlayalım.

![5](/static/img/posts/ASLR/5.png)  

Buraya kadar bildiğimiz şeyler. gdb ile açtık ve breakpointimizi koyduk. Şimdi de programımız kaç karakterden sonra dönüş adresi yazacak onu bulalım.

![6](/static/img/posts/ASLR/6.png)  

`EEEE` karakterinde geri dönüyor. Demekki 76 karakterden sonra `eip` üzerine bir adres yazabilirim.

Dikkat ederseniz programımızın içerisinde kullanılmamış ve `/bin/sh` komutunu çalıştıran bir fonksiyon var. Bunun adresini bulup `eip` adresinin yerine yazarsam shell alabiliriz sanırım.

![7](/static/img/posts/ASLR/7.png)  

Adresi bulduk... Şimdi yerine koyalım

![8](/static/img/posts/ASLR/8.png)  

Evet gdb üzerinden başarılı bir şekilde shellimizi aldık. Ancak burda adresler değişken değildi yani eğer biz gdb üzerinden değilde direk terminal üzerinden exploitimizi çalıştırırsak bu çalışmayacaktır. (ASLR koruması açıkken)

![9](/static/img/posts/ASLR/9.png)  

ASLR koruması açıkken `Segmentation Fault` hatası aldık. İsterseniz bir de bu koruma kapalıyken çalışıp çalışmadığına bir göz atalım. (maksat içimiz rahat etsin)

![10](/static/img/posts/ASLR/10.png)  

Evet exploitimiz `ASLR` koruması kapalıyken cayır cayır çalışmakta ancak koruma açıldığı zaman o kadar da etkili değil. Bunun için ufak bir bash script yazacağız. Kısaca yapacağımız şey ise bir döngü oluşturup sürekli olarak exploitimizi çalıştıracağız ve şans eseri doğru adresin denk gelmesini bekleyeceğiz.

    while true
    do
    ./vuln $(python -c 'print "A"*+"BBBB"+"CCCC"+"DDDD"+"\Xb9\x11\x40\x00"')
    done

**

![11](/static/img/posts/ASLR/11.png)  

Ve gördüğünüz gibi `ASLR` koruması açıkken de exploitimizi çalıştırmış olduk.

### `Son Not`

Şans ağırlıklı bir çözüm oldu ancak konunun anlaşılması için diğer yöntemlere göre daha basit ve daha sade bir çözüm olduğunu düşündüğüm için bu yolu tercih ettim. İlerleyen zamanlarda (vakit buldukça) diğer yöntemler ile alakalı da birşeyler yazmaya çalışacağım :)
