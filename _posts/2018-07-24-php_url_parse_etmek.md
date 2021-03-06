---
layout: post
title:  "PHP URL Ayrıştırmak (URL Parse Etmek)"
author: "Alp Eren Işık"
date:   2018-07-24
categories: [PHP]
permalink: /:year/:month/:day/:title:output_ext
---

***


# parse_url()
  parse_url fonksiyonu PHP4 ve üstü sürümlerde desteklenmektedir. Bu fonksiyon ile istediğimiz URL'i parçalara ayırmamız mümkün. Ayrıca bu fonksiyon bir URL doğrulayıcı olarak kullanılmamalıdır. Yaptığı sadece URL'i birazdan belirteceğim parçalara ayırmaktır.

# Bileşenler
  Belirli bir URL bileşenini dizge olarak döndürmek isterseniz şu sabitlerden birini kullanabilirsiniz. => `PHP_URL_SCHEME` , `PHP_URL_HOST` , `PHP_URL_PORT` , `PHP_URL_USER` , `PHP_URL_PASS` , `PHP_URL_PATH` , `PHP_URL_QUERY` , `PHP_URL_FRAGMENT` .

# Dönen Değerler
  NOT: Bozulmuş URL'lerde parse_url() `False` döndürebilir ve `E_WARNING` çıktısı verebilir.

  `scheme` = http://

  `host` = konak ismi

  `port` = port

  `user` = kullanıcı

  `pass` = parola

  `path` = dosya yolu

  `query` = sorgu / ?'den sonra

  `fragment` = örgü / #'den sonra

    Örnek 1-)
      <?php
        $url = 'http://username:password@hostname/path?arg=value#anchor';
        /* print_r() fonksiyonu bir dizinin içeriğini yazdırmak için kullanılır*/
          print_r(parse_url($url));
            echo parse_url($url, PHP_URL_PATH);
      ?>
      // Çıktısı
        Array
      (
        [scheme] => http
        [host] => hostname
        [user] => username
        [pass] => password
        [path] => /path
        [query] => arg=value
        [fragment] => anchor
      )
        /path

#

      Örnek 2-)
      <?php
        $url = 'https://www.youtube.com/watch?v=IQaXTd8Ljg4';
          $arr = parse_url($url);
            print_r($arr);
      ?>
      //Çıktı
        Array
      (
        [scheme] => https
        [host] => www.youtube.com
        [path] => /watch
        [query] => v=IQaXTd8Ljg4
      )


# parse_str()
  parse_str() fonksiyonu, bir sorgu dizesini değişkenlere ayrıştırır.

    Örnek 1-)
    <?php
      parse_str("name=AlpEren&age=20");
        echo $name."<br>";
          echo $age;
    ?>
    // Çıktı
    AlpEren
    20

#

    Örnek 2-)
    <?php
      parse_str("name=AlpEren&age=20",$myArray);
        print_r($myArray);
          print_r($myArray['name']);
            echo"\n";
              echo $myArray['age'];
    ?>
    // Çıktı
          Array
        (
            [name] => AlpEren
            [age] => 20
        )
        AlpEren
        20                                                                             




# parse_url && parse_str Örnekleri

    Örnek 1-)
    <?php  
      $url = 'https://www.youtube.com/results?search_query=PauSiber';    
        $parts = parse_url($url);  
          $query = array();  
            parse_str($parts['query'], $query);  
              echo "v= ". $query['search_query'];  
              ?>
    // (Sadece Query değerini almış olduk)
    // Çıktı
    v= PauSiber           

#

    Örnek 2-)
    <?php  
      $url = 'https://ders.im/?q=ders.im&hPP=6&idx=documents&p=0';    
        $parts = parse_url($url);  
          $query = array();  
            parse_str($parts['query'], $query);  
              echo "v= ". $query['q'];  
              ?>
      // Çıktı
            v= ders.im
