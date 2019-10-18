---
layout: post
title:  "Linux System Call Hooking"
author: "Alp Eren Işık"
date:   2019-10-16
categories: [hooking]
permalink: /:year/:month/:day/:title:output_ext
---

# `System Call(Sistem Çağrısı) Nedir ?`

![system_call](/static/img/posts/linux-system-call-hooking/12.png)

Temel olarak anlatacak olursak bizim çalıştırılabilir programımız ya `Library Functions`'a gidip program içerisinde kullanılacak olan fonksiyonları alır yada direk `System Call`'lara başvurur. Bu sistem çağrıları ise önce `kernel`a daha sonra da `hardware`e bağlanır.

Sistem çağrıları sayesinde yazılımcı doğrudan donanıma müdahale etmez. Donanım üzerinde gerçekleştireceği işlemi sistem çağrılarını kullanarak gerçekleştirir. Bu sayede olası sistem hatalarından kaçınılmış olur.

Linux çekirdeğinde `X86` mimarisi için yaklaşık 380 civarında sistem çağrısı bulunur.

# `System Call Trace`

Çalışabilir programımızın hangi `sistem çağrıları`'nı kullandığını görmek ve incelemek için `strace` komutunu kullanabiliriz.

Bu yazımızda `LS` komutunu `hook` edeceğimiz için ilk önce bahsetmiş olduğumuz `strace` komutuyla `sistem çağrılarına` bakalım.

    strace ls

![system_call](/static/img/posts/linux-system-call-hooking/1.png)

Gördüğünüz gibi ekrana sığmayan uzun bir çıktımız oldu. Bu uzun çıktıyı incelemek yerine dilerseniz alternatif bir yolumuz daha var.

    strace ls 1>/dev/null 2>/tmp/ls.strace; cat /tmp/ls.strace | cut -d'(' -f1 | sort -u

Bu sayede sadece `sistem çağrıları`'nın isimlerini listelemiş olduk.

![system_call](/static/img/posts/linux-system-call-hooking/2.png)

Bu sistem çağrılarından dizin girişleri alan `getdents64` sistem çağrısı dikkatimizi çekiyor. Hadi hep beraber [man page](https://linux.die.net/man/2/getdents64)'ini inceleyelim...

    int getdents(unsigned int fd, struct linux_dirent *dirp,
         unsigned int count);

Gördüğünüz gibi 4 adet argüman alıyor. Ayrıca [man page](https://linux.die.net/man/2/getdents64) içerisinde `linux_dirent` yapısı da mevcut.

    struct linux_dirent {
      unsigned long  d_ino;     /* Inode number */
      unsigned long  d_off;     /* Offset to next linux_dirent */
      unsigned short d_reclen;  /* Length of this linux_dirent */
      char           d_name[];  /* Filename (null-terminated) */
                      /* length is actually (d_reclen - 2 -
                        offsetof(struct linux_dirent, d_name) */
      /*
      char           pad;       // Zero padding byte
      char           d_type;    // File type (only since Linux 2.6.4;
                                // offset is (d_reclen - 1))
      */

    }


Son olarak gene [man page](https://linux.die.net/man/2/getdents64) içerisinde veya aşağıdaki komut sayesinde bu `sistem çağrısının` `readdir.c` içerisinde kullanıldığını görebiliriz.

    grep -B 1 getdents64 /usr/include/asm-generic/unistd.h

![system_call](/static/img/posts/linux-system-call-hooking/3.png)

Eğer `readdir.c` dosyasında `getdents64` aratırsanız bir de `linux_dirent64` yapısını göreceksiniz. O da şu şekilde :

    /* SPDX-License-Identifier: GPL-2.0 */
    #ifndef _LINUX_DIRENT_H
    #define _LINUX_DIRENT_H

    struct linux_dirent64 {
      u64		d_ino;
      s64		d_off;
      unsigned short	d_reclen;
      unsigned char	d_type;
      char		d_name[0];
    };

    #endif


** linux_dirent64 yapısı `ls` komutu çalıştıktan sonra gelen listeyi döndürür.

**NOT** : Eğer `ls` içerisinden getdents64 sistem çağrısını incelemek istersek :

    strace ls 2>&1 | grep getdents64

    ls -la | wc -l

![system_call](/static/img/posts/linux-system-call-hooking/4.png)


# `System Call Table`

`System Call Table` yani `sys_call_table` kernelin içerdiği tüm sistem çağrılarını tutulduğu ve hafıza yani memory içerisinde nerede olduğunu gösteren tablodur.

`System Call Table`‘nin adresini linux içerisinde bulmamız gerek. Bu sayede `System Call` ile bizim yazdığımız Hook sistem çağrısını değiştirebilelim. Daha sonra `ls` komutu çalıştığında `getdents64` sistem çağrısı yerine bizim yazmış olduğumuz Hook sistem çağrısı çalışmış olacak.

`sys_call_table` adresini bulmak için :

    sudo grep sys_call_table /boot/System.map-`uname -r`

![system_call](/static/img/posts/linux-system-call-hooking/7.png)

Görmüş olduğunuz gibi `ffffffff81e001e0` adresi benim `sys_call_table` adresim. Her sistemde bu adres değiştiği için ilerde bu adresi yazdığımız kısıma sizin kendi `sys_call_table` adresinizi yazmanız gerekecek.

# `System Call Hooking`

Artık kolları sıvayıp `System Call Hooking` için [LKM](https://isikerenalp.github.io/2019/10/14/lkm.html)'mizi yazabiliriz.

Peki ne yapacağız ? Hemen anlatalım. Burada asıl amacımız günün sonunda LKM'mizi kernel'a yükledikten sonra `ls` komutu çalıştığı zaman belirtmiş olduğumuz dosya isminde bir dosya varsa bunun gösterilmemesini/listelenmemesini sağlamak. Sanırım artık hazırız :) Ufaktan başlayalım.

Öncelikle her zamanki gibi kütüphanelerimizi ekleyerek başlayalım :)

    #include <linux/module.h>
    #include <linux/init.h>
    #include <linux/kernel.h>
    #include <linux/moduleparam.h>
    #include <linux/unistd.h>
    #include <linux/semaphore.h>
    #include <linux/dirent.h>
    #include <asm/cacheflush.h>

Şimdi de geliştireceğimiz modül hakkında bazı bilgiler girelim.

    MODULE_AUTHOR("Alp Eren");
    MODULE_DESCRIPTION("Hide a file getdents syscalls");
    MODULE_LICENSE("GPL")

`sys_call_table`

    void **sys_call_table;

Şimdi de saklamak istediğimiz dosyayı belirtelim.

    #define FILE_NAME "secret.txt"

Orjinal getdents64 sistem çağrısı :

    asmlinkage int (*originial_getdents64) (unsigned int fd, struct linux_dirent64 *dirp, unsigned int count);

Şimdi de bu orjinal getdents64 sistem çağrısı yerine geçecek Hookumuzu yazalım. Burada asıl amacımız `linux_dirent64` üzerinde bir döngü oluşturup daha sonra her bir dosya ismini arayıp , belirttiğimiz gizlenilecek dosya ile aynı ismi taşıyan dosya varsa bunu ls komutunda göstermemesini sağlamak.

    asmlinkage int sys_getdents64_hook(unsigned int fd, struct linux_dirent64 *dirp, unsigned int count) {
    	int rtn;
    	struct linux_dirent64 *cur = dirp;
    	int i = 0;
    	rtn = original_getdents64(fd, dirp, count);
    	while (i < rtn) {
    		if (strncmp(cur->d_name, FILE_NAME, strlen(FILE_NAME)) == 0) {
    			int reclen = cur->d_reclen;
    			char *next_rec = (char *)cur + reclen;
    			int len = (int)dirp + rtn - (int)next_rec;
    			memmove(cur, next_rec, len);
    			rtn -= reclen;
    			continue;

    		}
    		i += cur->d_reclen;
    		cur = (struct linux_dirent*) ((char*)dirp + i);
    	}

    	return rtn;
    }

`sys_call_table`'ı yazılabilir yapmamız lazım ki asıl sistem çağrısı ile kendi oluşturduğumuz Hookumuzun sistem çağrılarını değiştirebilelim. Bunun içinde `sys_call_table` adresinin `table entry` ‘sini manual olarak writable yapmamız gerek.

O yüzden bu kısımda `lookup_address` fonksiyonu ile `page table`‘nin adresini bulup sonrasında içerisinde `sys_call_table`‘ye yazma yetkisi vereceğiz.

    int set_page_rw(unsigned long addr) {
    	unsigned int level;
    	pte_t *pte = lookup_address(addr, &level);
    	if(pte->pte &~ _PAGE_RW) pte->pte |= _PAGE_RW;
    	return 0;
    }

Eğer yüklediğiniz LKM'yi ilerde silmek isterseniz `sys_call_table`'ı tekrardan `Read-Only` yapmanız gerekecek ki bu sayede herşey eski haline dönebilsin.

    int set_page_ro(unsigned long addr) {
    	unsigned int level;
    	pte_t *pte = lookup_address(addr, &level);
    	pte->pte = pte->pte &~_PAGE_RW;
    	return 0;
    }

LKM'mizi oluşturup `insmod` ile kernel içerisine yüklediğimiz zaman çalışacak fonksiyonumuzu yazalım. Bu aşamada `0x`'den sonraki kısma sizin kendi `sys_call_table` adresinizi yazmanız gerekecek.

    static int __init getdents_hook_init(void) {
    	sys_call_table = (void*)0xffffffff81e001e0;
    	original_getdents64 = sys_call_table[__NR_getdents64];
    	set_page_rw(sys_call_table);
    	sys_call_table[__NR_getdents64] = sys_getdents_hook;
    		return 0;
    }

Peki LKM‘yi silmek istersek, oluşturduğumuz Hook fonksiyonunu `sys_call_table`‘den çıkartıp herşeyi tekrar eski haline çevirmesini ve aynı zamanda table entry‘ide tekrardan Read-Only olmasını söylüyoruz.

    static void __exit getdents_hook_exit(void) {
    	sys_call_table[__NR_getdents64] = original_getdents64;
    	set_page_ro(sys_call_table);
    		return 0;
    }

Son olarak tüm bu `__init` ve `__exit` fonksiyonlarını modül içine yükleyebiliriz.

    module_init(getdents_hook_init);
    module_exit(getdents_hook_exit);


Tüm bu aşamalardan sonra LKM'miz aşağıdaki gibi olacak...

![C](/static/img/posts/linux-system-call-hooking/8.png)

(Bu aşamadan sonra bazı teknik aksaklıklardan dolayı testi ubuntuda gerçekleştirdim.)

Evet kodumuzu hazırladık. Şimdi sıra `Makefile` dosyasını oluşturmaya geldi.

![Makefile](/static/img/posts/linux-system-call-hooking/9.png)

`Makefile` dosyamızı da oluşturduğumuza göre artık test aşamasına geçebiliriz. Öncelikli olarak `make` komutu ile tüm bu kodu derleyelim. Daha sonra eğer saklayacağımız dosya yoksa dosyamızı oluşturalım eğer dosya zaten varsa `insmod` komutu ile LKM'mizi kernel'a yükleyip test edebilirsiniz.

    sudo insmod system_call_hooking.ko

![system_call_hooking](/static/img/posts/linux-system-call-hooking/10.png)

Evet gördüğünüz üzere `secret.txt` dosyası LKM'mizi kernel'a yükledikten sonra listelenmedi. `rmmod` komutu ile istediğiniz zaman LKM'yi kernel içerisinden çıkartabilirsiniz.

    sudo rmmod system_call_hooking

![system_call_hooking](/static/img/posts/linux-system-call-hooking/11.png)

# `NOT`
Yetkilendirmeler için `arch/x86/include/asm/pgtable_types.h` dosyasına bakabilirsiniz.

![system_call_hooking](/static/img/posts/linux-system-call-hooking/6.png)
