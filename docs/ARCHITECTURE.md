# YeniZil-Pair — Mimari

Durum: **tasarım aşaması, kod yok**

Kablosuz kapı zili ve kapı açıcı. Seri üretilip birden fazla binaya kurulacak. Donanım, radyo, güvenlik katmanı ve olay döngüsü önceki sürümden (github.com/murat-kacar/YeniZil) alınacak. Bu belge eşleştirme mimarisinin kararlarını tutar. Kodlama kuralları proje kökündeki `CLAUDE.md` dosyasında.

---

## 1. Kararlar

### Karar 1 — Seri ürün, rol başına tek firmware

Ürün birden fazla binaya kurulacak, kurulumu yazılımcı değil kurulumcu yapacak. Bütün dış ünitelere aynı firmware, bütün iç ünitelere aynı firmware yükleniyor. Kaynak kodda binaya ya da daireye özgü hiçbir değer yok. Kurulum bilgisayarsız, sadece düğmelerle yapılıyor.

### Karar 2 — Ağ kimliği dış ünitenin MAC'inden, şifre cihazda üretilir

- **Ağ kimliği:** Dış ünitenin fabrika MAC adresinden üretiliyor. Her binanın ağı tekil oluyor, komşu binaların mesajları ayrılıyor.
- **Ağ şifresi:** Dış ünite ilk açılışta donanım RNG'siyle rastgele üretip flash'ta (NVS) saklıyor. Kaynak kodda ya da depoda yer almıyor.
- **Şifrenin iç ünitelere iletimi:** Sadece eşleştirmede, ECDH ile türetilen ortak anahtarla şifrelenmiş olarak (BLE Mesh provisioning ve Wi-Fi WPS'teki gibi). Şifre havada hiçbir zaman açık gitmiyor, MAC'ten türetilmiyor.
- **Kerckhoffs ilkesi:** Güvenlik sadece şifreye dayanıyor. Kaynak kodun gizli olması güvenlik önlemi sayılmıyor. Saldırgan kodu değil, satın aldığı ya da söktüğü gerçek bir cihazı kullanabilir.

### Karar 3 — Eşleştirme tezgâhta, iki buton birlikte basılı tutularak

Eşleştirme kurulumdan önce, bütün cihazlar yan yanayken yapılıyor. Sonra iç üniteler dairelerine taşınıyor.

1. Dış ünite elektriğe takılır (bkz. Karar 4).
2. N. dairenin iç ünitesi eşleştirilirken dış ünitedeki **N numaralı zil butonu** ile iç ünitenin **kapı açma butonu** birlikte basılı tutulur.
3. İki butonun basılı tutma süresi **14–17 sn** aralığındayken iki cihaz eşleşir: ECDH ile ortak anahtar türetilir, ağ şifresi şifrelenmiş olarak iç üniteye geçer, iç ünite N. daire olarak kaydedilir.
4. Daire numarası ayrıca girilmiyor, basılan zil butonundan geliyor.

Her cihaz basılı tutma süresini kendi basış anından sayıyor. İki butona Δ sn arayla basılırsa iki cihazın birlikte hazır olduğu süre 3 − |Δ| sn oluyor. Eşleşme yarım saniyeden kısa sürdüğü için 2,5 sn'ye kadar fark tolere ediliyor. Eşleşme olmazsa kurulumcu bırakıp tekrar dener.

### Karar 4 — Eşleştirme sadece dış ünite açıldıktan sonraki ilk 10 dakikada

Dış ünitenin zil butonları sokakta, herkes erişebilir. Eşleştirme sadece butonlarla başlayabilseydi, bir saldırgan satın aldığı ya da söktüğü bir iç üniteyi sokaktan herhangi bir daire olarak ağa katabilir ve kapıyı açabilirdi. Pencerenin dar olması ya da kodun gizli olması bunu engellemez.

**Karar:** Eşleştirme sadece dış ünite elektriğe takıldıktan sonraki **ilk 10 dakika** içinde kabul ediliyor (Zigbee "permit join" ile aynı mantık).
- Tezgâhta dış ünite takılır, daireler bu sürede eşleştirilir.
- Kurulumdan sonra dış ünite sürekli açık, pencere kapalı.
- Bir iç ünite değiştirilecekse kurulumcu dış ünitenin fişini çekip takar, 10 dakikalık pencere yeniden açılır. Dış ünitenin ESP'si bina içinde olduğu için bunu sadece içeri girebilen biri yapabilir.
- Kalan risk: bir elektrik kesintisinden sonraki 10 dakika. Saldırganın kesintiyi denk getirmesi gerekiyor.

### Karar 5 — Basış süreleri birbirinden ayrık

Eşleştirme basışı bırakıldığında daire artık ağda olduğu için, normal basış sayılsaydı zil çalar ve kapı açılırdı. Bu yüzden aralıklar ayrık:

| Basılı tutma | Anlamı |
|---|---|
| 50 ms – 10 sn | Normal basış: zil ya da kapı açma, bırakınca tetiklenir |
| 10 – 14 sn | Hiçbir şey olmaz (ayırıcı boşluk) |
| 14 – 17 sn | Eşleştirme. Bırakınca eylem tetiklenmez |
| 17 sn'den uzun | Takılı buton sayılır, hiçbir şey olmaz |

Önceki sürümdeki 30 sn'lik normal basış üst sınırı 10 sn'ye iniyor.

### Karar 6 — Eşleşme onayı

Eşleşme tamamlanınca iç ünitenin zili bir kez kısa çalıyor, bağlantı LED'i de dış ünitenin heartbeat'iyle saniyede bir yanıp sönmeye başlıyor. Kurulumcu iki işaretten eşleşmeyi anlıyor.

### Karar 7 — Ek düğme yok

Eşleştirme için ayrı bir servis düğmesi ya da LED eklenmiyor. Mevcut zil butonları, kapı açma butonu, zil ve bağlantı LED'i yetiyor.

### Karar 8 — Yakınlık ve çakışma önlemleri

Eşleştirme tezgâhta yapıldığı için:
- Eşleştirme mesajları en düşük gönderim gücüyle (2 dBm) gönderiliyor, menzil birkaç metreye iniyor.
- Dış ünite sadece çok güçlü sinyalle (yan yana) gelen eşleştirme isteğini kabul ediyor (yakınlık kontrolü, Google Fast Pair'deki gibi).
- Aynı daire için pencere içinde iki ayrı aday görülürse eşleşme iptal ediliyor (WPS "PBC session overlap" kuralı).

---

## 2. Önceki sürümden aynen gelenler

- Radyo sürekli dinliyor. Her çerçeve 20 ms arayla 3 kez gönderiliyor.
- Dış ünite saniyede bir heartbeat yayınlıyor. Her heartbeat iç ünitenin bağlantı LED'ini 10 ms yakıyor, iç ünite bağlantı durumu tutmuyor.
- AES-128-CCM, nonce = gönderen MAC + kalıcı sayaç, tekrar koruması ve kalıcı alıcı sayaçları.
- Donanım ve kablolama, her LED'e 240 Ω seri direnç.

---

## 3. Açık sorular

1. **Fabrika ayarına dönme:** Bir iç ünite nasıl sıfırlanır? Sıfırlanan ünitenin sayacı ile eski şifre birlikte kullanılırsa nonce tekrarlanır. Sıfırlama, şifre yenilenmeden mümkün olmamalı.
2. **Şifre yenileme:** Dış ünite değişince yeni ağ kimliği ve yeni şifre oluşuyor, bütün iç üniteler yeniden eşleşmeli. Bir iç ünite kaybolursa ya da çalınırsa şifre nasıl yenilenir?
3. **Daire sayısı:** Şu an 4 daire kodda sabit. Binaya göre değişen daire sayısı donanım çeşidiyle mi (4'lü, 8'li dış ünite), kurulumla mı belirlenecek?
4. **Flash şifreleme ve secure boot:** Söken biri flash'tan şifreyi okuyabilir. Seri üründe standart çözüm ESP32'nin flash şifrelemesi ve secure boot'u. Bunlar eFuse ve muhtemelen ESP-IDF üretim adımı gerektiriyor, "Arduino-first" kuralıyla çatışabilir.
5. **İleride (şimdilik kapsam dışı):** Sahada yazılım güncellemesi (OTA), radyo sertifikasyonu (CE/RED).
