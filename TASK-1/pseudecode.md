Prosedür ATMParaCekme
    // 1. Kartı algılayana kadar bekle
    EkranaYaz("Lütfen kartınızı yerleştiriniz.")
    kartOkundu = KartBekle()
    Eğer kartOkundu == Yanlış ise
        EkranaYaz("Geçersiz kart. Kart iade ediliyor.")
        KartIadeEt()
        Dön // program sonlanır
    SonEğer

    // 2. PIN doğrulama (en fazla 3 deneme)
    deneme = 0
    dogruPIN = False
    PINKodu = KarttanPINAl()  // kartın gerçek PIN'i
    Döngü deneme < 3 ve dogruPIN == False iken
        EkranaYaz("PIN kodunu giriniz:")
        girilenPIN = PINOku()
        Eğer girilenPIN == PINKodu ise
            dogruPIN = True
        Aksi
            deneme = deneme + 1
            Eğer deneme < 3 ise
                EkranaYaz("Yanlış PIN, tekrar deneyiniz.")
            SonEğer
        SonEğer
    DöngüSonu
    Eğer dogruPIN == False ise
        EkranaYaz("PIN üç kez hatalı. Kart bloke edildi.")
        KartBlokeEt()
        Dön
    SonEğer

    // 3. İşlem menüsü
    MenüYaz("1- Bakiye Sorgulama", "2- Para Çekme", "3- Çıkış")
    seçim = MenüSeç()

    Eğer seçim == 1 ise
        bakiye = HesapBakiyesiOku()
        EkranaYaz("Mevcut bakiye: " + bakiye)
        KartIadeEt()
        Dön
    SonEğer

    Eğer seçim == 2 ise
        // 4. Çekilecek tutarı al ve kontrol et
        tekrar:
        EkranaYaz("Çekmek istediğiniz tutarı giriniz:")
        tutar = TutarOku()

        Eğer tutar <= 0 ise
            EkranaYaz("Hatalı tutar, lütfen tekrar giriniz.")
            Git tekrar
        SonEğer
        Eğer (tutar mod 10) != 0 ise   // 10'un katı değilse mesaj ver:contentReference[oaicite:0]{index=0}
            EkranaYaz("Tutar 10'un katı olmalıdır.")
            Git tekrar
        SonEğer

        bakiye = HesapBakiyesiOku()
        Eğer tutar > bakiye ise
            EkranaYaz("Yetersiz bakiye.")
            Git tekrar
        SonEğer

        atmKasa = ATMBakiyesiOku()
        Eğer tutar > atmKasa ise
            EkranaYaz("ATM'de yeterli nakit yok.")
            Git tekrar
        SonEğer

        // 5. Para çekme ve güncelleme
        HesapBakiyesiGuncelle(bakiye - tutar)
        ATMBakiyesiGuncelle(atmKasa - tutar)
        NakitVer(tutar)
        FisBas(tutar, bakiye - tutar)  // istenirse fiş bas

        EkranaYaz("İşlem tamamlandı. Lütfen paranızı ve kartınızı alınız.")
        KartIadeEt()
        Dön
    SonEğer

    // 6. Çıkış seçilirse
    Eğer seçim == 3 ise
        KartIadeEt()
        Dön
    SonEğer

SonProsedür

