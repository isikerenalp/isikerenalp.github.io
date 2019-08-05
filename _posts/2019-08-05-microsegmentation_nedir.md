---
layout: post
title:  "Microsegmentation Nedir ?"
author: "Alp Eren Işık"
date:   2019-08-05
categories: [network]
permalink: /:year/:month/:day/:title:output_ext
---

**Microsegmentation**, ağ güvenliği uzmanları tarafından, genel sistemin güvenli kalmasını kolaylaştırmak amacıyla bir ağı daha küçük parçalara bölmek için kullanılan bir işlemdir. Bu yöntem bulut sistemlerine veya veri merkezlerine uygulanabilir ve güvenlik uzmanlarının tüm sistemin parçalarını tek tek emniyete almasını sağlar. Bu aynı zamanda kötü niyetli kullanıcılar için azaltılmış bir saldırı yüzeyi sunduğundan, geleneksel çevre güvenliğine göre avantaj sağlar. **Microsegmentation** tipik olarak ağ bölümlerini bir güvenlik bölgesine bağlamak için yapılandırılmış bir dizi güvenlik duvarı arayüzü kullanır. Bu bölge, yalnızca izin verilen kullanıcıların, cihazların ve uygulamaların o bölgeye erişmesine izin veren kurallarla güvence altına alınmaktadır. **Microsegmentation** etkili çalışması için güvenlik bölgelerinin, kontrollerinin ve iş akışlarının sıkı bir şekilde tanımlanması önemlidir.

![schema](/static/img/posts/microsegmentation/microsegmentation.png)

# `Microsegmentation vs. VLANs, Firewalls, ACLs`

 Şirketler yıllarca ağ bölümlemesi için güvenlik duvarlarına , sanal yerel alan ağlarına (VLAN) ve erişim kontrol listelerine (ACL) güveniyorlardı. **Microsegmentation** daha fazla saldırı direnci için güvenlik politikalarını bireysel iş yüklerine uygular. VLAN'ların çok kaba bölümleme yapmanıza izin verdiği yerlerde, **microsegmentation**  daha ince bölümleme yapmanıza izin verir. Bu nedenle, her yerde trafiğin ayrıntılı bölümlerine inmeniz gerekir. Kısaca özel bir vlan, sistemlerin aynı alt ağdaki diğer sistemlere ulaşmasını engeller ama bir katman 3 (ağ katmanı) bağlantısı üzerinden ulaşabilecekleri diğer sistemlere bağlantılarını engellemez ancak **microsegmentation** hangi bağlantılara izin verildiğini ve hangilerinin engellenmesi gerektiğini kontrol etmek için genellikle politika temelli bir kontrol sağlar. Hassas veri ve uygulamaları korumak için genellikle veri merkezinde yapılır.

 ![schema](/static/img/posts/microsegmentation/microsegmentation-azure.jpg)

Doğu-batı trafik kontrolü için iç güvenlik duvarlarının yapılandırılması ve ardından bu konfigürasyonların zaman içinde sürdürülmesi konusundaki manuel çaba, kurumlar için çok karmaşık ve maliyetli olmuştur. Buna karşılık, SDN (Software-defined networking) ve SDDC (Software-defined data center) yetenekleri, ağın **microsegmentation** yoluyla talebe göre hizmet sağlama, parametreleri değiştirme esnekliği ve her sanal makine üzerinde güvenliği sağlama yeteneğini destekler.

Bir başka **microsegmentation** kavramı, sanallaştırılmış güvenliğin “zero-trust model (sıfır güven modeli)” olarak adlandırılan, sadece bir iş yükünde veya uygulamada yalnızca gerekli izinlerin ve bağlantıların etkinleştirimesini ve varsayılan olarak başka bir şeyin engellenmesini sağlayabilir. Mikro-segmentasyon kullanarak, yöneticiler bir iş yükü veya uygulamanın nerede kullanılacağı, hangi tür verilere erişileceği ve uygulamanın ne kadar önemli / hassas olması gerektiğine dayanan bir güvenlik politikası programlayabilir.

# `Kuzey-Güney ve Doğu-Batı Trafik Nedir ?`

Kuzey-Güney ve Doğu-Batı trafiği, veri merkezi bağlamında bir ağ trafiği akış şeklidir. Tarayıcı aracılığıyla bazı web uygulamalarına erişmeye çalıştığınızda, web uygulaması, bazı veri merkezlerinde bulunan bir uygulama sunucusunda dağıtılmaktadır. Çok katmanlı bir mimaride, tipik bir veri merkezi yalnızca uygulama sunucusunu değil, yük dengeleyici, veri tabanı gibi diğer sunucuları da yönlendiriciler ve anahtarlar gibi ağ bileşenleriyle birlikte içerir.

Örneğin Web uygulamasına eriştiğinizde ;

1. Adım: İstemci ve  yük dengeleyici arasında ağ akışı oluşur. Bu birinci tür ağ akışı yani müşteri ile sunucu arasındaki ağ akışına **Kuzey-Güney** trafik adı verilir. Kuzey-Güney trafiği bir sunucu-müşteri trafiğidir.

2. Adım: Bir veri merkezinde bulunan yük dengeleyici, uygulama sunucusu, veri tabanı vb. arasında bir ağ akışı oluşur. Bu ikinci tür ağ akışı yani bir veri merkezinde veya farklı veri merkezleri ile farklı sunucular arasında oluşan ağ akışı **Doğu-Batı** trafiği olarak adlandırılır. Doğu-Batı trafiği bir sunucu-sunucu trafiğidir .

![schema](/static/img/posts/microsegmentation/microsegmentation-architecture.jpg)
