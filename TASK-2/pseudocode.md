// ============================================================================
//     E-TİCARET SİSTEMİ 
// ============================================================================
// Bu kod, bir kullanıcının ürünü sepete eklemesinden siparişin tamamlanmasına
// kadar olan tüm süreci tek ve bütünsel bir akışta birleştirir.
// Gerçek bir mimaride bu fonksiyonlar farklı modül/sınıf/servislerde yer alırdı.
// ============================================================================

// --- BÖLÜM 1: VERİ YAPILARI VE TEMEL FONKSİYONLAR ---

OBJECT Kullanici: { id, oturumId, email }
OBJECT Sepet: { id, kullaniciId, urunListesi: [], toplamTutar, indirimKodu }
OBJECT Siparis: { id, kullaniciId, urunler: [], genelToplam, durum }

// Bu fonksiyon, kullanıcının eylemine göre ana yönlendirmeyi yapar.
FONKSIYON AnaIstekYonlendirici(istek):
    kullanici = OturumdanKullaniciGetir(istek.cookie);
    
    EĞER istek.eylem == "SEPETE_EKLE" İSE
        YANIT = SepeteUrunEkle(kullanici, istek.urunId, istek.miktar);
    DEĞİLSE EĞER istek.eylem == "ODEMEYI_TAMAMLA" İSE
        YANIT = OdemeyiVeSiparisiTamamla(kullanici, istek.odemeBilgileri);
    BİTİR EĞER
    
    KullaniciyaYanitGonder(YANIT);
BİTİR FONKSIYON


// --- BÖLÜM 2: SEPETE ÜRÜN EKLEME SÜRECİ ---

FONKSIYON SepeteUrunEkle(kullanici, urunId, miktar):
    BAŞLA: ÜRÜN EKLEME
    
    // KONTROL NOKTASI: STOK KONTROLÜ
    urun = Veritabani.UrunGetir(urunId);
    EĞER urun.stokAdedi < miktar İSE
        DÖN "Hata: Ürün için stok yetersiz.";
    BİTİR EĞER
    
    aktifSepet = Veritabani.SepetiGetir(kullanici.id);
    urunSepetteVarMi = YANLIŞ;
    
    // Sepetteki mevcut ürünleri kontrol et
    DÖNGÜ aktifSepet.urunListesi İÇİNDEKİ HER sepetUrunu İÇİN:
        EĞER sepetUrunu.id == urunId İSE
            sepetUrunu.miktar += miktar; // Miktarı artır
            urunSepetteVarMi = DOĞRU;
            DÖNGÜYÜ KIR;
        BİTİR EĞER
    BİTİR DÖNGÜ
    
    EĞER urunSepetteVarMi == YANLIŞ İSE
        yeniUrun = {id: urunId, miktar: miktar, fiyat: urun.fiyat};
        aktifSepet.urunListesi.EKLE(yeniUrun); // Yeni ürün olarak ekle
    BİTİR EĞER
    
    aktifSepet.toplamTutar = SepetToplaminiHesapla(aktifSepet);
    Veritabani.SepetiKaydet(aktifSepet);
    
    BİTİR: ÜRÜN EKLEME
    DÖN "Başarılı: Ürün sepete eklendi.";
BİTİR FONKSIYON


// --- BÖLÜM 3: ÖDEME VE SİPARİŞİ TAMAMLAMA SÜRECİ (EN KRİTİK BÖLÜM) ---

FONKSIYON OdemeyiVeSiparisiTamamla(kullanici, odemeBilgileri):
    BAŞLA: SİPARİŞİ TAMAMLAMA SÜRECİ
    
    aktifSepet = Veritabani.SepetiGetir(kullanici.id);
    
    // KONTROL NOKTASI 1: ÖDEME ÖNCESİ SON STOK KONTROLÜ
    stoklarYeterliMi = DOĞRU;
    DÖNGÜ aktifSepet.urunListesi İÇİNDEKİ HER sepetUrunu İÇİN:
        urunStok = Veritabani.StokAdediGetir(sepetUrunu.id);
        EĞER urunStok < sepetUrunu.miktar İSE
            stoklarYeterliMi = YANLIŞ;
            DÖNGÜYÜ KIR;
        BİTİR EĞER
    BİTİR DÖNGÜ
    
    EĞER stoklarYeterliMi == YANLIŞ İSE
        BİTİR: SİPARİŞİ TAMAMLAMA SÜRECİ
        DÖN "Hata: Sepetinizdeki bir ürünün stoğu tükendi. Lütfen sepetinizi güncelleyin.";
    BİTİR EĞER

    // KONTROL NOKTASI 2: ÖDEME AĞ GEÇİDİ İLE İLETİŞİM
    yeniSiparis = Veritabani.YeniSiparisOlustur(kullanici, aktifSepet, "Ödeme Bekleniyor");
    odemeYaniti = OdemeAgGecidi.OdemeIstegiGonder(yeniSiparis.genelToplam, odemeBilgileri, yeniSiparis.id);
    
    // KONTROL NOKTASI 3: ÖDEME SONUCUNUN DOĞRULANMASI
    EĞER odemeYaniti.durum == "BAŞARILI" İSE
        // --- KRİTİK BÖLGE: Bu işlemlerin tamamı ya başarılı olmalı ya da hiç yapılmamalı ---
        VERİTABANI_TRANSACTION_BAŞLAT;
        
        // Adım 3a: Sipariş durumunu güncelle
        islem1 = Veritabani.SiparisDurumunuGuncelle(yeniSiparis.id, "Hazırlanıyor");
        
        // Adım 3b: Stok adetlerini düşür
        islem2 = Veritabani.StokAdetleriniDusur(aktifSepet.urunListesi);
        
        // Adım 3c: İndirim kodunu geçersiz kıl
        islem3 = Veritabani.IndirimKodunuGecersizKil(aktifSepet.indirimKodu);
        
        EĞER islem1 VE islem2 VE islem3 İSE
            // Tüm adımlar başarılı, değişiklikleri kalıcı yap
            VERİTABANI_TRANSACTION_ONAYLA (COMMIT);
            
            // Adım 4: Müşteriye bildirim gönder
            EpostaServisi.OnayEpostasiGonder(kullanici.email, yeniSiparis.id);
            
            // Adım 5: Kullanıcının sepetini temizle
            Veritabani.KullanicininSepetiniTemizle(kullanici.id);
            
            SONUC_MESAJI = "Başarılı: Siparişiniz alındı.";
        DEĞİLSE
            // Adımlardan en az biri başarısız oldu, tüm değişiklikleri geri al
            VERİTABANI_TRANSACTION_GERI_AL (ROLLBACK);
            HataLogla("KRİTİK HATA: Sipariş onayı veritabanı hatası. Sipariş ID: " + yeniSiparis.id);
            // Otomatik para iadesi sürecini başlat
            OdemeAgGecidi.ParaIadesiYap(yeniSiparis.id);
            SONUC_MESAJI = "Hata: İşlem sırasında teknik bir sorun oluştu. Ücret iadeniz yapılacaktır.";
        BİTİR EĞER
        // --- KRİTİK BÖLGE SONU ---
        
    DEĞİLSE
        // Ödeme işlemi başarısız oldu
        Veritabani.SiparisDurumunuGuncelle(yeniSiparis.id, "Ödeme Başarısız");
        SONUC_MESAJI = "Hata: Ödeme işlemi başarısız oldu. " + odemeYaniti.hataMesaji;
    BİTİR EĞER
    
    BİTİR: SİPARİŞİ TAMAMLAMA SÜRECİ
    DÖN SONUC_MESAJI;
BİTİR FONKSIYON
