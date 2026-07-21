# C#/.NET Koleksiyon Türlerinin Karşılaştırmalı Analizi

**IEnumerable, IQueryable, ICollection, IList ve List**

*Berat Soylu — Bilgisayar Mühendisliği Bölümü, Trakya Üniversitesi, Edirne*

---

> **Özet.** C#/.NET platformunda bir koleksiyonu temsil eden türlerin çoğu ortak bir arayüz hiyerarşisinden türer ve aralarındaki ayrımlar yeterince anlaşılmadığında, işlevsel olarak çalışan ancak verimsiz yazılımlar ortaya çıkabilir. Bu çalışmada `IEnumerable<T>`, `IQueryable<T>`, `ICollection<T>`, `IList<T>` ve somut `List<T>` sınıfı; sundukları yetenekler, çalışma yerleri, değiştirilebilirlik ve erişim biçimleri açısından karşılaştırmalı olarak incelenmektedir. Özellikle Entity Framework Core bağlamında `IEnumerable` ile `IQueryable` ayrımının sorgu performansı üzerindeki etkisi ele alınmakta ve tür seçimine ilişkin uygulama önerileri sunulmaktadır.
>
> **Anahtar Kelimeler:** C#, .NET, koleksiyon türleri, LINQ, Entity Framework Core, IQueryable, IEnumerable, gecikmeli çalışma.

---

## 1. Giriş

Bir yöntem tasarlanırken geriye döndürülecek koleksiyonun türüne karar vermek, ilk bakışta önemsiz görünen ancak sonuçları bakımından belirleyici bir tercihtir. `List<T>`, `IEnumerable<T>`, `ICollection<T>` ve `IQueryable<T>` gibi türlerin tümü söz dizimsel olarak geçerlidir ve derlenir. Bununla birlikte, bir yazılımın *çalışması* ile *doğru tasarlanmış olması* arasında belirgin bir fark bulunmaktadır. Bu fark, özellikle nesne–ilişkisel eşleme (ORM) araçlarının, örneğin Entity Framework Core'un, kullanıldığı senaryolarda kendini göstermektedir; yanlış bir tür seçimi, veritabanına gereksiz yük bindiren bir sorgu ile yalnızca gerekli verinin çekildiği verimli bir sorgu arasındaki farkı doğurabilmektedir.

Bu çalışmada söz konusu türler tek tek ele alınmakta, aralarındaki kalıtım ilişkisi ortaya konmakta ve her bir türün hangi durumlarda tercih edilmesi gerektiği tartışılmaktadır.

## 2. Ortak Kalıtım Hiyerarşisi

İncelenen türlerin büyük bölümü birbirinden türeyen **arayüzlerdir (interface)**. Buna karşılık `List<T>`, bu arayüzleri uygulayan somut bir **sınıftır (class)**. Türler arasındaki kalıtım ilişkisi temel olarak aşağıdaki gibidir:

```
IEnumerable<T>          → Dolaşılabilir (foreach)
   ├── IQueryable<T>    → Sorgu veritabanına çevrilir
   └── ICollection<T>   → Sayılabilir, eleman eklenip çıkarılabilir
          └── IList<T>  → İndeksle erişilebilir
                 └── List<T>   → Somut sınıf; tüm işlemleri kapsar
```

Hiyerarşide alt seviyelere inildikçe türün sunduğu **yetenek artmakta**, buna karşılık **esneklik azalmaktadır**. `IEnumerable<T>` en az taahhütte bulunan tür iken (yalnızca dolaşma), `List<T>` tüm işlemleri kapsamaktadır. Uygun tür çoğunlukla, gerçekten ihtiyaç duyulan en düşük yeteneği sunan türdür.

## 3. IEnumerable\<T\>

Hiyerarşinin kökünde yer alan `IEnumerable<T>`, tek bir taahhütte bulunur: üzerinde `foreach` döngüsüyle ileri yönde dolaşılabilir. Eleman sayısını doğrudan bilmez, eleman eklenip çıkarılamaz ve indeksle erişime izin vermez.

```csharp
IEnumerable<int> sayilar = new List<int> { 1, 2, 3 };

foreach (var s in sayilar)
    Console.WriteLine(s);
```

En belirleyici özelliği **gecikmeli çalışmadır (deferred execution)**. Bir LINQ sorgusu tanımlandığında sorgu hemen yürütülmez; sonuç fiilen kullanılana kadar ertelenir.

```csharp
IEnumerable<int> query = sayilar.Where(n => n > 1); // henüz yürütülmedi
foreach (var n in query)                             // filtreleme burada yürütülür
    Console.WriteLine(n);
```

`IEnumerable<T>` üzerinde tanımlanan LINQ işlemleri bellekte yürütülür (*LINQ to Objects*). Bu tür, bir koleksiyonun yalnızca okunup dolaşılacağı durumlarda, özellikle de yöntem parametrelerinde tercih edilir.

## 4. IQueryable\<T\>

Dış görünümü itibarıyla `IEnumerable<T>`'ye benzeyen ve ondan türeyen `IQueryable<T>`, iç işleyişi bakımından ondan ayrılır. Temel fark, sorgunun yürütüldüğü yerdir: `IEnumerable<T>` sorguyu bellekte çalıştırırken, `IQueryable<T>` sorguyu bir **ifade ağacı (expression tree)** biçiminde tutar, bunu SQL gibi bir sorgu diline çevirir ve veri kaynağında yürütür.

```csharp
IQueryable<Kullanici> sorgu = _context.Kullanicilar
    .Where(k => k.Yas > 18);
// Üretilen SQL: SELECT * FROM Kullanicilar WHERE Yas > 18

var yetiskinler = sorgu.ToList(); // sorgu veritabanında burada yürütülür
```

`Where`, `OrderBy` ve `Select` gibi işlemler zincirlenebilir ve bunların tümü tek bir SQL sorgusunda birleşir. `IQueryable<T>` de gecikmeli çalışır; `ToList()`, `First()` veya `Count()` çağrılana kadar veri kaynağına erişilmez.

## 5. IEnumerable ve IQueryable Ayrımının Performansa Etkisi

Uygulamada en sık karşılaşılan performans sorunlarından biri, bu iki türün ayırt edilememesinden kaynaklanmaktadır. Aşağıdaki iki kod bloğu birbirine benzese de tamamen farklı biçimde yürütülür.

```csharp
// Verimsiz: tüm tablo önce belleğe alınır, filtreleme sonrasında C# tarafında yapılır
IEnumerable<Kullanici> hepsi = _context.Kullanicilar.AsEnumerable();
var yetiskinler = hepsi.Where(k => k.Yas > 18);
// SQL: SELECT * FROM Kullanicilar   (koşul içermez; tüm satırlar çekilir)
```

```csharp
// Verimli: filtreleme veritabanında yürütülür, yalnızca gerekli satırlar aktarılır
IQueryable<Kullanici> yetiskinler = _context.Kullanicilar.Where(k => k.Yas > 18);
// SQL: SELECT * FROM Kullanicilar WHERE Yas > 18
```

> **Tasarım kuralı.** Bir Entity Framework Core sorgusu `AsEnumerable()` ya da `ToList()` ile veya bir `IEnumerable<T>` değişkene atandığı andan itibaren, sonraki tüm LINQ işlemleri bellekte yürütülür. Bu nedenle filtreleme, sıralama ve izdüşüm (`Select`) işlemleri, sorgu henüz `IQueryable` aşamasındayken tanımlanmalı; materyalizasyon (`ToList` vb.) mümkün olan en son adıma bırakılmalıdır.

## 6. ICollection\<T\>

`ICollection<T>`, `IEnumerable<T>`'nin üzerine somut yetenekler ekleyerek koleksiyonun bellekte hazır bulunmasını ve içeriğinin değiştirilebilmesini sağlar. `Count` özelliği koleksiyonu dolaşmadan eleman sayısını verir; `Add`, `Remove` ve `Clear` yöntemleri ekleme, çıkarma ve temizleme işlemlerini gerçekleştirir; `Contains` ise belirli bir elemanın bulunup bulunmadığını denetler.

```csharp
ICollection<string> isimler = new List<string>();
isimler.Add("Ali");
isimler.Add("Veli");
isimler.Remove("Ali");
int adet = isimler.Count;                // dolaşmadan → 1
bool varMi = isimler.Contains("Veli");   // → true
```

`ICollection<T>` sıralamayı ve indeksli erişimi garanti etmez. Entity Framework Core'da, ilişkili kayıtları temsil eden gezinme özellikleri (navigation property) çoğunlukla `ICollection<T>` türünde tanımlanır; örneğin bir `Musteri` varlığının `ICollection<Siparis> Siparisler` özelliği bu kullanıma örnektir.

## 7. IList\<T\>

`IList<T>`, `ICollection<T>`'nin üzerine sıralama ve indeks kavramlarını ekler. Köşeli parantezle indeksli erişim sağlayan `this[int]`, belirli konuma ekleme yapan `Insert`, belirli konumdaki elemanı silen `RemoveAt` ve bir elemanın sırasını döndüren `IndexOf` başlıca üyeleridir.

```csharp
IList<string> liste = new List<string> { "Ali", "Veli", "Ayse" };

string ilk = liste[0];              // → "Ali"
liste.Insert(1, "Fatma");           // → [Ali, Fatma, Veli, Ayse]
liste.RemoveAt(0);                  // → [Fatma, Veli, Ayse]
int sira = liste.IndexOf("Veli");   // → 1
```

Bu tür, elemanların sırasının anlamlı olduğu ve konuma göre erişimin gerekli olduğu durumlarda tercih edilir.

## 8. List\<T\>

Önceki bölümlerde ele alınan türlerin tümü birer arayüz, yani birer sözleşmedir. `List<T>` ise bu sözleşmeleri uygulayan somut bir sınıf olup, `new` işleci ile doğrudan örneği oluşturulabilen tek türdür. İç yapısında dinamik boyutlu bir dizi tutar ve kapasitesi dolduğunda otomatik olarak büyür. Önceki türlerin tüm yeteneklerine ek olarak `Sort()`, `AddRange()` ve `Find()` gibi işlevler sunar.

```csharp
List<int> sayilar = new List<int> { 3, 1, 2 };
sayilar.Add(5);
sayilar.Sort();                          // → [1, 2, 3, 5]
sayilar.AddRange(new[] { 8, 13 });
int ikinci = sayilar[1];                 // → 2
```

Bir koleksiyonun yöntem içinde oluşturulup üzerinde çeşitli işlemlerin yapılacağı durumlarda `List<T>` uygun bir seçimdir. Ancak bu türün yöntem imzasında kullanılması çoğunlukla önerilmez (bkz. Bölüm 10).

## 9. İlgili Diğer Türler

- **`T[]` (dizi):** En yüksek erişim başarımını sunar, ancak boyutu sonradan değiştirilemez.
- **`IReadOnlyCollection<T>` / `IReadOnlyList<T>`:** Bir koleksiyonun sayılabilir/dolaşılabilir olduğunu ancak değiştirilemeyeceğini ifade eder; genel (public) API'lerde tercih edilir.
- **`HashSet<T>`:** Tekrarsız elemanlardan oluşan bir küme sağlar ve üyelik denetimini yüksek başarımla gerçekleştirir.
- **`Dictionary<TKey, TValue>`:** Anahtar–değer eşlemesi sunar; hızlı arama gerektiren durumlar için uygundur.

## 10. Tür Seçimine İlişkin Uygulama Önerileri

**Parametrelerde mümkün olan en genel tür istenmelidir.** Bir yöntem koleksiyonu yalnızca dolaşacaksa `IEnumerable<T>` parametresi tanımlamak yeterlidir; böylece çağıran kod `List`, dizi veya `HashSet` — hangisi olursa olsun uyumlu bir koleksiyon iletebilir.

```csharp
public decimal ToplamHesapla(IEnumerable<Urun> urunler) { ... }
```

**Dönüş türünde, çağıranın gereksinim duyacağı ancak fazlasını taahhüt etmeyen tür seçilmelidir:**

- Yalnızca okunup dolaşılacaksa → `IEnumerable<T>` veya `IReadOnlyList<T>`
- Ekleme/çıkarma yapılacaksa → `ICollection<T>` veya `IList<T>`
- Dışarıda daha da filtrelenebilecek bir EF Core sorgusu → `IQueryable<T>`

> **Not.** Bir yöntemin içinde `List<T>` kullanmak olağandır. Ancak sonucu dışarıya `List<T>` olarak döndürmek, çağıran kodun listeyi değiştirebilmesine olanak tanır ve dönüş türünü ileride değiştirmeyi güçleştirir. Bu durumda `IReadOnlyList<T>` döndürmek çoğunlukla daha güvenlidir.

**Tablo 1.** C#/.NET koleksiyon türlerinin karşılaştırmalı özeti.

| Tür | Temel İşlev | Çalışma Yeri | Değiştirilebilir | İndeks Erişimi |
|-----|-------------|--------------|:---:|:---:|
| `IEnumerable<T>` | Dolaşma (foreach) | Bellek | Hayır | Hayır |
| `IQueryable<T>` | Sorgu (SQL'e çevrilir) | Veritabanı | Hayır | Hayır |
| `ICollection<T>` | Sayma, ekleme, çıkarma | Bellek | Evet | Hayır |
| `IList<T>` | İndeksli erişim | Bellek | Evet | Evet |
| `List<T>` | Tüm işlemler (somut sınıf) | Bellek | Evet | Evet |

## 11. Sonuç

Tür seçiminde belirleyici ilke, ihtiyaç duyulan en düşük yeteneği sunan türün tercih edilmesidir: parametrelerde en genel tür istenmeli, dönüş türünde ise gereğinden fazla taahhütte bulunulmamalıdır. Entity Framework Core kullanılan uygulamalarda `IEnumerable` ile `IQueryable` arasındaki ayrım özellikle önemlidir; bu ayrım, sorgunun bellekte mi yoksa veritabanında mı yürütüleceğini ve dolayısıyla veri kaynağından ne kadar verinin çekileceğini belirler. Koleksiyon türlerinin bilinçli seçimi, yalnızca bir üslup tercihi değil, uygulamanın başarımını ve sürdürülebilirliğini etkileyen bir mühendislik kararıdır.

## Kaynaklar

1. Microsoft, "System.Collections.Generic Ad Alanı," *.NET API Belgeleri*, Microsoft Learn.
2. Microsoft, "Language Integrated Query (LINQ)," *C# Kılavuzu*, Microsoft Learn.
3. Microsoft, "Querying Data — Entity Framework Core," *EF Core Belgeleri*, Microsoft Learn.
4. Microsoft, "IQueryable\<T\> Interface," *.NET API Belgeleri*, Microsoft Learn.
