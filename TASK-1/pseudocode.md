// ============================================================================
//                  ATM PARA ÇEKME SİSTEMİ - BİRLEŞİK SÖZDE KOD
// ============================================================================
// Bu kod, bir ATM para çekme işleminin tüm adımlarını tek bir akışta birleştirir.
// Yazılım Mühendisliği Notu: Gerçek bir uygulamada, bu fonksiyonlar farklı
// sınıflara (örn: AccountService, TransactionManager, HardwareController)
// ve katmanlara (örn: Service Layer, Data Access Layer) ayrılırdı.
// ============================================================================

// ----------------------------------------------------------------------------
// BÖLÜM 1: VERİ YAPILARI VE SABİTLER
// ----------------------------------------------------------------------------

// Bir müşteri hesabını temsil eden nesne yapısı
OBJECT Account:
    STRING accountNumber
    STRING pinHash      // PIN'in şifrelenmiş (hash'lenmiş) hali
    DECIMAL balance
    STRING status       // "Active", "Blocked" vb.
    DECIMAL dailyWithdrawalTotal

// Sistem genelinde kullanılacak sabit değerler
CONSTANT MAX_PIN_ATTEMPTS = 3
CONSTANT DAILY_WITHDRAWAL_LIMIT = 15000.00
CONSTANT MIN_BILL_DENOMINATION = 10 // ATM'nin verebildiği en küçük banknot

// ----------------------------------------------------------------------------
// BÖLÜM 2: YARDIMCI FONKSİYONLAR
// ----------------------------------------------------------------------------

// --- PIN Doğrulama Fonksiyonu ---
// Kullanıcının girdiği PIN'i 3 deneme hakkı tanıyarak doğrular.
// Başarısız denemeler sonucunda hesabı bloke eder.
FUNCTION VerifyUserPin(account):
    pinAttempts = 0

    WHILE pinAttempts < MAX_PIN_ATTEMPTS:
        enteredPin = GetPinFromUserInput()

        IF Hash(enteredPin) == account.pinHash:
            RETURN TRUE // PIN doğru, doğrulama başarılı
        ELSE:
            pinAttempts = pinAttempts + 1
            remainingAttempts = MAX_PIN_ATTEMPTS - pinAttempts
            
            IF remainingAttempts > 0:
                DisplayMessage("Hata Mesajı: 'Hatalı PIN. Kalan hak: " + remainingAttempts + "'")
            END IF
        END IF
    END WHILE

    // Döngüden çıkıldıysa 3 deneme de başarısız olmuştur.
    BlockAccountInDatabase(account)
    DisplayMessage("Hata Mesajı: 'Kartınız bloke edilmiştir'")
    RETURN FALSE // PIN doğrulanamadı, işlem başarısız

END FUNCTION


// --- Tutar Kontrol Fonksiyonu ---
// Çekilmek istenen tutarın iş kurallarına (pozitif olması, banknot katı olması,
// günlük limiti aşmaması) uygunluğunu kontrol eder.
FUNCTION IsAmountValid(amount, account):
    // Kontrol 1: Tutar pozitif mi?
    IF amount <= 0:
        DisplayMessage("Hata Mesajı: 'Geçersiz bir tutar girdiniz.'")
        RETURN FALSE
    END IF

    // Kontrol 2: Tutar banknot katı mı?
    IF amount MOD MIN_BILL_DENOMINATION != 0:
        DisplayMessage("Hata Mesajı: 'Yalnızca " + MIN_BILL_DENOMINATION + " TL ve katları tutarında işlem yapabilirsiniz.'")
        RETURN FALSE
    END IF

    // Kontrol 3: Günlük limiti aşıyor mu?
    IF (account.dailyWithdrawalTotal + amount) > DAILY_WITHDRAWAL_LIMIT:
        DisplayMessage("Hata Mesajı: 'Bu işlemle günlük para çekme limitinizi aşıyorsunuz.'")
        RETURN FALSE
    END IF

    // Tüm kontroller başarılı
    RETURN TRUE

END FUNCTION

// ----------------------------------------------------------------------------
// BÖLÜM 3: ANA İŞ AKIŞI FONKSİYONU
// ----------------------------------------------------------------------------

// ATM'ye kart takıldığında tetiklenen ana süreç.
FUNCTION ExecuteWithdrawalProcess(cardData):
    // 1. Hesap Bilgilerini Al ve Kontrol Et
    account = GetAccountFromDatabase(cardData.accountNumber)

    IF account IS NULL OR account.status != "Active":
        DisplayMessage("Hata Mesajı: 'Kart geçersiz veya kullanıma kapalı.'")
        EjectCard()
        RETURN // İşlemi sonlandır
    END IF

    // 2. PIN Doğrulama Sürecini Başlat
    isPinVerified = VerifyUserPin(account)

    IF isPinVerified IS FALSE:
        // Gerekli mesajlar (kart bloke edildi vb.) VerifyUserPin fonksiyonu içinde zaten gösterildi.
        EjectCard()
        RETURN // İşlemi sonlandır
    END IF

    // 3. Kullanıcıdan Çekilecek Tutarı Al
    requestedAmount = GetAmountFromUserInput()

    // 4. Tutarın Geçerliliğini Kontrol Et
    isAmountOk = IsAmountValid(requestedAmount, account)

    IF isAmountOk IS FALSE:
        // Gerekli hata mesajı IsAmountValid fonksiyonu içinde zaten gösterildi.
        DisplayMainMenu()
        EjectCard()
        RETURN // İşlemi sonlandır
    END IF

    // 5. Bakiye Kontrolü
    IF requestedAmount > account.balance:
        DisplayMessage("Hata Mesajı: 'Yetersiz Bakiye.'")
        DisplayMainMenu()
        EjectCard()
        RETURN // İşlemi sonlandır
    END IF

    // 6. Para Çekme İşlemini Gerçekleştir (Transactional Block)
    BEGIN TRANSACTION
        // ÖNCE veritabanı güncellenir, SONRA para verilir.
        updateSuccess = UpdateAccountBalanceInDatabase(account, requestedAmount)

        IF updateSuccess IS TRUE:
            DispenseCash(requestedAmount)
            LogTransaction(account.accountNumber, "Withdrawal", requestedAmount)
            PrintReceipt()
            DisplayMessage("Mesaj: 'İşleminiz başarıyla tamamlandı. Lütfen paranızı alınız.'")
            COMMIT TRANSACTION // Değişiklikleri onayla ve kalıcı hale getir
        ELSE:
            DisplayMessage("Hata Mesajı: 'Teknik bir sorun nedeniyle işleminiz gerçekleştirilemedi.'")
            ROLLBACK TRANSACTION // Değişiklikleri geri al
        END IF
    END TRANSACTION

    EjectCard()
    DisplayWelcomeScreen()
    RETURN // İşlem bitti

END FUNCTION
