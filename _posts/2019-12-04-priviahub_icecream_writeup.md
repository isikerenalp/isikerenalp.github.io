---
layout: post
title:  "Priviahub - Icecream Writeup"
author: "Alp Eren Işık"
date:   2019-12-04
categories: [priviahub, writeup]
permalink: /:year/:month/:day/:title:output_ext
---

# `Priviahub - Icecream Writeup`

[Priviahub](https://priviahub.com/)'da geçtiğimiz günlerde `Web` kategorisinde emekli olan `Icecream` makinesinin çözümüne değineceğim.

Öncelikle adettendir bir `nmap` taraması ile başlayalım.

![nmap](/static/img/posts/priviahub-icecream/nmap.png)

Evet gördüğümüz üzere `80` portunda `Apache` çalışıyor. Makinemizde zaten `Web` kategorisinde yer aldığı için zaman kaybetmeden bakalım web tarafında ne varmış :)

![nmap](/static/img/posts/priviahub-icecream/1.png)

Anasayfa da pek bişi yok gibi. Biraz daha gezinelim bizleri neler bekliyor.

![nmap](/static/img/posts/priviahub-icecream/2.png)

Gördüğünüz üzere diğer sayfalarda gezinmeye başlayınca `?file=` parametresi ile dosyaları/sayfaları çağırdığını görüyoruz. Aklımıza ilk gelen şey tabiki de `LFI` var mı yok mu diye kontrol etmek oldu.

![nmap](/static/img/posts/priviahub-icecream/3.png)

Ve bingo ! Hatamız geldi. Artık yetkimiz olduğu sürece istediğimiz dosyayı çağırıp okuyabiliriz.

![nmap](/static/img/posts/priviahub-icecream/4.png)

# User Flag

`C:\Users\Icecream\Desktop\non-privflag.txt`

![nmap](/static/img/posts/priviahub-icecream/user.png)

# Root Flag

`C:\Users\Administrator\Desktop\privflag.txt`

![nmap](/static/img/posts/priviahub-icecream/root.png)
