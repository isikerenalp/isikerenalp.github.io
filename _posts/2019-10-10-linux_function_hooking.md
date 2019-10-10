---
layout: post
title:  "Linux Function Hooking"
author: "Alp Eren Işık"
date:   2019-10-10
categories: [hooking]
permalink: /:year/:month/:day/:title:output_ext
---

`Function Hooking`, çağrıların önceden var olan işlevlerine müdahale etmek ve işlevlerin çalışma zamanında davranışını değiştirmek için kullanılan bir dizi tekniktir. Çalışma zamanında sistemdeki paylaşılan kitaplıklardan gelen çağrıları dinamik olarak yüklememize ve yürütmemize izin veren dinamik yükleyici API'sini kullanarak linux taki işlevsellik üzerine odaklanacağız ve `LD_PRELOAD` ortam değişkenini kullanacağız.

`LD_PRELOAD` ortam değişkeni, ilk olarak yükleyici tarafından yüklenecek olan bir kütüphaneyi belirtmek için kullanılır. Önce kendi kütüphanemizi yüklemek, işlev çağrılarına müdahale etmemizi ve dinamik yükleyici API'sini kullanarak başlangıçta amaçlanan işlevi bir işlev işaretçisine bağlayıp, orijinal argümanları bunun içinden geçirerek işlev çağrısını etkin bir şekilde sarabilmemizi sağlar.

Basit bir örnek ile bu işlemin nasıl yapıldığına bakalım.

Ekrana `Hello World !` yazan bir `C` kodumuz olsun.

![hooking](/static/img/posts/linux-function-hooking/1.png)

Burda `puts` fonksiyonunu çalışma zamanında(load-time) kesip oluşturmuş olduğumuz diğer `puts` fonksiyonu ile çıktıyı değiştireceğiz.

`Load-Time` : Bir yazılım çalışmaya başlamadan önce bir yüklenme zamanı vardır. Bu yükleme zamanında program tam olarak çalışabilir hale gelene kadar gerekli olan gereksinimlerini kendine yükler.

Fonksiyonu `hook` edeceğimiz kütüphanemiz aşağıdaki gibi olacak.

![hooking](/static/img/posts/linux-function-hooking/3.png)

Peki bu ne yapıyor ona bakalım.

Gerekli kütüphanelerimizi ekledikten sonra hook edeceğimiz fonksiyonu yazalım `int puts(const char *message)`

Daha sonra orjinal fonksiyonumuzun yerine geçecek olan fonksiyonumuzu yazdık. `int (*new_puts)(const char *message)`

Sıra en başta eklemiş olduğumuz `dlfcn.h` kütüphanesinden gelen ve `Function Hooking` için önemli olan `dlsym` fonksiyonumuzu kullanmaya geldi. `new_puts = dlsym(RTLD_NEXT, "puts")` Aldığımız ilk argüman olan `RTLD_NEXT` dinamik yükleyici API'sinin ikinci argüman ile ilişkilendirilen işlevin bir sonraki örneğe dönmesini istediğimizi söylüyor. İkinci argümanımız ise dönülecek örneğin ismini istiyor yani müdahale de bulunacağımız `puts` fonksiyonunu.

Artık kütüphanemizi derleyip `LD_PRELOAD` ortam değişkenine atayabiliriz.  

![hooking](/static/img/posts/linux-function-hooking/4.png)

Derlemek için `gcc libexample.c -o libexample.so -fPIC -shared -ldl -D_GNU_SOURCE`

Derledikten sonra `libexample.so` adıyla kütüphanemiz bulunduğumuz dizine ekleniyor.

Şimdi export ile LD_PRELOAD ortam değişkenine atalım `export LD_PRELOAD="kütüphanenizin_bulunduğu_dizin/kütüphane_adı"`

LD_PRELOAD ortam değişkeni yazılım çalışmaya başladığında bizim kütüphanemizi içine alarak yazılımı manipüle etmemize imkan sağlıyor.

![hooking](/static/img/posts/linux-function-hooking/6.png)

# Kaynakça

    https://blog.netspi.com/function-hooking-part-i-hooking-shared-library-function-calls-in-linux/

    https://vvhack.org/t/linux-load-time-function-hooking/161
