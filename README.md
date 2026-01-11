# Deney Sonu Teslimatı

Sistem Programlama ve Veri Yapıları bakış açısıyla veri tabanlarındaki performansı öne çıkaran hususlar nelerdir?

Aşağıda kutucuk (checkbox) ile gösterilen maddelerden en az birini seçtiğiniz açık kaynak kodlu bir VT kaynak kodları üzerinde göstererek açıklayınız. Açıklama bölümüne kısaca metninizi yazıp, kod üzerinde gösterim videonuzun linkini en altta belirtilen kutucuğa yerleştiriniz.

- [X]  Seçtiğiniz konu/konuları bu şekilde işaretleyiniz. **!**
    
---

# 1. Sistem Perspektifi (Operating System, Disk, Input/Output)

### Disk Erişimi

- [ ]  **Blok bazlı disk erişimi** → block_id + offset
- [ ]  Rastgele erişim

### VT için Page (Sayfa) Anlamı

- [ ]  VT hangisini kullanır? **Satır/ Sayfa** okuması

---

### Buffer Pool

- [X]  Veritabanları, Sık kullanılan sayfaları bellekte (RAM) kopyalar mı (caching) ?

- [X]  LRU / CLOCK gibi algoritmaları
- [ ]  Diske yapılan I/O nasıl minimize ederler?

# 2. Veri Yapıları Perspektifi

- [ ]  B+ Tree Veri Yapıları VT' lerde nasıl kullanılır?
- [ ]  VT' lerde hangi veri yapıları hangi amaçlarla kullanılır?
- [ ]  Clustered vs Non-Clustered Index Kavramı
- [ ]  InnoDB satırı diskte nasıl durur?
- [ ]  LSM-tree (LevelDB, RocksDB) farkı
- [ ]  PostgreSQL heap + index ayrımı

DB diske yazarken:

- [ ]  WAL (Write Ahead Log) İlkesi
- [ ]  Log disk (fsync vs write) sistem çağrıları farkı

---

# Özet Tablo

| Kavram      | Bellek          | Disk / DB      |
| ----------- | --------------- | -------------- |
| Adresleme   | Pointer         | Page + Offset  |
| Hız         | O(1)            | Page IO        |
| PK          | Yok             | Index anahtarı |
| Veri yapısı | Array / Pointer | B+Tree         |
| Cache       | CPU cache       | Buffer Pool    |

---

# Video [Linki](https://youtu.be/x1EhczEopQE) 
(23060173, MERT, TERECİ, Makine Öğrenmesi, CNN tabanlı görüntü işleme). 

---

# Açıklama (Ort. 600 kelime)

# Buffer Pool ve LRU Algoritması: MySQL/InnoDB Kaynak Kod Analizi

## Giriş

Veritabanı sistemlerinin performansı büyük ölçüde disk ile bellek arasındaki veri transferinin ne kadar verimli yönetildiğine bağlıdır. Disk erişimi, RAM erişimine kıyasla yaklaşık 100.000 kat daha yavaştır. Bu hız farkı, modern veritabanı yönetim sistemlerinin en kritik optimizasyon noktalarından birini oluşturur. MySQL'in InnoDB depolama motoru, bu sorunu Buffer Pool adı verilen bir bellek önbellekleme mekanizması ile çözer.

## Buffer Pool Nedir?

Buffer Pool, InnoDB'nin RAM'de ayırdığı ve sık erişilen veritabanı sayfalarını önbelleğe aldığı bir alandır. Veritabanı sunucularında genellikle fiziksel belleğin %80'ine kadarı bu havuza ayrılır. Her sayfa tipik olarak 16KB boyutundadır ve birden fazla satır veya indeks girdisi içerebilir.

MySQL kaynak kodunda Buffer Pool yapısı `storage/innobase/include/buf0buf.h` dosyasında tanımlanan `buf_pool_t` struct'ı ile temsil edilir. Bu yapı içerisinde `free` listesi boş sayfaları, `LRU` listesi ise kullanımda olan sayfaları tutar. Ayrıca `page_hash` hash tablosu sayesinde istenen bir sayfa hızlıca bulunabilir.

Bir SQL sorgusu çalıştırıldığında, InnoDB öncelikle ihtiyaç duyulan sayfanın Buffer Pool'da olup olmadığını kontrol eder. Sayfa bellekte mevcutsa doğrudan oradan okunur ve diskten okuma yapılmasına gerek kalmaz. Bu durum "buffer pool hit" olarak adlandırılır ve veritabanı performansını dramatik şekilde artırır.

## LRU Algoritması

Buffer Pool'un kapasitesi sınırlıdır. Yeni sayfalar için yer açılması gerektiğinde hangi sayfaların çıkarılacağına karar vermek kritik bir problemdir. InnoDB bu sorunu LRU (Least Recently Used - En Az Kullanılan) algoritmasının geliştirilmiş bir versiyonuyla çözer.

Klasik LRU algoritması, en uzun süredir erişilmemiş sayfayı çıkarmayı önerir. Ancak InnoDB, tam tablo taraması gibi işlemlerin tüm önbelleği kirletmesini önlemek için "midpoint insertion" stratejisi kullanır. Bu stratejide LRU listesi iki alt listeye bölünür: listenin başında sık erişilen "young" (genç/sıcak) sayfalar, sonunda ise daha az erişilen "old" (eski/soğuk) sayfalar bulunur.

Kaynak kodda bu yapı `buf_pool_t` struct'ındaki `LRU` listesi ve `LRU_old` pointer'ı ile yönetilir. `LRU_old_ratio` değişkeni, listenin ne kadarının eski sayfalar için ayrılacağını belirler ve varsayılan olarak listenin yaklaşık %37'si eski sayfalara ayrılır.

Yeni bir sayfa Buffer Pool'a alındığında, doğrudan listenin başına değil, orta noktaya (midpoint) yani eski alt listenin başına eklenir. Sayfa tekrar erişildiğinde ve belirli bir süre geçtiyse, young alt listesine terfi ettirilir. Bu mekanizma, tek seferlik okunan sayfaların değerli önbellek alanını işgal etmesini engeller.

## Performans Etkisi

Buffer Pool ve LRU algoritmasının birlikte çalışması, veritabanı performansını birkaç şekilde optimize eder. İlk olarak, sık erişilen veriler bellekte tutularak disk I/O operasyonları minimize edilir. İkinci olarak, akıllı sayfa değiştirme politikası sayesinde gerçekten önemli veriler önbellekte kalırken, nadiren kullanılan veriler çıkarılır. Üçüncü olarak, midpoint insertion stratejisi büyük tarama operasyonlarının çalışma kümesini bozmasını önler.

## Veri Yapıları Perspektifi

Kaynak kod incelendiğinde, Buffer Pool'un temel veri yapılarının ustaca kullanıldığı görülür. LRU listesi çift yönlü bağlı liste (doubly linked list) olarak implemente edilmiştir. Bu yapı, sayfaların liste içinde O(1) zamanda taşınmasına olanak tanır. Bir sayfaya erişildiğinde, o sayfa mevcut konumundan çıkarılıp listenin başına taşınır - bu işlem pointer manipülasyonu ile sabit zamanda gerçekleştirilir.

Hash tablosu (`page_hash`) ise sayfaların hızlı bulunmasını sağlar. Bir sayfa talep edildiğinde, (space_id, page_number) çifti hash fonksiyonundan geçirilir ve sayfanın bellekte olup olmadığı O(1) ortalama zamanda belirlenir. Bu kombinasyon - hash tablosu ile hızlı arama, bağlı liste ile hızlı sıralama - önbellekleme sistemlerinde yaygın kullanılan klasik bir tasarım kalıbıdır.

## Sistem Programlama Perspektifi

Buffer Pool yönetimi, işletim sistemi seviyesinde birçok kavramla doğrudan ilişkilidir. Mutex'ler (`LRU_list_mutex`, `free_list_mutex`) eşzamanlı erişimi kontrol eder ve veri tutarlılığını sağlar. Birden fazla thread aynı anda Buffer Pool'a erişebilir, ancak kritik bölümler kilitlerle korunur.

Ayrıca Buffer Pool, işletim sisteminin sayfa önbelleğine (page cache) benzer bir mantıkla çalışır. Her ikisi de sık erişilen verileri hızlı belleğe alır ve benzer değiştirme politikaları kullanır. Ancak InnoDB kendi Buffer Pool'unu yöneterek, veritabanı iş yüklerine özel optimizasyonlar yapabilmektedir.

## Sonuç

Buffer Pool ve LRU algoritması, MySQL/InnoDB'nin yüksek performanslı çalışmasının temel taşlarıdır. Bu mekanizmalar, sistem programlama ve veri yapıları perspektifinden incelendiğinde, teorik bilginin pratikte nasıl uygulandığının mükemmel bir örneğini sunar. Linked list veri yapısı, hash tabloları ve önbellekleme algoritmaları gibi temel kavramlar, gerçek dünya veritabanı sistemlerinde kritik performans kazanımları sağlamak için bir araya getirilmiştir.

## VT Üzerinde Gösterilen Kaynak Kodları

Buffer Pool Yapısı [Linki](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/include/buf0buf.h) \
Buffer Pool Implementasyonu: [Linki](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/buf/buf0buf.cc) \
LRU Algoritması [Linki](https://github.com/mysql/mysql-server/blob/8.0/storage/innobase/buf/buf0lru.cc) \
MySQL Dokümantasyonu: [Linki](https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html) \
... \
...
