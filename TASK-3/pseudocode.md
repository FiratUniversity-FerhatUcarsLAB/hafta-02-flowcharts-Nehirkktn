### Hastane Bilgi Sistemi Akışı için Pseudocode

```
//======================================================================
// SİSTEM TANIMLAMALARI
//======================================================================
// Bu sistem, harici olarak kabul ettiğimiz iki servise bağımlıdır:
EXTERNAL_SERVICE Veritabani
EXTERNAL_SERVICE SmsServisi

//======================================================================
// ANA PROGRAM AKIŞI
//======================================================================
PROCEDURE Baslat_Hastane_Sistemi()
    // Kullanıcı sisteme giriş yapar.
    aktifKullanici = GetKullaniciBilgisi()

    // Ana menüyü göster ve kullanıcının seçimini al.
    CALL Goster_Ana_Menu(aktifKullanici)
END PROCEDURE

//======================================================================
// ANA MENÜ YÖNETİMİ
//======================================================================
FUNCTION Goster_Ana_Menu(kullanici)
    DISPLAY "Hoş geldiniz, " + kullanici.isim
    DISPLAY "Lütfen bir işlem seçiniz:"
    DISPLAY "1. Randevu Al"
    DISPLAY "2. Tahlil Sonuçlarını Görüntüle"

    secim = GetKullaniciSecimi()

    // Kullanıcının seçimine göre ilgili süreci başlat.
    CASE secim OF
        "1":
            CALL Yonet_Randevu_Sureci(kullanici)
        "2":
            CALL Yonet_Tahlil_Sonucu_Sureci(kullanici)
        OTHERWISE:
            DISPLAY "Hatalı seçim. Lütfen tekrar deneyin."
            CALL Goster_Ana_Menu(kullanici) // Menüyü tekrar göster.
    END CASE
END FUNCTION

//======================================================================
// MODÜL 1: Tahlil Sonucu Görüntüleme Süreci
//======================================================================
PROCEDURE Yonet_Tahlil_Sonucu_Sureci(kullanici)
    // Veritabanından kullanıcının tahlil sonuçlarını oku.
    tahlilSonuclari = CALL Veritabani.Oku("Tahliller", WHERE kullaniciID = kullanici.ID)

    // Sonucu ekranda göster.
    DISPLAY "Tahlil Sonuçlarınız:", tahlilSonuclari
    DISPLAY "SONUÇ: Tahlil Sonuçları Gösterildi."
    // Süreç biter.
END PROCEDURE

//======================================================================
// MODÜL 2: Randevu Alma Süreci (Detaylı Akış)
//======================================================================
PROCEDURE Yonet_Randevu_Sureci(kullanici)
    
    // Adım 1: Kimlik Doğrulama (Bu adımda kullanıcının ID'si zaten biliniyor)
    hastaID = kullanici.ID
    
    // Adım 2: Hastanın sistemde kayıtlı olup olmadığını kontrol et.
    hastaKayitliMi = CALL Veritabani.KayitVarMi("Hastalar", WHERE ID = hastaID)
    
    // Karar Anı: Hasta kayıtlı ise devam et, değilse hata ver.
    IF hastaKayitliMi IS TRUE THEN
        // --- BAŞARI YOLU ---
        
        // Adım 3 & 4: Poliklinik, doktor ve saat seçimi
        poliklinik = GET Secim("Lütfen poliklinik seçin:", Veritabani.GetList("Poliklinikler"))
        doktor = GET Secim("Lütfen doktor seçin:", Veritabani.GetList("Doktorlar", WHERE poliklinikID = poliklinik.ID))
        saat = GET Secim("Lütfen uygun bir saat seçin:", Veritabani.GetList("UygunSaatler", WHERE doktorID = doktor.ID))
        
        // Adım 5: Randevuyu oluştur ve veritabanına yaz.
        yeniRandevu = CREATE_OBJECT(hastaID, doktor.ID, saat)
        islemBasarili = CALL Veritabani.Yaz("Randevular", yeniRandevu)
        
        IF islemBasarili THEN
            // Adım 6: Onay SMS'i gönder.
            hastaTelefon = Veritabani.Oku("Hastalar.Telefon", WHERE ID = hastaID)
            mesaj = "Sayın " + kullanici.isim + ", " + doktor.isim + " için randevunuz başarıyla oluşturulmuştur."
            CALL SmsServisi.Gonder(hastaTelefon, mesaj)
            
            // Sonuç: Başarı mesajını göster.
            DISPLAY "SONUÇ: Randevu Başarıyla Alındı."
        ELSE
            DISPLAY "HATA: Randevu veritabanına kaydedilirken bir sorun oluştu."
        ENDIF
        
    ELSE
        // --- HATA YOLU ---
        
        // Sonuç: Hata mesajını göster.
        DISPLAY "SONUÇ: Hasta Bulunamadı."
        
    ENDIF
    
    // Süreç biter.
END PROCEDURE
```
