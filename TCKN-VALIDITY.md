# data-quality-management-with-SQL

**SQL-based data quality validations designed to support data governance and compliance initiatives.**

---

## Oracle SQL - TCKN (Türkiye Cumhuriyeti Kimlik Numarası) Validasyonu - Veri Kalitesi Kontrolü

Bu sorgu, Türkiye Cumhuriyeti vatandaşlarına ait kimlik numaralarının (TCKN) doğruluğunu Oracle SQL üzerinde kontrol etmek için geliştirilmiştir.

Sorgu; TCKN'nin yapısal doğruluğunu, belirli algoritmik kontrolleri ve temel geçerlilik kurallarını uygulayarak her kayıt için `IS_VALID` kolonunda geçerlilik bilgisi üretir.

---

## Kurallar

- TCKN boş olmamalıdır.
- TCKN 11 haneli olmalıdır.
- İlk hane 0 olamaz.
- Son hane çift sayı (0, 2, 4, 6, 8) olmalıdır.
- Onuncu hane doğrulaması algoritmaya göre yapılır:  
  1., 3., 5., 7. ve 9. hanelerin toplamının 7 ile çarpımından 2., 4., 6. ve 8. haneler çıkartılır.  
  Sonucun 10'a bölümünden kalan sayı 10. haneye eşit olmalıdır.
- On birinci hane doğrulaması algoritmaya göre yapılır:  
  İlk 10 hanenin toplamının 10'a bölümünden kalan, 11. haneye eşit olmalıdır.

### Opsiyonel Filtreler

- Müşteri tipi 'Bireysel' olmalıdır. (`CUSTOMER_TYPE = 'Bireysel'`)
- Türkiye Cumhuriyeti vatandaşı olmalıdır. (`NATIONALITY = 90`)

---

## Kullanım

Bu sorguyu doğrudan veri tabanınıza çalıştırarak geçerlilik kontrolü yapabilirsiniz.  
`your_table_name` ifadesini kendi tablolarınızla değiştirmeniz ve kolon isimlerini revize etmeniz yeterlidir.

---

## Açıklamalar

- `IS_VALID = 1` ➔ TCKN geçerli.
- `IS_VALID = 0` ➔ TCKN geçersiz.

---

## Neden Bu Sorguya İhtiyaç Duyulur?

TCKN, Türkiye'de bireysel müşteri kimlik doğrulaması için kritik öneme sahiptir.

Yanlış veya hatalı kaydedilmiş TCKN'ler:

- İş süreçlerinde aksamalara,
- Yasal uyumluluk risklerine

sebep olabilir.

Bu sorgu, veri kalitesi yönetimi süreçlerinde güvenilir bir kontrol mekanizması sağlar.

> **Not:**  
> Sorgu yalnızca yapısal ve algoritmik doğrulama yapar. Kimlik numarasının resmi geçerliliğini (örneğin Nüfus Müdürlüğü verisiyle karşılaştırmayı) sağlamaz.

---

## SQL Sorgusu

```sql
WITH parsed_tckn AS (
    SELECT
        TCKN,
        NATIONALITY,
        CUSTOMER_TYPE,
        TO_NUMBER(SUBSTR(TCKN,1,1)) AS d1,
        TO_NUMBER(SUBSTR(TCKN,2,1)) AS d2,
        TO_NUMBER(SUBSTR(TCKN,3,1)) AS d3,
        TO_NUMBER(SUBSTR(TCKN,4,1)) AS d4,
        TO_NUMBER(SUBSTR(TCKN,5,1)) AS d5,
        TO_NUMBER(SUBSTR(TCKN,6,1)) AS d6,
        TO_NUMBER(SUBSTR(TCKN,7,1)) AS d7,
        TO_NUMBER(SUBSTR(TCKN,8,1)) AS d8,
        TO_NUMBER(SUBSTR(TCKN,9,1)) AS d9,
        TO_NUMBER(SUBSTR(TCKN,10,1)) AS d10,
        TO_NUMBER(SUBSTR(TCKN,11,1)) AS d11
    FROM your_schema_name.your_table_name
    WHERE 
        TCKN IS NOT NULL -- Dolu olan TCKN verisinin geçerliliği kontrol edilmelidir
--     AND CUSTOMER_TYPE = 'Bireysel' -- Sadece bireysel müşterilerin verisi kontrol edilmelidir
--     AND NATIONALITY = 90 -- Sadece Türk vatandaşlarının verisi kontrol edilmelidir
)
SELECT
    TCKN,
    NATIONALITY,
    CUSTOMER_TYPE,
    CASE 
        WHEN LENGTH(TRIM(TCKN)) <> 11 THEN 0 -- 11 haneli olmalıdır
        WHEN d1 = 0 THEN 0 -- İlk basamak sıfır olamaz
        WHEN d11 NOT IN (0, 2, 4, 6, 8) THEN 0 -- Son basamak çift sayı olmalıdır
        WHEN MOD(((d1 + d3 + d5 + d7 + d9) * 7 - (d2 + d4 + d6 + d8) + 30), 10) <> d10 THEN 0 -- 10. hane doğrulaması
        WHEN MOD((d1 + d2 + d3 + d4 + d5 + d6 + d7 + d8 + d9 + d10), 10) <> d11 THEN 0 -- 11. hane doğrulaması
        ELSE 1
    END AS IS_VALID
FROM parsed_tckn;
