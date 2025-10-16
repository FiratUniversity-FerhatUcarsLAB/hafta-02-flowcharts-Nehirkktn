İsim - Soy isim Nehir KÖKTEN
Öğrenci No:250541039

sistemin kısa açıklaması (maks. 5-6 satır)
Bu sistem, kullanıcı kimliğini 3 deneme hakkı sunan güvenli PIN doğrulaması ile başlatır. Ardından, çekilmek istenen tutarı; miktar, banknot katları ve günlük limit gibi iş kurallarına göre doğrular ve son olarak hesap bakiyesini kontrol eder. Tüm kontroller başarılıysa, veri bütünlüğünü garanti altına alan bir 'transaction' bloğu içinde önce bakiyeyi günceller, sonra parayı fiziksel olarak verir. Bu atomik yaklaşım, para verilip bakiye güncellenmemesi gibi kritik hataları önler. Süreç, her adımda kullanıcıyı bilgilendirir ve kartı iade ederek güvenli bir şekilde sonlanır.
