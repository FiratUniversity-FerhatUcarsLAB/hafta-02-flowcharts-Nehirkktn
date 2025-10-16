### Üniversite Ders Kayıt Sistemi - Detaylı Pseudocode

```
//======================================================================
// SİSTEM BAŞLANGICI
//======================================================================
PROCEDURE Baslat_Ders_Kayit_Sistemi()
    
    // Adım 1: Öğrenci kimlik doğrulaması yapılır.
    aktifOgrenci = Ogrenci_Giris_Yap()

    // EĞER giriş başarılı ise (öğrenci bilgileri dolu ise)
    IF aktifOgrenci IS NOT NULL THEN
        // Adım 2: Ders seçimi ve kontrol işlemleri başlar.
        CALL Ders_Secim_Islemleri(aktifOgrenci)
    ELSE
        DISPLAY "HATA: Kullanıcı adı veya şifre hatalı. Sistemden çıkılıyor."
    END IF

END PROCEDURE

//======================================================================
// ÖĞRENCİ GİRİŞ FONKSİYONU
//======================================================================
FUNCTION Ogrenci_Giris_Yap()
    DISPLAY "Öğrenci Numaranızı Girin:"
    ogrenciNo = GET_INPUT()
    
    DISPLAY "Şifrenizi Girin:"
    sifre = GET_INPUT()

    // Veritabanından kullanıcıyı doğrula
    ogrenciBilgileri = Veritabani.Kullanici_Dogrula(ogrenciNo, sifre)
    
    // EĞER kullanıcı varsa, öğrencinin transkripti ve kredi limiti gibi
    // gerekli tüm bilgilerini de yükle.
    IF ogrenciBilgileri IS NOT NULL THEN
        ogrenciBilgileri.transkript = Veritabani.Ogrenci_Transkriptini_Getir(ogrenciNo)
        ogrenciBilgileri.krediLimiti = Veritabani.Ogrenci_Kredi_Limitini_Getir(ogrenciNo)
        RETURN ogrenciBilgileri
    ELSE
        RETURN NULL
    END IF
END FUNCTION

//======================================================================
// ANA DERS SEÇİM İŞLEMLERİ
//======================================================================
PROCEDURE Ders_Secim_Islemleri(ogrenci)
    dersSepeti = YENİ LİSTE() // Öğrencinin seçtiği dersler için boş bir sepet oluştur.
    kayitTamamlandi = FALSE
    
    // Öğrenci "Kaydı Onayla" veya "Çıkış" seçene kadar DÖNGÜ devam eder.
    WHILE kayitTamamlandi IS FALSE DO
        // Kullanıcı arayüzünü göster
        DISPLAY "============================================="
        DISPLAY "Açılan Dersler:", Veritabani.Donem_Derslerini_Listele()
        DISPLAY "Ders Sepetiniz:", dersSepeti
        mevcutKredi = Hesapla_Toplam_Kredi(dersSepeti)
        DISPLAY "Toplam Kredi: " + mevcutKredi + " / " + ogrenci.krediLimiti
        DISPLAY "============================================="
        DISPLAY "1- Ders Ekle"
        DISPLAY "2- Ders Çıkar"
        DISPLAY "3- Kaydı Danışman Onayına Gönder"
        DISPLAY "4- Çıkış Yap"
        
        secim = GET_INPUT()
        
        CASE secim OF
            "1": // Ders Ekleme
                DISPLAY "Eklemek istediğiniz dersin kodunu girin:"
                dersKodu = GET_INPUT()
                secilenDers = Veritabani.Ders_Bilgisini_Getir(dersKodu)
                
                // EĞER ders sistemde mevcutsa kontrollere başla
                IF secilenDers IS NOT NULL THEN
                    
                    //----------- KONTROL 1: KONTENJAN KONTROLÜ -----------
                    IF secilenDers.mevcutKayit < secilenDers.kontenjan THEN
                        
                        //----------- KONTROL 2: ÖN KOŞUL KONTROLÜ -----------
                        onKosulSaglandi = On_Kosul_Kontrolu(ogrenci.transkript, secilenDers.onKosullar)
                        IF onKosulSaglandi IS TRUE THEN
                            
                            //----------- KONTROL 3: KREDİ LİMİTİ KONTROLÜ -----------
                            IF (mevcutKredi + secilenDers.kredi) <= ogrenci.krediLimiti THEN
                                
                                //----------- KONTROL 4: ZAMAN ÇAKIŞMASI KONTROLÜ -----------
                                cakismaVarMi = Zaman_Cakismasi_Kontrolu(dersSepeti, secilenDers)
                                IF cakismaVarMi IS FALSE THEN
                                    
                                    // Tüm kontroller başarılı. Dersi sepete ekle.
                                    dersSepeti.EKLE(secilenDers)
                                    DISPLAY "BAŞARILI: '" + secilenDers.ad + "' sepete eklendi."
                                
                                ELSE // Zaman çakışması varsa
                                    DISPLAY "HATA: Bu ders, sepetinizdeki başka bir dersle çakışıyor."
                                END IF
                                
                            ELSE // Kredi limiti aşılıyorsa
                                DISPLAY "HATA: Bu dersi eklerseniz kredi limitinizi aşıyorsunuz."
                            END IF
                            
                        ELSE // Ön koşul sağlanmıyorsa
                            DISPLAY "HATA: Bu dersin ön koşullarını sağlamıyorsunuz."
                        END IF
                        
                    ELSE // Kontenjan doluysa
                        DISPLAY "HATA: Bu dersin kontenjanı dolu."
                    END IF
                    
                ELSE // Ders bulunamadıysa
                    DISPLAY "HATA: Girdiğiniz kodla eşleşen bir ders bulunamadı."
                END IF
                BREAK // Case'den çık
                
            "2": // Ders Çıkarma
                DISPLAY "Çıkarmak istediğiniz dersin kodunu girin:"
                dersKodu = GET_INPUT()
                dersSepeti.ÇIKAR(WHERE kod = dersKodu)
                DISPLAY "Ders sepetten çıkarıldı (eğer sepetteyse)."
                BREAK
                
            "3": // Kaydı Onaya Gönder
                //----------- KONTROL 5: DANIŞMAN ONAYI SÜRECİNİ BAŞLATMA -----------
                IF dersSepeti.BOŞ_MU() IS FALSE THEN
                    Veritabani.Kaydi_Onaya_Gonder(ogrenci.ID, dersSepeti)
                    DISPLAY "BAŞARILI: Ders seçimleriniz danışman onayına gönderildi."
                    kayitTamamlandi = TRUE // Ana DÖNGÜYÜ sonlandır.
                ELSE
                    DISPLAY "HATA: Onaya göndermeden önce en az bir ders seçmelisiniz."
                END IF
                BREAK
                
            "4": // Çıkış
                DISPLAY "Değişiklikler kaydedilmeden çıkılacak. Emin misiniz? (E/H)"
                onay = GET_INPUT()
                IF onay == "E" THEN
                    kayitTamamlandi = TRUE // Ana DÖNGÜYÜ sonlandır.
                END IF
                BREAK

        END CASE
    END WHILE
    
    DISPLAY "Ders kayıt oturumu sonlandırıldı."

END PROCEDURE

//======================================================================
// YARDIMCI KONTROL FONKSİYONLARI
//======================================================================

// Ön koşul kontrolü için yardımcı fonksiyon
FUNCTION On_Kosul_Kontrolu(ogrenciTranskripti, gerekliOnKosullar)
    // DÖNGÜ: Dersin her bir ön koşulu için...
    FOR EACH onKosul KODU IN gerekliOnKosullar
        dersAlinmisMi = FALSE
        // İÇ DÖNGÜ: Öğrencinin transkriptindeki her bir ders için...
        FOR EACH tamamlananDers IN ogrenciTranskripti
            IF tamamlananDers.kod == onKosul KODU AND tamamlananDers.not >= GECER_NOT THEN
                dersAlinmisMi = TRUE
                BREAK // İç döngüden çık, sonraki ön koşula geç.
            END IF
        END FOR
        
        // EĞER bir tane bile ön koşul sağlanmıyorsa, işlemi sonlandır.
        IF dersAlinmisMi IS FALSE THEN
            RETURN FALSE // Ön koşul sağlanmadı.
        END IF
    END FOR
    
    RETURN TRUE // Tüm ön koşullar sağlandı.
END FUNCTION

// Zaman çakışması kontrolü için yardımcı fonksiyon
FUNCTION Zaman_Cakismasi_Kontrolu(mevcutSepet, yeniDers)
    // DÖNGÜ: Sepetteki her bir ders için...
    FOR EACH sepettekiDers IN mevcutSepet
        // İÇ DÖNGÜ: Sepetteki dersin her bir zaman dilimi için...
        FOR EACH zaman1 IN sepettekiDers.dersSaatleri
            // ÜÇÜNCÜ DÖNGÜ: Eklenen yeni dersin her bir zaman dilimi için...
            FOR EACH zaman2 IN yeniDers.dersSaatleri
                // EĞER günler aynıysa ve saatler kesişiyorsa
                IF zaman1.gun == zaman2.gun AND (zaman1.baslangic < zaman2.bitis AND zaman1.bitis > zaman2.baslangic) THEN
                    RETURN TRUE // Çakışma var.
                END IF
            END FOR
        END FOR
    END FOR
    
    RETURN FALSE // Hiçbir çakışma bulunamadı.
END FUNCTION

// Toplam krediyi hesaplamak için yardımcı fonksiyon
FUNCTION Hesapla_Toplam_Kredi(sepet)
    toplamKredi = 0
    FOR EACH ders IN sepet
        toplamKredi = toplamKredi + ders.kredi
    END FOR
    RETURN toplamKredi
END FUNCTION
```
