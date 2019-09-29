---
layout: post
title:  "HackTheBox - Swagshop Writeup"
author: "Alp Eren Işık"
date:   2019-09-29
categories: [hackthebox, writeup]
permalink: /:year/:month/:day/:title:output_ext
---

***
# `HackTheBox - Swagshop Writeup`
***

`10.10.10.140` adresinde Magento yazılımının olduğunu gördükten sonra searchsploit ile herhangi bir yayınlanmış exploiti var mı yok mu diye baktım. İlk dikkatimi çeken `exploits/xml/webapps/37977.py` exploiti oldu.

![magento-37977](/static/img/posts/htb-swagshop/1.png)

Gerekli bilgileri exploitimde girdikten sonra bir de proxy ekledim.

![magento-37977](/static/img/posts/htb-swagshop/2.png)

`Burp Suite` ile gerekli proxy ayarımı da yaptıktan sonra exploitimi çalıştırdım. Default olarak gelen `forme:forme` kullanıcı adı ve parolası ile admin kullanıcısını oluşturmuş olduk. Dilerseniz exploiti çalıştırmadan önce ilgili satırdan istediğiniz bir `username:password` verebilirsiniz.

![magento-37977](/static/img/posts/htb-swagshop/3.png)

`10.10.10.140/index.php/admin` URL'sine giderek admin bilgilerini giriyoruz. (`forme:forme`)

![magento-admin-login](/static/img/posts/htb-swagshop/4.png)


![magento-admin-panel](/static/img/posts/htb-swagshop/5.png)

Admin panelinde herhangi bir exploit olabileceğini düşünerek tekrardan searchsploit ile bulduğum exploitleri incelemeye başladım. `exploits/php/webapps/37811.py` exploitini denemeye karar verdim ve kolları sıvadım.

![magento-37811](/static/img/posts/htb-swagshop/6.png)

Exploitin içerisinde bizden `/app/etc/local.xml` de bulunan tarih değerini istiyor.

![magento-37811](/static/img/posts/htb-swagshop/7.png)

Exploitimizde bulunan ilgili bilgileri doldurup güncelledikten sonra tekrardan proxy mizi ayarladık. Ve exploitimizin son hali aşağıdaki gibi oldu.

![magento-37811](/static/img/posts/htb-swagshop/8.png)

Exploitimize gerekli parametrelerini verdikten sonra [reverse shell](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) komutumuzu da ekledik ve ilgili portu dinlemeye aldık.

![magento-37811](/static/img/posts/htb-swagshop/9.png)

Gördüğünüz gibi exploit sorunsuz bir şekilde çalıştı. Artık sunucuya bağlantı sağladığımıza göre sıra hak yükseltmede  :)

# `Privilege escalation`

![hak-yükseltme](/static/img/posts/htb-swagshop/10.png)

`/var/www/html/` dizininde bulunan dosyaları `vi` üzerinden `root` yetkisi ile açabileceğimizi farkettim. Daha sonra `:!/bin/bash` diyerek `root` komut satırına düşeceğiz. (Burada /var/www/html/ dizini altında yeni bir dosya oluşturmak yerine /var/www/html/index.php diyerek var olan dosyayı da açabilirsiniz)

![hak-yükseltme](/static/img/posts/htb-swagshop/11.png)

`:) ve root olduk`

![hak-yükseltme](/static/img/posts/htb-swagshop/root.png)

User & System Owns

![hak-yükseltme](/static/img/posts/htb-swagshop/flags.png)
