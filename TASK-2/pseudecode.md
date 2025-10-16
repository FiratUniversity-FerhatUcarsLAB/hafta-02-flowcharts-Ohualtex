// --- Veri Yapıları ---

struct Urun {
    id
    ad
    birimFiyat
}

struct SepetKalemi {
    urunId
    ad
    birimFiyat
    adet
    araToplam   // birimFiyat * adet
}

struct Sepet {
    kalemler: liste<SepetKalemi>
    kuponKodu: string?  // opsiyonel
    araToplam
    indirimToplam
    kargoUcreti
    vergiToplam
    genelToplam
}

// --- Yardımcı Hesaplar ---

fonksiyon AraToplamHesapla(sepet):
    toplam = 0
    her kalem içinde sepet.kalemler:
        kalem.araToplam = kalem.birimFiyat * kalem.adet
        toplam += kalem.araToplam
    döndür toplam

fonksiyon IndirimHesapla(araToplam, kuponKodu):
    eğer kuponKodu boş ise döndür 0
    eğer kuponKodu geçerli değilse döndür 0
    // örnek kural: %10 indirim
    döndür araToplam * 0.10

fonksiyon KargoHesapla(araToplam):
    // örnek kural: 500 üzeri ücretsiz, aksi halde 49
    eğer araToplam >= 500 döndür 0
    aksi halde döndür 49

fonksiyon VergiHesapla(vergilendirilebilirTutar):
    // örnek KDV: %20
    döndür vergilendirilebilirTutar * 0.20

fonksiyon ToplamlariGuncelle(sepet):
    sepet.araToplam = AraToplamHesapla(sepet)
    sepet.indirimToplam = IndirimHesapla(sepet.araToplam, sepet.kuponKodu)
    vergilendirilebilirTutar = maksimum(0, sepet.araToplam - sepet.indirimToplam)
    sepet.kargoUcreti = KargoHesapla(vergilendirilebilirTutar)
    sepet.vergiToplam = VergiHesapla(vergilendirilebilirTutar)
    sepet.genelToplam = vergilendirilebilirTutar + sepet.kargoUcreti + sepet.vergiToplam
    döndür sepet

// --- Sepet İşlevleri ---

fonksiyon SepetOlustur():
    sepet = yeni Sepet
    sepet.kalemler = boş liste
    sepet.kuponKodu = boş
    sepet.araToplam = 0
    sepet.indirimToplam = 0
    sepet.kargoUcreti = 0
    sepet.vergiToplam = 0
    sepet.genelToplam = 0
    döndür sepet

fonksiyon KalemBul(sepet, urunId):
    her kalem içinde sepet.kalemler:
        eğer kalem.urunId == urunId döndür kalem
    döndür null

fonksiyon SepeteEkle(sepet, urun, adet):
    eğer adet <= 0 döndür sepet
    kalem = KalemBul(sepet, urun.id)
    eğer kalem != null:
        kalem.adet += adet
    aksi:
        yeniKalem = yeni SepetKalemi
        yeniKalem.urunId = urun.id
        yeniKalem.ad = urun.ad
        yeniKalem.birimFiyat = urun.birimFiyat
        yeniKalem.adet = adet
        ekle sepet.kalemler içine yeniKalem
    ToplamlariGuncelle(sepet)
    döndür sepet

fonksiyon AdetGuncelle(sepet, urunId, yeniAdet):
    kalem = KalemBul(sepet, urunId)
    eğer kalem == null döndür sepet
    eğer yeniAdet <= 0:
        kaldır sepet.kalemler içinden kalem
    aksi:
        kalem.adet = yeniAdet
    ToplamlariGuncelle(sepet)
    döndür sepet

fonksiyon SepettenCikar(sepet, urunId):
    kalem = KalemBul(sepet, urunId)
    eğer kalem != null:
        kaldır sepet.kalemler içinden kalem
    ToplamlariGuncelle(sepet)
    döndür sepet

fonksiyon KuponUygula(sepet, kuponKodu):
    sepet.kuponKodu = kuponKodu
    ToplamlariGuncelle(sepet)
    döndür sepet

fonksiyon SepetiGoruntule(sepet):
    yaz "---- Sepet ----"
    eğer sepet.kalemler boş ise:
        yaz "Sepetiniz boş."
        çık
    her kalem içinde sepet.kalemler:
        yaz kalem.ad, " x ", kalem.adet, " = ", kalem.araToplam
    yaz "Ara Toplam: ", sepet.araToplam
    yaz "İndirim: ", sepet.indirimToplam
    yaz "Kargo: ", sepet.kargoUcreti
    yaz "Vergi: ", sepet.vergiToplam
    yaz "Genel Toplam: ", sepet.genelToplam

// --- Ödeme / Checkout (sade akış) ---

fonksiyon OdemeBaslat(sepet, odemeBilgileri, teslimatAdresi):
    eğer sepet.kalemler boş ise:
        yaz "Sepet boş, ödeme başlatılamaz."
        döndür "HATA"
    // Stok doğrulama (basit)
    eğer StokYetersiz(sepet.kalemler):
        yaz "Bazı ürünler stokta yok."
        döndür "HATA"
    // Tutarları tekrar hesapla ve kilitle
    ToplamlariGuncelle(sepet)
    // Ödeme sağlayıcıya yönlendir
    odemeSonucu = OdemeSaglayiciIsle(odemeBilgileri, sepet.genelToplam)
    eğer odemeSonucu == "BASARILI":
        SiparisOlustur(sepet, teslimatAdresi)
        SepetiTemizle(sepet)
        yaz "Sipariş alındı."
        döndür "BASARILI"
    aksi:
        yaz "Ödeme başarısız."
        döndür "HATA"

fonksiyon SepetiTemizle(sepet):
    sepet.kalemler = boş liste
    sepet.kuponKodu = boş
    ToplamlariGuncelle(sepet)

// --- Örnek Kullanım ---

sepet = SepetOlustur()
urunA = Urun{id:1, ad:"Kulaklık", birimFiyat:350}
urunB = Urun{id:2, ad:"Klavye", birimFiyat:650}

SepeteEkle(sepet, urunA, 1)
SepeteEkle(sepet, urunB, 1)
KuponUygula(sepet, "INDIRIM10")
SepetiGoruntule(sepet)
OdemeBaslat(sepet, odemeBilgileri, teslimatAdresi)

