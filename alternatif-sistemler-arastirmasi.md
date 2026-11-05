# Kampanya ve Bilgilendirme Metinleri için Otomatik İnceleme Sistemi

**Alternatif yaklaşımlar araştırması — backend odaklı**

- **Tarih:** 5 Kasım 2026
- **Kapsam:** Müşteriye gönderilen kampanya ve bilgilendirme metinlerinin imla, yazım, noktalama, anlaşılabilirlik, doğru bilgi ve mevzuat açısından otomatik incelenmesi
- **Not:** Bazı bilgiler üretici beyanına dayanıyor; bunlar metinde *(üretici beyanı)* olarak işaretlendi. Mevzuat bölümleri hukuki görüş değildir; güncel metin mevzuat ve hukuk ekibiyle teyit edilmelidir.

---

## 1. Yönetici özeti

1. **İnceleme işi tek bir problem değil, en az altı farklı problem.** Yazım/noktalama, anlatım, anlaşılabilirlik, doğru bilgi, mevzuat uyumu ve kurum dili ayrı ayrı ele alındığında her biri farklı bir teknikle en iyi sonucu veriyor.
2. **Yalnızca fine-tune edilmiş bir LLM, en kritik iki kontrolü tek başına çözmez:**
   - *Doğru bilgi:* Model, kampanyanın gerçek faiz oranını, tarihini veya bonus tutarını bilemez. Bu bilgi kampanya kayıt sisteminden gelmeli ve metinle deterministik olarak karşılaştırılmalı.
   - *Mevzuat:* Kurallar değiştikçe modeli yeniden eğitmek gerekir. Kuralları ayrı bir "kural kütüphanesinde" tutup modele inceleme anında vermek (RAG) daha sürdürülebilir.
3. **Globalde bu alandaki en somut çalışma (Capital One, 2025/2026)** bile küçük modelleri fine-tune ettiğinde kural ihlali tespitinde en iyi ~%67 doğruluğa ulaştı ve insan denetimini şart koştu. Ticari ürünler (Saifr, Persado, Acrolinx) ABD/İngiltere mevzuatına ve İngilizceye odaklı; Türkçe ve BDDK/Reklam Kurulu kurallarına uygun hazır bir ürün bulunamadı.
4. **Türkçe için açık kaynak yapı taşları mevcut:** GECTurk (Koç Üniversitesi; TDK kurallarına dayalı 25 hata türü), hunspell-tr, Zemberek, VNLP, okunabilirlik formülleri, Türkçe LLM'ler (Kumru, Trendyol-LLM, Cosmos) ve Türkçe değerlendirme setleri (Cetvel, TR-MTEB).
5. **Öneri:** Fine-tune ile başlamak yerine **katmanlı hibrit bir mimari** (Bölüm 5-F): deterministik kurallar + yapılandırılmış bilgi çıkarımı ve kayıtlarla karşılaştırma + kural kütüphanesiyle desteklenen (RAG) açık kaynak LLM + doğrulama katmanı + insan onayı. Sistem çalıştıkça onaycıların kabul/ret kararları etiketli veri olarak birikir; **fine-tune, bu veriyle ve yalnızca ölçümle zayıf kaldığı görülen kategoriler için ikinci aşamada yapılır.**

---

## 2. Problemin ayrıştırılması

| Kontrol boyutu | Örnek hata | Hatanın doğası | En uygun teknik |
|---|---|---|---|
| **Yazım (imla)** | "herkez", "TL'ye/TL ye", "de/da" ve "ki" bitişik/ayrı yazımı, büyük harf | Kurala bağlı, büyük ölçüde deterministik | Sözlük + morfolojik analiz + sıralı etiketleme modeli (GECTurk tipi) |
| **Noktalama** | Virgülle bağlanmış iki bağımsız cümle, eksik kesme işareti | Kurala bağlı, bağlam gerektirir | Kural + GEC modeli / LLM |
| **Dil bilgisi ve anlatım bozukluğu** | Özne-yüklem uyumsuzluğu, gereksiz sözcük, anlam belirsizliği | Bağlamsal | LLM |
| **Anlaşılabilirlik / sadelik** | Çok uzun cümle, jargon, edilgen yapı, belirsiz koşul | Kısmen ölçülebilir | Okunabilirlik metrikleri + LLM rubriği |
| **Doğru bilgi** | Metinde %1,99, kampanya kaydında %2,19; geçersiz tarih (31 Kasım) | Olgusal, **kaynak veriye bağlı** | Yapılandırılmış bilgi çıkarımı + kayıtlarla deterministik karşılaştırma |
| **Mevzuat / zorunlu ibareler** | Kredi reklamında yıllık maliyet oranının ana mesajda olmaması; koşulsuz "faizsiz" ifadesi | Kurala bağlı ama yorum gerektirir | Kural kütüphanesi + RAG + kural bazlı LLM kararı |
| **Kurum dili ve terminoloji** | Ürün adının yanlış yazımı, hitap biçimi, yasaklı ifade | Kurum içi standart | Terim sözlüğü + regex kuralları |
| **Kanal kısıtları** | SMS karakter sınırı, ticari iletide ret (iletişim izni geri alma) bilgisinin eksikliği | Deterministik | Kural motoru |

**Önemli ayrım:** Ticari elektronik ileti (kampanya) ile bilgilendirme mesajlarının yasal gereklilikleri farklıdır. Sistem, girişte mesaj türünü (ve kanalı, ürünü) bilmeli ki doğru kural setini uygulasın.

---

## 3. Globalde ve açık kaynakta yapılanlar

### 3.1 Finans sektörüne özel ticari çözümler

| Çözüm | Ne yapıyor | Bize dersi |
|---|---|---|
| **Saifr** (Fidelity Labs'ten ayrılan girişim, 2022) | Pazarlama içeriğini FINRA 2210, SEC 482 ve SEC Marketing Rule'a göre tarıyor; risk tespiti ile "önerilen dil" modellerini ayrı sunuyor. Fidelity'nin 20+ yıllık uyum verisiyle eğitildiğini belirtiyor *(üretici beyanı)*. | **Riski tespit etme** ve **düzeltme önerme** ayrı görevler olarak tasarlanmalı. Şirketin kendi ifadesiyle: Kurum, düzenleyiciye "LLM öyle dedi" diyemez; sorumluluk kurumdadır. |
| **Persado Marketing Compliance AI** (Mayıs 2025) | Bankanın müşteri iletişiminde uyumsuz ifadeyi buluyor, ilgili düzenlemeyi gösteriyor ve uyumlu alternatif metin öneriyor. Pazarlama ve uzmanlaşmış mevzuat ajanlarından oluşan bir "ajan komitesi" kullanıyor; hukuk incelemesini %90 azalttığını iddia ediyor *(üretici beyanı)*. | Bulgunun yanında **dayanak kuralın gösterilmesi** ve uzman ajanlara bölme fikri. |
| **Acrolinx** (Microsoft Marketplace'te "Markup AI" adıyla da listeleniyor) | Stil rehberini, terim listelerini ve kuralları dijitalleştirip hem insan hem yapay zekâ tarafından yazılan metne uyguluyor; okunabilirlik puanlıyor. İngiltere'deki FCA Consumer Duty uyumu için bankalara konumlanıyor. | **Stil rehberinin makinece okunabilir kurallara dönüştürülmesi.** |
| **Smart Communications, Messagepoint** (müşteri iletişim yönetimi platformları) | Okunabilirlik, ton ve duygu analizi modülleri. | Anlaşılabilirlik, ayrı ve ölçülebilir bir boyut. |

**Sonuç:** Hepsi İngilizce ve ABD/İngiltere mevzuatı odaklı. Türkçe dil kuralları, BDDK veri yerleşimi gereklilikleri ve Reklam Kurulu içtihadı nedeniyle **kurum içi geliştirme** gerekçeli görünüyor.

### 3.2 Akademik ve kurumsal araştırmalar

#### Capital One — "A Multi-Stage Workflow for the Review of Marketing Content with Reasoning LLMs" (arXiv 2601.06054)

Doğrudan bizim problemimize en yakın yayımlanmış çalışma.

- **Mimari (2 aşama):**
  1. **Kural getirme:** Bir embedding modeli, metinle ilgili gereksinimleri sıralıyor; böylece LLM'e yalnızca kısa bir aday listesi gidiyor. Embedding modeli fine-tune edildiğinde, benzer recall (~%87-89) için gereken aday sayısı 88'den 50'ye düşüyor.
  2. **İhlal tespiti:** Fine-tune edilmiş küçük bir LLM (Llama 3.2 1B/3B, LoRA), her **(metin, gereksinim)** çifti için önce gerekçe, sonra Doğru/Yanlış kararı üretiyor.
- **Kurallar** harici bir ontolojiyle değil, ortalama 176 kelimelik **özetlenmiş metinler** olarak modele veriliyor.
- **Eğitim:** SFT ile GRPO (pekiştirmeli öğrenme) karşılaştırılmış. Gerekçe üretilen senaryoda GRPO daha iyi; en iyi sonuç 3B modelde **doğruluk ~0,67, kesinlik ~0,64, duyarlılık ~0,74**.
- **Yazarların sonucu:** Yaklaşım umut verici ama güvenilir olmaktan uzak; **insan denetimi olmadan kullanılması öngörülmüyor.**
- **Bize dersi:** (a) "Kural başına ikili karar + gerekçe" kalıbı denetlenebilir ve iyi bir tasarım; (b) küçük modeli fine-tune etmek tek başına yeterli kaliteyi vermiyor; (c) kural getirme katmanı (RAG) kendi başına büyük verim sağlıyor.

#### İngiltere FCA Consumer Duty (anlaşılabilirlik)

- FCA, bankaların müşteri iletişiminin **anlaşıldığını test etmesini** bekliyor. Mart 2026'da yayımlanan incelemede iyi uygulamalar olarak değişiklik öncesi/sonrası test, anket, anlama kontrolleri ve A/B testi öne çıkıyor.
- Yapay zekâ araçları bu testi **destekleyen**, yerini almayan araçlar olarak görülüyor.
- **Bize dersi:** "Anlaşılabilirlik" puanı yalnızca formülle değil, zaman içinde müşteri geri bildirimi ve şikâyet verisiyle de kalibre edilmeli.

### 3.3 Türkçe dil araçları (açık kaynak)

| Araç | Ne yapar | Lisans / durum | Sistemdeki olası rolü |
|---|---|---|---|
| **GECTurk** (Koç Ünv., IJCNLP-AACL 2023) | TDK yazım kurallarına dayalı **25 hata türü**, 130 binden fazla sentetik paralel cümle, elle etiketlenmiş zor test seti, veri üretim hattı ve temel modeller | Açık (GitHub: GGLAB-KU/gecturk) | Yazım/noktalama tespit modeli eğitimi ve **değerlendirme verisi**. Önemli bulgu: hata türü tespitine odaklı **küçük modeller, büyük ön-eğitimli modellerden daha iyi** sonuç verdi. |
| **GECTurk WEB** (2024/2025) | Hataları bulup düzeltiyor ve **hangi kuralın ihlal edildiğini açıklıyor**; BERTurk tabanlı sıralı etiketleyici. Kullanılabilirlik puanı 88,3. | Açık kaynak (GGLAB-KU/gecturkweb), gecturk.net | Açıklanabilir yazım kontrolü bileşeni için hazır şablon. |
| **Organic GEC** (Ersoy & Yıldız, 2024) | "Clean insertions" yöntemiyle Türkçe GEC verisi üretimi; mT5 modelleri | Makale + modeller | GEC modelini kurum verisiyle zenginleştirme yöntemi. |
| **de/da hata düzeltme** (Öztürk vd., 2020, Boğaziçi) | BERT tabanlı sıralı etiketleyici ile "de/da" bağlaç-ek karışıklığı tespiti | Gömmeler paylaşılmış | En sık hatalardan biri için uzman bileşen. |
| **LLM ile Türkçe yazım düzeltme** (Güzel Turhan, PeerJ 2025) | LLM'leri Türkçe yazım düzeltmede karşılaştırıyor; NoisyWikiTr veri seti | Makale | Bulgu: **dile özgü seq2seq model**, çok dilli modellerden ve BERT tabanlı modellerden daha iyi. |
| **Zemberek-NLP** | Morfolojik analiz, normalizasyon, temel yazım denetimi ve öneri | Apache 2.0; yavaş bakım modunda | Kök/ek analizi ve kelime düzeyi yazım kontrolü (kural motoru için). |
| **hunspell-tr** (tdd-ai) | Türkçe Hunspell sözlüğü | MPL 2.0 | Hızlı ilk filtre; kurum terimleriyle genişletilebilir. |
| **VNLP** (VNGRS, 2024) | Türkçe NLP paketi; Jamspell tabanlı yazım düzeltme | Açık kaynak (lisans kontrol edilmeli) | Alternatif yazım düzeltici. |
| **LanguageTool** | Kural tabanlı dil bilgisi denetleyicisi | LGPL | **Türkçeyi desteklemiyor** (GECTurk WEB makalesine göre). Kural XML yapısı örnek alınabilir. |
| **Vale** | YAML ile tanımlanan kurallarla metin "linter"ı | MIT | Türkçe morfolojiyi bilmez, ama **kurum dili, yasaklı ifade, terim ve format kuralları** için hafif ve denetlenebilir kural motoru. |
| **Okunabilirlik formülleri** | Ateşman (1997, Flesch uyarlaması), Bezirci-Yılmaz (2010, YOD), Çetinkaya-Uzun (2010) | Formül | Anlaşılabilirlik için hızlı, deterministik ilk sinyal. Formüller farklı sonuç verebiliyor; eşikler kurum metinleriyle kalibre edilmeli. |

### 3.4 Türkçe LLM'ler ve değerlendirme setleri

- **Kumru** (VNGRS): 7,4 milyar parametreli Türkçe odaklı model; 2B sürümü Ekim 2025'te açık kaynak olarak yayımlandı. Cetvel'de çok daha büyük modelleri geçtiğini belirtiyor *(üretici beyanı)*.
- **Trendyol-LLM-7B, Commencis-LLM-7B:** Mistral-7B tabanlı, Türkçe talimat verisiyle ince ayarlanmış modeller.
- **Cosmos** (YTÜ): Sıfırdan Türkçe eğitilmiş cosmosGPT ve Türkçe Llama türevleri.
- **Global açık modeller** (Qwen, Gemma, Llama ailesi): Cetvel'de Türkçe modellerle birlikte değerlendirilmiş.
- **Değerlendirme setleri:**
  - **Cetvel** (2025): Türkçe için 26 görevlik birleşik değerlendirme; düzeltme görevleri de içeriyor.
  - **TurkishMMLU** (EMNLP 2024): Türkçe çoktan seçmeli bilgi testi.
  - **TR-MTEB** (EMNLP 2025): Türkçe embedding modelleri için 26 veri setlik değerlendirme. RAG'de kural/doküman getirme için model seçiminde kullanılmalı (bge-m3 ve multilingual-e5 Türkçede güçlü görünüyor).
- **Model seçimi kuralı:** Genel değerlendirme setleri yol gösterir; asıl seçim **kendi metinlerinizden oluşan altın setle** yapılmalı (Bölüm 7).

### 3.5 LLM ile dil düzeltmenin bilinen riski: aşırı düzeltme

- LLM'ler hatayı düzeltirken metni "akıcılaştırma" eğiliminde. "Yalnızca gerekli düzeltmeyi yap" talimatı bu sorunu tam çözmüyor (Çince, Ukraynaca ve İngilizce çalışmalarda gösterilmiş).
- **Banka bağlamındaki tehlike:** Hukuk tarafından onaylanmış yasal uyarı cümlelerinin, ürün adlarının veya rakamların "iyileştirilmesi".
- **Literatürdeki çözümler:**
  - **Düzenleme düzeyinde çoğunluk oylaması** (2026): Aynı modelden birkaç düzeltme alınıp yalnızca tekrar eden düzenlemeler tutuluyor. Eğitim gerektirmeden kesinliği artırıyor.
  - **Minimal düzenleme için fine-tune** (2025): Eğitim verisinin hazırlanışı, eğitim yönteminden daha belirleyici.
  - **Değerlendirmede F0.5:** Kesinliği duyarlılıktan iki kat önemli sayıyor, çünkü yanlış öneri kaçırılan hatadan daha zararlı kabul ediliyor.
- **Tasarım önlemleri** (Bölüm 5-F'de): LLM'den tam metin değil **"orijinal ifade → önerilen ifade + gerekçe" listesi** istemek ve **korumalı ifadeleri** kilitlemek.
- İlgili açık model: **mEdIT** (Grammarly, 2024), çok dilli metin düzenleme için talimatla eğitilmiş modeller. Türkçeyi kapsamıyor ama veri ve eğitim yöntemi örnek alınabilir.

### 3.6 Doğru bilgi kontrolü (olgusal tutarlılık)

| Model | Yaklaşım | Türkçe desteği |
|---|---|---|
| **MiniCheck** (EMNLP 2024) | 770M parametreli Flan-T5 tabanlı doğrulayıcı, "iddia kaynak dokümanla destekleniyor mu?" sorusunda GPT-4 seviyesinde | Yok (İngilizce) |
| **Bespoke-MiniCheck-7B** | Aynı yöntemle 7B model | Yok; ticari olmayan lisans |
| **AlignScore** (2023) | RoBERTa tabanlı tutarlılık skoru | Yok |
| **HHEM-2.1-Open** (Vectara) | Halüsinasyon tespit modeli, Apache 2.0 | Yok; çok dilli sürüm (HHEM-2.3) yalnızca ticari API |

**Sonuç:** Türkçe için hazır açık kaynak doğrulayıcı yok. Bizim senaryomuzda bu aslında avantaj, çünkü doğrulanacak bilgilerin çoğu **yapılandırılmış** (oran, tutar, tarih, alt/üst limit, vade):

1. LLM metinden bu değerleri JSON olarak **çıkarır**.
2. Kod, bu değerleri kampanya kayıt sistemi ve ürün parametreleriyle **deterministik olarak karşılaştırır**.
3. Yapılandırılamayan koşullar (ör. "yalnızca yeni müşteriler") için LLM, kaynak dokümanı da görerek "destekleniyor / desteklenmiyor / kaynakta yok" kararı verir.
4. İleride MiniCheck yöntemiyle Türkçe sentetik veriden küçük bir doğrulayıcı eğitilebilir.

### 3.7 Altyapı ve yöntem araçları

- **Prompt optimizasyonu — DSPy:** Prompt'ları etiketli örneklere göre otomatik optimize ediyor. `BootstrapFinetune` ile büyük modelin davranışını küçük modele damıtabiliyor. Fine-tune'a göre daha az veriyle başlanabilecek bir ara yol.
- **LLM-as-a-judge (modeli değerlendirici olarak kullanma):** Uygulamadaki öneriler şöyle: ikili (geçti/kaldı) kriterler 1-5 ölçeklerinden daha tutarlı; değerlendirici, insan tarafından etiketlenmiş altın setle doğrulanmalı; değerlendirici model, prompt ve rubrik sürümleri her çalışmayla birlikte saklanmalı. Araçlar: DeepEval, Langfuse, Opik, Arize Phoenix, promptfoo, Autorubric.
- **Fine-tune araçları:** LoRA/QLoRA için Unsloth, LLaMA-Factory, Axolotl; GRPO ve DPO için Hugging Face TRL.
- **Model sunumu:** vLLM (yüksek verim, JSON şemasına zorlanmış yapılandırılmış çıktı); prototip için Ollama.
- **Etiketleme ve geri bildirim:** Argilla, Label Studio.
- **Orkestrasyon:** LangGraph (çok adımlı ve ajanlı akışlar) veya sade Python + FastAPI.

---

## 4. Mevzuat ve kurum içi kısıtların tasarıma etkisi

| Kısıt | Özet | Tasarıma etkisi |
|---|---|---|
| **BDDK — Bankaların Bilgi Sistemleri ve Elektronik Bankacılık Hizmetleri Hakkında Yönetmelik (2020)** | Bulut hizmeti, sistemlerin yurt içinde bulunması koşuluyla kullanılabiliyor; birincil ve ikincil sistemlerin yurt içinde konumlanması zorunlu. | Yurt dışı API tabanlı LLM'ler yerine **kurum içi veya yurt içi barındırılan açık kaynak modeller**. Ekibin mevcut tercihiyle uyumlu. |
| **Ticari Reklam ve Haksız Ticari Uygulamalar Yönetmeliği** (2015; 2022 değişikliği) | Mal/hizmet satışıyla birlikte sunulan finansman reklamlarında vade, faiz oranı, tüketiciye toplam maliyet ve geri ödeme koşulları yer almalı. | Kural kütüphanesinde **ürün türüne bağlı zorunlu bilgi kuralları**. |
| **Reklam Kurulu kararları** | Kredi reklamlarında faiz dışındaki tüm ücret, komisyon ve masrafların; toplam maliyetin ve yıllık maliyet oranının **ana mesajda** yer alması bekleniyor. Ocak 2025 toplantısında incelenen 209 dosyanın 189'unda ihlal bulundu. | Yalnızca "ibare var mı?" değil, **"ana mesajda mı?"** kontrolü (konum ve görünürlük). Kurul kararları RAG kaynağı olarak kullanılabilir. |
| **Ticari elektronik ileti mevzuatı** (6563 sayılı Kanun ve yönetmeliği) | Ticari iletilerde ret (iletişim izni geri alma) imkânına ilişkin bilgi bulunmalı; bilgilendirme mesajları farklı değerlendiriliyor. | Mesaj türü sınıflandırması + kanal bazlı zorunlu ibare kontrolü. |
| **Sorumluluk ve model riski** | Nihai karar insanda. | Sistem **karar destek aracı**: her bulgu gerekçe ve dayanakla sunulur; onaycının kararı kayıt altına alınır (denetim izi). |

> ⚠️ Kaynaklarda 2026 yılı içinde reklam mevzuatında değişiklik yapıldığına dair işaretler var ancak içeriği doğrulanamadı. Kural kütüphanesi oluşturulurken Resmî Gazete'deki güncel metin esas alınmalı.

**Kural kütüphanesi önerilen kayıt yapısı:**

```yaml
- id: REK-KREDI-004
  baslik: Kredi reklamında yıllık maliyet oranı ana mesajda yer almalı
  kapsam:
    urun: [ihtiyac_kredisi, tasit_kredisi, konut_kredisi]
    mesaj_turu: [ticari]
    kanal: [sms, eposta, push, banner, web]
  kontrol_tipi: llm          # deterministik | llm | hibrit
  ciddiyet: engelleyici      # engelleyici | onemli | oneri
  dayanak: "Ticari Reklam ve Haksız Ticari Uygulamalar Yönetmeliği md. X; Reklam Kurulu kararı ..."
  aciklama: >
    Faiz oranı belirtilen kredi reklamlarında yıllık maliyet oranı ve toplam
    maliyet, dipnotta değil ana mesajda yer almalıdır.
  ihlal_ornekleri:
    - "Aylık %2,49 faizle ihtiyaç kredisi! Detaylar web sitemizde."
  uygun_ornekler:
    - "Aylık %2,49 faiz, yıllık maliyet oranı %X, toplam geri ödeme Y TL ..."
  sahibi: Mevzuat Uyum
  gecerlilik_baslangic: 2026-01-01
```

Bu yapı hem deterministik motorun hem de RAG katmanının ortak kaynağı olur. Kural güncellendiğinde model yeniden eğitilmez; yalnızca kayıt güncellenir.

---

## 5. Alternatif sistem mimarileri

### A. Tek bir fine-tune edilmiş LLM (ekibin ilk fikri)

- **Nasıl:** Geçmiş inceleme kayıtlarından *(orijinal metin → revize metin + revize gerekçesi)* çiftleri çıkarılır; açık kaynak bir model LoRA/QLoRA ile ince ayarlanır.
- **Artıları:** Kurum dilini ve ekibin revize üslubunu öğrenir; tek bir model, tek bir çağrı; işletim maliyeti düşük.
- **Eksileri:**
  - **Doğru bilgi kontrolünü çözmez:** Model kampanyanın gerçek parametrelerini bilmez.
  - Kural değiştiğinde yeniden eğitim gerekir; bilginin güncelliği eğitim tarihine bağlı kalır.
  - Bulguların dayanağı gösterilemez; açıklanabilirlik ve denetlenebilirlik düşük.
  - Temiz ve tutarlı etiketli veri gerektirir. Geçmiş revizeler çoğunlukla gerekçesiz veya tutarsız olabilir.
  - Aşırı düzeltme ve katastrofik unutma (genel dil yeteneğinin bozulması) riski.
  - Capital One örneğinde fine-tune edilmiş küçük modeller ~%67 doğrulukta kaldı.
- **Ne zaman uygun:** İkinci veya üçüncü aşamada, biriken veriyle **dar ve iyi tanımlı görevler** için (ör. yazım hatası tespiti, kural ihlali sınıflandırması, öneri üslubu).

### B. Genel açık kaynak LLM + prompt + RAG (fine-tune yok)

- **Nasıl:** Kural kütüphanesi, TDK yazım kuralları özetleri, kurum stil rehberi ve ürün dokümanları vektör veritabanında tutulur. Her metin için ilgili kurallar getirilir; LLM, JSON şemasına zorlanmış yapılandırılmış çıktı üretir.
- **Artıları:** Hızlı başlangıç; kural değişince yalnızca doküman güncellenir; her bulgu kaynağıyla sunulabilir; etiketli veri ihtiyacı düşük.
- **Eksileri:** Prompt'a duyarlılık ve tutarsızlık; iyi Türkçe için daha büyük model (dolayısıyla GPU) gerekebilir; aşırı düzeltme; gecikme.
- **Ne zaman uygun:** **MVP için en hızlı yol.** Sonraki tüm seçeneklerin de temeli.

### C. Kural tabanlı + klasik NLP (LLM'siz)

- **Nasıl:** hunspell-tr ve Zemberek ile yazım kontrolü, Vale tarzı YAML kuralları (terim, yasaklı ifade, format), zorunlu ibare kontrolleri, okunabilirlik formülleri, tarih ve sayı doğrulama.
- **Artıları:** Deterministik, açıklanabilir, çok ucuz ve hızlı, tamamen denetlenebilir.
- **Eksileri:** Anlatım bozukluğu ve anlam düzeyindeki hataları yakalayamaz; yanlış pozitif oranı yüksek olabilir; kural bakımı emek ister.
- **Ne zaman uygun:** **Tek başına yetersiz, ama her mimaride ilk katman olmalı.** Ucuz ve kesin kontrolleri LLM'e bırakmamak gerekir.

### D. Uzman modeller hattı

- **Nasıl:** Her kontrol için ayrı küçük model: GECTurk tipi BERTurk sıralı etiketleyici (yazım/noktalama), de/da modeli, kural ihlali sınıflandırıcısı, Türkçe doğrulayıcı (NLI). LLM yalnızca düzeltme önerisi yazmak için kullanılır.
- **Artıları:** Her bileşen ayrı ölçülebilir ve iyileştirilebilir; küçük modeller hızlı ve CPU'da bile çalışabilir. GECTurk bulgusu (küçük, hata türüne odaklı modeller daha iyi) bu yaklaşımı destekliyor.
- **Eksileri:** Çok bileşen, çok bakım; her bileşen için ayrı etiketli veri.
- **Ne zaman uygun:** Ölçek büyüdüğünde ve veri biriktiğinde; özellikle yazım ve noktalama için.

### E. Çok ajanlı inceleme

- **Nasıl:** Uzman ajanlar (İmla, Anlaşılabilirlik, Doğruluk, Mevzuat) ayrı prompt veya modellerle çalışır. Doğruluk ajanı, kampanya ve ürün API'lerini araç olarak çağırır. Bir "Baş Editör" ajanı bulguları birleştirir, çelişkileri çözer ve nihai öneriyi oluşturur. Persado'nun "ajan komitesi" bu kalıba örnek.
- **Artıları:** Modüler; her ajan bağımsız geliştirilip test edilebilir; araç kullanımı doğal.
- **Eksileri:** Gecikme ve maliyet artar; hatalar ajanlar arasında yayılabilir; test etmek ve hata ayıklamak zorlaşır.
- **Ne zaman uygun:** Hibrit mimarinin LLM katmanı büyüdüğünde, o katmanı organize etmenin bir yolu olarak.

### F. Katmanlı hibrit mimari (önerilen)

```
                   ┌──────────────────────────────────────────────┐
  Talep            │ GİRİŞ: metin + kanal + mesaj türü + ürün kodu  │
  (onay akışı) ───▶│        + kampanya kaydı kimliği               │
                   └──────────────────────┬───────────────────────┘
                                          ▼
   ┌───────────────────────────────────────────────────────────────────┐
   │ KATMAN 1 — Deterministik kontroller (ms düzeyinde, LLM yok)         │
   │ • Sözlük ile yazım (hunspell-tr / Zemberek) + kurum terim sözlüğü   │
   │ • Regex kurallar: yasaklı ifade, sayı/para/tarih formatı, geçersiz  │
   │   tarih, URL ve kısa numara formatı                                 │
   │ • Kanal kuralları: SMS uzunluğu, ret bilgisi vb. zorunlu ibareler   │
   │ • Okunabilirlik: Ateşman / Bezirci-Yılmaz, cümle uzunluğu           │
   └───────────────────────────────┬───────────────────────────────────┘
                                   ▼
   ┌───────────────────────────────────────────────────────────────────┐
   │ KATMAN 2 — Doğru bilgi kontrolü                                     │
   │ • LLM metinden oran, tutar, tarih, limit, koşul çıkarır (JSON)      │
   │ • Kod, değerleri kampanya kaydı ve ürün parametreleriyle karşılaştırır │
   │ • Yapılandırılamayan koşullar: kaynak dokümana dayalı LLM kararı    │
   └───────────────────────────────┬───────────────────────────────────┘
                                   ▼
   ┌───────────────────────────────────────────────────────────────────┐
   │ KATMAN 3 — LLM incelemesi (RAG)                                     │
   │ • Kural kütüphanesinden ilgili kurallar getirilir (embedding)       │
   │ • Kural başına karar: ihlal var mı? + gerekçe + dayanak             │
   │ • Dil bilgisi, anlatım, noktalama, anlaşılabilirlik bulguları       │
   │ • Düzeltme: "orijinal ifade → önerilen ifade + gerekçe" listesi     │
   │ • Korumalı ifadeler (yasal metin, ürün adı, rakam) değiştirilemez   │
   └───────────────────────────────┬───────────────────────────────────┘
                                   ▼
   ┌───────────────────────────────────────────────────────────────────┐
   │ KATMAN 4 — Doğrulama                                                │
   │ • Çoklu örnekleme + düzenleme düzeyinde çoğunluk oylaması           │
   │ • Önerilen metin Katman 1 ve 2'den tekrar geçirilir                 │
   │   (öneri yeni hata yaratmamalı)                                     │
   │ • Değerlendirici model: anlam korundu mu? bulgu yerinde mi?         │
   └───────────────────────────────┬───────────────────────────────────┘
                                   ▼
   ┌───────────────────────────────────────────────────────────────────┐
   │ ÇIKTI — Rapor + insan onayı                                         │
   │ • Bulgular: kategori, ciddiyet, konum, dayanak, öneri, güven        │
   │ • Özet karar önerisi: Onay / Revize                                 │
   │ • Onaycı her bulguyu kabul eder ya da reddeder ──┐                  │
   └──────────────────────────────────────────────────┼────────────────┘
                                                      ▼
                         GERİ BİLDİRİM VERİSİ → altın set büyür →
                         ölçüm → hedefli fine-tune / damıtma (Faz 3)
```

**Neden bu mimari?**

| İlke | Karşılığı |
|---|---|
| Ucuz ve kesin olan işi LLM'e bırakma | Katman 1 |
| Olgusal doğruluğu modelin hafızasına değil, kaynak veriye dayandır | Katman 2 |
| Kuralları modelin ağırlıklarında değil, güncellenebilir bir kütüphanede tut | Katman 3 (RAG) |
| Her bulgu dayanağıyla birlikte gelsin | Kural başına karar + `dayanak` alanı |
| Aşırı düzeltmeyi yapısal olarak engelle | Diff listesi, korumalı ifadeler, çoğunluk oylaması |
| Fine-tune için veriyi sistem kendisi üretsin | Onaycı kabul/ret geri bildirimi |

**Bulgu çıktısı örneği (API yanıtı):**

```json
{
  "inceleme_id": "INC-2026-11-000123",
  "ozet_karar_onerisi": "revize",
  "bulgular": [
    {
      "id": "B-1",
      "katman": "dogru_bilgi",
      "kategori": "tutar_uyusmazligi",
      "ciddiyet": "engelleyici",
      "konum": {"baslangic": 81, "bitis": 87},
      "metindeki": "100 TL",
      "beklenen": "75 TL",
      "aciklama": "Kampanya kaydında bonus tutarı 75 TL olarak tanımlı.",
      "dayanak": "Kampanya kaydı KMP-2026-0412",
      "guven": 0.99
    },
    {
      "id": "B-2",
      "katman": "deterministik",
      "kategori": "gecersiz_tarih",
      "ciddiyet": "engelleyici",
      "metindeki": "31.11.2026",
      "aciklama": "Kasım ayı 30 gündür.",
      "dayanak": "Tarih doğrulama kuralı DT-001"
    },
    {
      "id": "B-3",
      "katman": "llm",
      "kategori": "noktalama",
      "ciddiyet": "oneri",
      "metindeki": "geçerlidir, katılım için",
      "onerilen": "geçerlidir. Katılım için",
      "aciklama": "İki bağımsız cümle virgülle bağlanmış.",
      "dayanak": "TDK Yazım Kılavuzu — nokta",
      "guven": 0.82
    }
  ],
  "onerilen_metin": "…",
  "surum": {"model": "…", "prompt": "v12", "kural_kutuphanesi": "2026-11-01"}
}
```

**Uçtan uca örnek:**

> *"Değerli müşterimiz, kredi kartınızla yapacağınız 1000 TL ve üzeri alışverişlerde 100 TL bonus kazanma fırsatını kaçırmayın! Kampanya 31.11.2026 tarihine kadar geçerlidir, katılım için KAMPANYA yazıp 4444'e gönderin."*

| Katman | Yakalanan |
|---|---|
| 1 — Deterministik | "31.11.2026" geçersiz tarih; "1000 TL" kurum standardına göre "1.000 TL"; ticari SMS'te ret bilgisi eksik |
| 2 — Doğru bilgi | Kampanya kaydında bonus 75 TL, alt limit 1.500 TL → iki uyuşmazlık (engelleyici) |
| 3 — LLM | Virgülle bağlanmış iki cümle; kampanya koşulları (peşin/taksitli, işlem sayısı, üst limit) belirsiz → anlaşılabilirlik bulgusu |
| 4 — Doğrulama | Önerilen metin Katman 1 ve 2'den yeniden geçirilir; rakamların kayıtla tutarlılığı tekrar teyit edilir |

---

## 6. Karşılaştırma tablosu

> Bu tablo, kaynaklardaki bulgulardan ve genel uygulama deneyiminden çıkarılmış bir değerlendirmedir; ekibin kendi ölçümleriyle güncellenmelidir.

| Kriter | A. Fine-tune | B. Prompt+RAG | C. Kural | D. Uzman modeller | E. Çok ajan | **F. Hibrit** |
|---|---|---|---|---|---|---|
| İlk sürüme ulaşma hızı | Yavaş | **Hızlı** | **Hızlı** | Yavaş | Orta | Orta |
| Yazım / noktalama | İyi | Orta-İyi | Orta | **İyi** | Orta-İyi | **İyi** |
| Anlatım / anlaşılabilirlik | İyi | İyi | Zayıf | Orta | İyi | **İyi** |
| Doğru bilgi | **Zayıf** | Orta | Kısmi (format) | Orta | İyi | **İyi** |
| Mevzuat kuralları | Orta | İyi | İyi (açık kurallarda) | Orta | İyi | **İyi** |
| Açıklanabilirlik / dayanak | Düşük | Orta-İyi | **Yüksek** | Orta | Orta | **Yüksek** |
| Kural değişikliğine uyum | **Zayıf** (yeniden eğitim) | İyi | İyi | Orta | İyi | **İyi** |
| Etiketli veri ihtiyacı | **Yüksek** | Düşük | Yok | Yüksek | Düşük | Düşük, zamanla artar |
| İşletim maliyeti | Düşük-Orta | Orta | Çok düşük | Düşük | **Yüksek** | Orta |

---

## 7. Fine-tune ne zaman ve nasıl yapılmalı?

**Ön koşul: ölçüm.** Önce B/F mimarisinin temel sürümü altın sette ölçülür. Fine-tune yalnızca **hedefin altında kalan belirli kategoriler** için yapılır.

| Seçenek | Ne için | Not |
|---|---|---|
| **LoRA SFT (3-8B model)** | Hata tespiti ve sınıflandırma, kurum üslubunda öneri | En yaygın ve ucuz yol. Unsloth / LLaMA-Factory. |
| **Minimal düzenleme GEC fine-tune** | Yazım/noktalama düzeltmesi | GECTurk verisi + kurum verisi. Veri hazırlığı, yöntemden daha belirleyici. |
| **GRPO (pekiştirmeli öğrenme)** | Gerekçe üreten kural ihlali tespiti | Capital One'da gerekçeli senaryoda SFT'den iyi sonuç verdi. TRL. |
| **Damıtma** | Büyük modelin kalitesini küçük, hızlı modele aktarma | Büyük model + iyi prompt ile etiket üret, küçük modeli eğit (DSPy BootstrapFinetune). |
| **Embedding fine-tune** | Kural ve doküman getirme kalitesi | Capital One'da gereken aday sayısını 88'den 50'ye düşürdü. Görece ucuz ve etkili. |
| **Uzman küçük modeller** | de/da, ki, büyük harf, kesme işareti | GECTurk bulgusu: küçük, odaklı modeller daha iyi. |

**Veri kaynağı:** Geçmiş inceleme kayıtları (orijinal metin, revize metin, revize yorumu). Bu kayıtlar **hata türü etiketleriyle** yapılandırılmalı (GECTurk'ün 25 türlük şeması başlangıç noktası olabilir). Sistem devreye girdikten sonra onaycıların kabul/ret kararları en temiz veri kaynağı olur.

---

## 8. Değerlendirme stratejisi

- **Altın set:** Geçmişten seçilmiş birkaç yüz metin (kanal, ürün ve mesaj türü dağılımı gözetilerek). Uzmanlar her hatanın konumunu, türünü, ciddiyetini ve doğru düzeltmesini etiketler. Mümkünse iki kişi etiketler ve aralarındaki uyum ölçülür.
- **Kategoriye göre farklı hedef:**
  - **Doğru bilgi ve mevzuat:** *Duyarlılık* öncelikli. Yanlış bir faiz oranını kaçırmak, yanlış alarm vermekten çok daha maliyetli.
  - **Yazım, üslup ve öneriler:** *Kesinlik* öncelikli (F0.5). Gereksiz öneriler onaycıları yorar ve sisteme güveni azaltır.
- **Operasyonel metrikler:** İnceleme süresi, metin başına revize tur sayısı, önerilerin kabul oranı, onaydan sonra fark edilen hata sayısı.
- **Gölge mod:** İlk dönemde sistem çalışır ama karar tamamen insanda kalır; sistem bulguları ile insan kararları karşılaştırılır.
- **Sürümleme:** Model, prompt, kural kütüphanesi ve değerlendirici rubriği her inceleme kaydıyla birlikte saklanır (denetim izi ve regresyon testi için).

---

## 9. Önerilen yol haritası

| Faz | İçerik | Çıkış kriteri |
|---|---|---|
| **Faz 0 — Envanter** | Geçmiş inceleme kayıtlarının toplanması; stil rehberi, terim listesi ve mevzuat maddelerinin **kural kütüphanesine** dönüştürülmesi; altın setin etiketlenmesi; model ve donanım envanteri | Kural kütüphanesinin ilk sürümü ve altın set hazır |
| **Faz 1 — MVP** | Katman 1 (deterministik) + Katman 3 (prompt + RAG) + bulgu raporu API'si; gölge modda çalıştırma | Altın sette kategori bazlı ilk ölçüm; onaycı geri bildirimi toplanıyor |
| **Faz 2 — Doğruluk ve güvence** | Katman 2 (kampanya/ürün verisi entegrasyonu) + Katman 4 (doğrulama, çoğunluk oylaması, korumalı ifadeler) | Doğru bilgi kategorisinde hedef duyarlılık; onaycı kabul oranı hedefte |
| **Faz 3 — Özelleştirme** | Toplanan geri bildirim verisiyle hedefli fine-tune, damıtma ve uzman küçük modeller; embedding fine-tune | Fine-tune edilen kategoride altın set skorunda anlamlı iyileşme |

---

## 10. Önerilen açık kaynak teknoloji yığını

| Bileşen | Seçenekler |
|---|---|
| LLM sunumu | vLLM (yapılandırılmış çıktı), prototip için Ollama |
| LLM adayları | Qwen, Gemma, Llama aileleri; Türkçe odaklı Kumru, Trendyol-LLM, Cosmos. Seçim kendi altın setinizle yapılmalı. |
| Embedding | bge-m3, multilingual-e5; TR-MTEB'deki Türkçe modeller |
| Vektör veritabanı | pgvector (PostgreSQL) veya Qdrant |
| Orkestrasyon / API | FastAPI + sade Python; gerekirse LangGraph |
| Yazım ve morfoloji | hunspell-tr, Zemberek-NLP, VNLP; GECTurk modelleri |
| Kural motoru | YAML kural kütüphanesi + Python kural motoru (Vale yaklaşımı örnek alınabilir) |
| Değerlendirme ve gözlem | DeepEval, promptfoo, Langfuse |
| Prompt optimizasyonu | DSPy |
| Fine-tune | Unsloth, LLaMA-Factory, Hugging Face TRL |
| Etiketleme | Argilla, Label Studio |

---

## 11. Ekiple netleştirilmesi gereken sorular

1. Geçmiş inceleme kayıtları hangi formatta tutuluyor, kaç adet var ve revize gerekçeleri yazılı mı?
2. Kampanya parametreleri (oran, tutar, tarih, limit, koşul) yapılandırılmış bir sistemde tutuluyor mu? **Doğru bilgi kontrolünün kalitesi buna bağlı.**
3. Şu an kullanılan açık kaynak LLM hangisi ve hangi donanımda (GPU türü ve sayısı) çalışıyor?
4. Hangi kanallar kapsamda: SMS, push, e-posta, uygulama içi, web, şube materyali? Görsel ve banner içindeki metinler de kapsamda mı? (Kapsamdaysa OCR gerekir.)
5. Kural kütüphanesinin sahibi kim olacak: Müşteri Deneyimi mi, Mevzuat Uyum mu, ortak mı?
6. Sistem onay akışına nasıl bağlanacak: mevcut iş akışı aracına API ile mi, ayrı bir arayüzle mi?

---

## Kaynaklar

**Finans sektörü ve kurumsal çalışmalar**
- Capital One — A Multi-Stage Workflow for the Review of Marketing Content with Reasoning LLMs: https://arxiv.org/abs/2601.06054
- Saifr: https://saifr.ai/saifrreview
- Fidelity'nin Saifr duyurusu (2022): https://www.businesswire.com/news/home/20220124005136/en/
- Saifr risk yorumlama ve önerilen dil modelleri (2023): https://www.businesswire.com/news/home/20230515005854/en/
- Persado Marketing Compliance AI (2025): https://www.persado.com/press-releases/persado-debuts-marketing-compliance-agentic-ai-for-financial-services/
- Acrolinx — Consumer Duty and Clear Communication: https://www.acrolinx.com/uk/blog/consumer-duty-and-clear-communication/
- FCA tüketici anlayışı incelemesi özeti (Forvis Mazars): https://www.forvismazars.com/uk/en/insights/industry-and-sector-insights/financial-services-insights/fca-consumer-understanding
- Capgemini — Generative AI for marketing disclosures: https://www.capgemini.com/ca-en/insights/research-library/how-banks-can-use-generative-ai-to-enhance-marketing-disclosures/

**Türkçe dil işleme**
- GECTurk: https://arxiv.org/abs/2309.11346 — kod ve veri: https://github.com/GGLAB-KU/gecturk
- GECTurk WEB: https://arxiv.org/abs/2410.12350 — https://gecturk.net
- Organic Data-Driven Approach for Turkish GEC and LLMs: https://arxiv.org/abs/2405.15320
- de/da hataları için bağlamsal gömmeler (Öztürk vd., 2020): https://tabilab.cmpe.bogazici.edu.tr/papers/ozturk2020role
- Leveraging LLMs for Spelling Correction in Turkish (PeerJ 2025): https://peerj.com/articles/cs-2889
- Zemberek-NLP: https://github.com/ahmetaa/zemberek-nlp
- hunspell-tr: https://github.com/tdd-ai/hunspell-tr
- VNLP: https://arxiv.org/abs/2403.01309
- LanguageTool dil listesi: https://languagetool.org/languages
- Vale: https://vale.sh
- Türkçe okunabilirlik formülleri üzerine bir çalışma: https://dergipark.org.tr/tr/pub/adeder/article/708220
- Türkçe NLP kaynak listesi: https://github.com/agmmnn/turkish-nlp-resources

**Türkçe LLM'ler ve değerlendirme**
- Kumru LLM: https://medium.com/vngrs/kumru-llm-34d1628cfd93
- Cetvel: https://arxiv.org/abs/2508.16431
- TurkishMMLU: https://aclanthology.org/2024.findings-emnlp.413/
- Türkçe dil modelleri karşılaştırması (2024): https://arxiv.org/abs/2404.17010
- TR-MTEB: https://aclanthology.org/2025.findings-emnlp.471/

**LLM ile düzeltme, doğrulama ve değerlendirme**
- Is ChatGPT a Highly Fluent GEC System? (2023): https://arxiv.org/abs/2304.01746
- Adapting LLMs for Minimal-edit GEC (2025): https://arxiv.org/abs/2506.13148
- Düzenleme düzeyinde çoğunluk oylaması (2026): https://arxiv.org/abs/2605.13624
- mEdIT: https://arxiv.org/abs/2402.16472
- MiniCheck: https://arxiv.org/abs/2404.10774
- AlignScore: https://arxiv.org/abs/2305.16739
- HHEM: https://huggingface.co/vectara/hallucination_evaluation_model
- Bespoke-MiniCheck-7B: https://huggingface.co/bespokelabs/Bespoke-MiniCheck-7B
- DSPy: https://dspy.ai
- MLflow — LLM-as-a-judge yazıları: https://mlflow.org/articles/tags/llm-as-a-judge/
- Autorubric: https://arxiv.org/abs/2603.00077

**Mevzuat**
- Ticari Reklam ve Haksız Ticari Uygulamalar Yönetmeliği (Resmî Gazete, 10.01.2015): https://resmigazete.gov.tr/eskiler/2015/01/20150110-5.htm
- 2022 değişikliği özeti: https://cottgroup.com/en/legislation/item/ticari-reklam-ve-haksiz-ticari-uygulamalarda-degisiklik
- Reklam Kurulu Ocak 2025 kararları: https://www.finansopia.com/ekonomi/reklam-kurulu-yilin-ilk-toplantisinda-ceza-yagdirdi-10-bankaya-reklam-yasagi-getirildi/
- BDDK Bilgi Sistemleri Yönetmeliği haberi: https://www.bloomberght.com/bddk-dan-bankalarin-bilgi-sistemleri-ve-elektronik-bankacilik-hizmetleri-hakkinda-yonetmelik-2249537
