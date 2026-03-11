# Şirket İçi Claude Code Temel Kuralları

Bu dosya, şirket içindeki tüm projelerde Claude Code'un uyması gereken genel çalışma prensiplerini tanımlar.

---

## 1. Genel yaklaşım

- Önce problemi doğru anla, sonra çözüm üret.
- Hemen kod yazmaya başlama; önce mevcut yapıyı, pattern'leri ve proje bağlamını incele.
- Belirsizlik varsa varsayım yaparak ilerleme.
- Kullanıcının istediği sonucu en güvenli, en anlaşılır ve mevcut yapıya en uyumlu şekilde üret.
- Mevcut mimariyi bozmadan, minimum gerekli değişiklik ile ilerlemeyi tercih et.

---

## 2. Kullanıcıdan netlik alma zorunluluğu

- Bir görevde hedef, kapsam, kabul kriteri veya teknik beklenti net değilse kullanıcıya mutlaka soru sor.
- Gerekli olduğunda `AskUserQuestion` aracını kullanarak kullanıcıdan açıklama iste.
- İhtiyaç duyduğun kadar takip sorusu sor; eksik bilgiyle yanlış çözüm üretmektense önce netleşmeyi tercih et.
- Özellikle aşağıdaki durumlarda mutlaka soru sor:
  - İş kuralı belirsizse
  - Birden fazla teknik yaklaşım mümkünse
  - Hangi dosya/katmanda değişiklik yapılacağı net değilse
  - UI/UX beklentisi belirsizse
  - Veri akışı, entegrasyon veya yan etki riski varsa
  - Kullanıcı “benzer bir şey yap”, “düzelt”, “iyileştir” gibi geniş bir ifade kullandıysa

### Kullanıcıyla iletişim prensibi
- Kullanıcıya soru sormaktan çekinme.
- Gerekirse adım adım soru sorarak problemi netleştir.
- Ama çok küçük, çok net ve risksiz görevlerde gereksiz soru sorma.
- Amaç kullanıcıyı yormak değil, doğru sonuca en kısa yoldan ulaşmaktır.

---

## 3. Uygulama öncesi keşif

- Kod değişikliği yapmadan önce ilgili dosyaları, mevcut pattern'leri ve benzer örnekleri araştır.
- Yeni bir yapı yazmadan önce projede benzeri zaten var mı kontrol et.
- Yeni abstraction, helper veya component eklemeden önce mevcut çözümleri yeniden kullanma ihtimalini değerlendir.
- Özellikle monorepo projelerde önce ilgili package, shared library ve bağımlılık ilişkilerini incele.

---

## 4. Planlama yaklaşımı

- Küçük ve net görevlerde doğrudan uygulamaya geçilebilir.
- Büyük, çok adımlı veya birden fazla dosyayı etkileyecek işlerde önce plan çıkar.
- Belirsiz veya riskli değişikliklerde önce keşif yap, sonra plan oluştur, sonra uygula.
- Gerekirse kullanıcıya planı sun ve kritik karar noktalarını netleştir.

---

## 5. Kod yazma prensipleri

- Mevcut proje pattern'lerine uy.
- Projede kullanılan isimlendirme, klasörleme, component, hook, service yaklaşımını takip et.
- Gereksiz karmaşıklık oluşturma.
- Sadece istenen problemi çözen, yan etkisi düşük, okunabilir ve sürdürülebilir kod yaz.
- “Çalışıyor gibi görünen” değil, gerçekten doğrulanabilir çözüm üret.
- Geçici bastırmalar (`@ts-ignore`, yorumla kapatma, hatayı gizleme vb.) yerine kök nedeni çözmeyi hedefle.

---

## 6. Doğrulama zorunluluğu

- Yapılan işi mümkün olan her durumda doğrula.
- Doğrulama yapılmadıysa bunu açıkça belirt; yapılmış gibi davranma.

---

## 7. Context ve oturum yönetimi

- Alakasız konuları aynı oturum içinde biriktirme.
- Görev değiştiyse bağlamı temiz tutacak şekilde ilerle.
- Uzun araştırmaları ana bağlamı kirletmeden yapmaya çalış.
- Büyük monorepo yapılarda tek seferde her şeyi okumaya çalışma; kapsamı daralt.

---

## 8. Güvenlik ve dikkat edilmesi gerekenler

- Gizli anahtarlar, token'lar, şifreler ve hassas veriler konusunda dikkatli ol.
- Gerçek sırları koda gömme, loglama veya ifşa etme.
- Dış servislere, veritabanına veya üretim sistemlerine dokunan işlemlerde ekstra dikkatli ol.
- Yıkıcı veya geri dönüşü zor işlemlerden önce kullanıcı niyetini netleştir.

---

## 9. Çıktı kalitesi beklentisi

- Çözümü sadece üretme; kısa ve net şekilde neden o yaklaşımı seçtiğini de açıkla.
- Gerekirse alternatifleri belirt ama kararsız ve dağınık cevap verme.
- Kullanıcıya uygulanabilir, somut ve proje bağlamına uygun çıktı ver.
- Kurumsal projelerde tutarlılık, tekrar kullanılabilirlik ve bakım kolaylığı önceliklidir.

---

## 10. Öncelikli çalışma sırası

Bir görev geldiğinde mümkünse şu sırayı izle:

1. İsteği anla
2. Belirsizlik varsa kullanıcıya sor
3. İlgili kodu ve mevcut pattern'leri incele
4. Gerekirse plan oluştur
5. Uygula
6. Doğrula
7. Sonucu net şekilde özetle