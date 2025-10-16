### Akıllı Ev Güvenlik Sistemi - Etkileşim Senaryosu Pseudocode'u

```
//======================================================================
// 1. KATILIMCILARIN (BİLEŞENLERİN) TANIMLANMASI
//======================================================================
// Her bileşen, kendi sorumluluklarına sahip bir nesne (object) olarak düşünülmüştür.

OBJECT AnaSistem
    // Özellikler (Properties)
    PROPERTY durum: STRING // Olası değerler: "DEVRE DIŞI", "KURULU", "ALARM"
    PROPERTY bagli_sensorler: Sensorler_Modulu
    PROPERTY bagli_alarm: Alarm_Mekanizmasi
    
    // Metotlar (Functions)
    PROCEDURE Kur()
    PROCEDURE Devre_Disi_Birak()
    PROCEDURE Ana_Gozetim_Dongusu()
END OBJECT

OBJECT Sensorler_Modulu
    PROCEDURE Veri_Oku() // Tüm sensörlerin durumunu döner.
END OBJECT

OBJECT Alarm_Mekanizmasi
    PROCEDURE Tetikle(tetikleyen_bilgi: STRING)
    PROCEDURE Durdur()
    PROCEDURE Bildirim_Gonder(kullanici: Kullanici, mesaj: STRING)
END OBJECT

OBJECT Kullanici
    // Bu nesne, dış dünyadan gelen etkileşimleri temsil eder.
END OBJECT


//======================================================================
// 2. ANA PROGRAM AKIŞI (SENARYONUN BAŞLANGICI)
//======================================================================
PROCEDURE Main()
    // Sistem bileşenlerini oluştur ve birbirine bağla.
    alarm_sistemi = NEW Alarm_Mekanizmasi()
    sensor_sistemi = NEW Sensorler_Modulu()
    ev_sistemi = NEW AnaSistem(bagli_sensorler: sensor_sistemi, bagli_alarm: alarm_sistemi)
    
    // --- Senaryo Başlangıcı ---
    // Adım 1: Kullanıcı, sistemi kurma komutunu gönderir.
    ev_sistemi.Kur()
    
    // Not: ev_sistemi.Kur() metodu, kendi içinde sonsuz bir döngü başlatır.
    // Program, kullanıcı müdahale edene kadar bu döngüde kalır.
    
END PROCEDURE


//======================================================================
// 3. ANA SİSTEMİN DETAYLI İŞLEYİŞİ
//======================================================================
PROCEDURE AnaSistem.Kur()
    self.durum = "KURULU"
    DISPLAY "Sistem Kuruldu. Gözetim döngüsü başlıyor."
    
    // Sonsuz gözetim döngüsünü başlat.
    CALL self.Ana_Gozetim_Dongusu()
END PROCEDURE

PROCEDURE AnaSistem.Ana_Gozetim_Dongusu()
    // Bu döngü, sistem "KURULU" olduğu sürece sürekli tekrar eder.
    WHILE self.durum == "KURULU" DO
    
        // Adım 2: Sistem, sensör modülünden anlık veri okumasını talep eder.
        rapor = self.bagli_sensorler.Veri_Oku()
        
        // Gelen rapora göre karar verilir.
        IF rapor == "TEHDİT_ALGILANDI" THEN
            // --- Alternatif Akış: Tehdit Durumu ---
            
            // Adım 4a: Sistem, bir tehdit olduğunu anlar.
            self.durum = "ALARM" // Sistemin durumunu değiştir.
            
            // Adım 4b: Sistem, alarm mekanizmasına alarmı tetiklemesini söyler.
            self.bagli_alarm.Tetikle("Hareket Sensörü") // Örnek bilgi
            
            // Alarm durumunda, kullanıcı müdahale edene kadar bu döngüden çıkar.
            BREAK WHILE
            
        ELSE // rapor == "NORMAL" ise
            // --- Normal Akış ---
            // Adım 3: Sistem, durumun normal olduğunu görür ve döngüye devam eder.
            // Hiçbir şey yapma, sadece bekleyerek bir sonraki kontrolü bekle.
        END IF
        
        WAIT(1 saniye) // Sistemi yormamak için kısa bir bekleme.
        
    END WHILE
END PROCEDURE

PROCEDURE AnaSistem.Devre_Disi_Birak()
    self.durum = "DEVRE DIŞI"
    DISPLAY "Sistem Devre Dışı Bırakıldı."
    
    // Adım 6: Alarm çalıyor olabilir, durdurma komutu gönder.
    self.bagli_alarm.Durdur()
END PROCEDURE


//======================================================================
// 4. DİĞER BİLEŞENLERİN İŞLEYİŞİ
//======================================================================
FUNCTION Sensorler_Modulu.Veri_Oku()
    // Gerçek bir sistemde bu fonksiyon, donanımdaki tüm sensörleri
    // (kapı, pencere, hareket vb.) kontrol eder.
    
    IF Herhangi_Bir_Sensor_Tehdit_Algiladi() THEN
        RETURN "TEHDİT_ALGILANDI"
    ELSE
        RETURN "NORMAL"
    END IF
END FUNCTION

PROCEDURE Alarm_Mekanizmasi.Tetikle(tetikleyen_bilgi: STRING)
    DISPLAY "ALARM! ALARM! Sirenler çalıyor!"
    
    // Adım 4c: Kullanıcıya asenkron bir bildirim gönder.
    mesaj = "KRİTİK UYARI: " + tetikleyen_bilgi + " bir tehdit algıladı!"
    CALL self.Bildirim_Gonder(Kullanici, mesaj)
END PROCEDURE

PROCEDURE Alarm_Mekanizmasi.Durdur()
    DISPLAY "Sirenler durduruldu."
END PROCEDURE
```
