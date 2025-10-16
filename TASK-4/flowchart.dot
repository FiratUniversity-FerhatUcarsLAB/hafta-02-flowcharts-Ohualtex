digraph DersKayit {
    rankdir=TB;
    node [shape=rectangle, style=rounded];

    Start [label="Başla / Giriş Yap"];
    AuthFail [label="Giriş Hatalı", shape=oval];
    Engelli [label="Kayıt Engeli Var", shape=oval];
    Menu [label="İşlem Menüsü\n(EKLE / SİL / LİSTE / BİTİR)"];

    // Ekle Akışı
    Ekle [label="EKLE İşlemi"];
    Onkosul [label="Önkoşullar Tam mı?", shape=diamond];
    SaatCakis [label="Saat Çakışması Var mı?", shape=diamond];
    Kontenjan [label="Kontenjan Uygun mu?", shape=diamond];
    Kredi [label="Kredi Uygun mu?", shape=diamond];
    KayitOK [label="Kayıt Başarılı", shape=oval];
    Bekleme [label="Bekleme Listesi", shape=oval];
    KrediFail [label="Kredi Sınırı Aşıldı", shape=oval];

    // Sil Akışı
    Sil [label="SİL İşlemi"];
    SilOK [label="Ders Silindi", shape=oval];
    BekleyenYukselt [label="Bekleyenleri Yükselt"];

    // Liste Akışı
    Liste [label="LİSTE İşlemi"];
    Goster [label="Mevcut Dersler ve Krediler", shape=oval];

    // Bitir
    Bitir [label="BİTİR"];
    Odeme [label="Ödeme Gerekli mi?", shape=diamond];
    OdemeOK [label="Ödeme Başarılı", shape=oval];
    OdemeFail [label="Ödeme Başarısız", shape=oval];
    End [label="Kayıt Süreci Tamamlandı", shape=oval];

    // Akış
    Start -> Menu;
    Start -> AuthFail [label="giriş hatalı"];
    Start -> Engelli [label="engel var"];

    Menu -> Ekle [label="EKLE"];
    Menu -> Sil [label="SİL"];
    Menu -> Liste [label="LİSTE"];
    Menu -> Bitir [label="BİTİR"];

    Ekle -> Onkosul;
    Onkosul -> SaatCakis [label="EVET"];
    Onkosul -> Menu [label="HAYIR"];
    SaatCakis -> Kontenjan [label="HAYIR"];
    SaatCakis -> Menu [label="EVET"];
    Kontenjan -> Kredi [label="EVET"];
    Kontenjan -> Bekleme [label="HAYIR"];
    Kredi -> KayitOK [label="EVET"];
    Kredi -> KrediFail [label="HAYIR"];
    KayitOK -> Menu;
    Bekleme -> Menu;
    KrediFail -> Menu;

    Sil -> SilOK -> BekleyenYukselt -> Menu;

    Liste -> Goster -> Menu;

    Bitir -> Odeme;
    Odeme -> OdemeOK [label="EVET"];
    Odeme -> OdemeFail [label="HAYIR"];
    OdemeOK -> End;
    OdemeFail -> End;
}

