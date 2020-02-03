---
layout: post
title:  "Priviahub - Minister Writeup"
author: "Alp Eren Işık"
date:   2020-02-03
categories: [priviahub, writeup]
permalink: /:year/:month/:day/:title:output_ext
---

# `Priviahub - Minister Writeup`

[Priviahub](https://priviahub.com/)'da geçtiğimiz günlerde `Web` kategorisinde emekli olan `Minister` makinesinin çözümüne anlatacağım.

Öncelikle adettendir bir `nmap` taraması ile başlayalım.

![nmap](/static/img/posts/priviahub-minister/nmap.png)

`8000` numaralı portta `Webmin` çalışıyormuş. Tarayıcımızdan makinemizin `8000` portuna gidip bakalım.

![web](/static/img/posts/priviahub-minister/web.png)

Bizi bir `login` ekranı karşıladı. Burada `Webmin`'in 1.920 sürümü çalışmakta. Çok uzatmadan hemen `searchsploit` ile herhangi bir exploiti var mı araştıralım.

![serchsploit](/static/img/posts/priviahub-minister/1.png)

Evet gördüğünüz gibi 1.920 sürümünde `Unauthenticated Remote Code Execution` zafiyeti bulunmakta ve bu zafiyeti sömüren exploitimizde `exploitdb` dizinimizin altında bulunmakta.

İlgili exploit : https://www.exploit-db.com/exploits/47230

Exploitimizi de bulduğumuza göre `metasploit` üzerinde exploitimizi bulup gerekli parametreleri verelim.

![metasploit](/static/img/posts/priviahub-minister/2.png)

    RHOSTS : 192.168.253.4
    RPORT  : 8000
    LHOST  : your_ip

Sanırım artık herşey hazır. Şimdi exploitimizi çalıştırıp test edelim bakalım çalışacak mı.

![exploit](/static/img/posts/priviahub-minister/3.png)

Ve bingo ! Exploitimiz başarılı bir şekilde çalıştı ve sisteme root olarak erişim sağlayabildik. Bundan sonra aşağıda bulunan dizinlerde user ve root `flag` dosyalarını okuyabiliriz.

# `FLAGS`

`user-flag` : /home/webminister/Desktop/non-privflag.txt

`root-flag` : /root/privflag.txt
