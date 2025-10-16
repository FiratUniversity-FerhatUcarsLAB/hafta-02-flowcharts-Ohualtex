// =========================
// VERİ MODELİ (Basit)
// =========================
STRUCT Patient(id, tcKimlik, ad, soyad, telefon, email, sifreHash)
STRUCT Doctor(id, ad, soyad, uzmanlik, calismaTakvimi)  // calismaTakvimi: [TimeSlot]
STRUCT TimeSlot(doctorId, baslangic, bitis, durum)      // durum: "MUSAIT" | "REZERVE" | "BLOKELI"
STRUCT Appointment(id, patientId, doctorId, baslangic, bitis, durum) // durum: "AKTIF" | "IPTAL" | "GERCEKLESTI"

GLOBAL Patients = MAP<tcKimlik, Patient>
GLOBAL Doctors  = MAP<id, Doctor>
GLOBAL Slots    = INDEX<doctorId, [TimeSlot]>
GLOBAL Appointments = MAP<id, Appointment>

// =========================
// YARDIMCILAR
// =========================
FUNCTION HashPassword(sifre): return HASH(sifre)
FUNCTION Now(): return CURRENT_DATETIME()
FUNCTION GenerateId(): return UNIQUE_ID()
FUNCTION Overlaps(aStart, aEnd, bStart, bEnd): 
    return (aStart < bEnd) AND (bStart < aEnd)

// =========================
// KAYIT / GİRİŞ
// =========================
FUNCTION RegisterPatient(tc, ad, soyad, tel, email, sifre):
    IF tc IN Patients: return ERROR("TC kayıtlı")
    p = Patient(GenerateId(), tc, ad, soyad, tel, email, HashPassword(sifre))
    Patients[tc] = p
    return OK(p.id)

FUNCTION Login(tc, sifre):
    IF tc NOT IN Patients: return ERROR("Kullanıcı yok")
    p = Patients[tc]
    IF HashPassword(sifre) != p.sifreHash: return ERROR("Şifre hatalı")
    return OK(p.id)  // token mantığı basit tutuldu

// =========================
// DOKTOR ARAMA / UYGUNLUK
// =========================
FUNCTION SearchDoctors(uzmanlik, tarihAraligi):
    result = []
    FOR EACH d IN Doctors:
        IF uzmanlik != NULL AND d.uzmanlik != uzmanlik: CONTINUE
        IF HasAvailability(d.id, tarihAraligi):
            APPEND result, d
    return result

FUNCTION HasAvailability(doctorId, tarihAraligi):
    FOR EACH slot IN Slots[doctorId]:
        IF slot.durum == "MUSAIT" AND
           slot.baslangic >= tarihAraligi.baslangic AND
           slot.bitis    <= tarihAraligi.bitis:
            return TRUE
    return FALSE

FUNCTION ListAvailableSlots(doctorId, tarihAraligi, slotSuresiDakika):
    result = []
    FOR EACH slot IN Slots[doctorId]:
        IF slot.durum != "MUSAIT": CONTINUE
        IF NOT( slot.baslangic >= tarihAraligi.baslangic AND slot.bitis <= tarihAraligi.bitis ): CONTINUE
        // İsteğe göre slotu parçala:
        t = slot.baslangic
        WHILE t + slotSuresiDakika <= slot.bitis:
            APPEND result, (t, t + slotSuresiDakika)
            t = t + slotSuresiDakika
    return result

// =========================
// RANDEVU OLUŞTURMA
// =========================
FUNCTION CreateAppointment(patientId, doctorId, baslangic, bitis):
    // 1) Hasta ve doktor var mı?
    IF NOT ExistsPatient(patientId): return ERROR("Hasta yok")
    IF NOT ExistsDoctor(doctorId): return ERROR("Doktor yok")

    // 2) Gelecek zaman mı?
    IF baslangic < Now(): return ERROR("Geçmiş zamana randevu olmaz")

    // 3) Doktor uygun mu?
    IF NOT IsDoctorAvailable(doctorId, baslangic, bitis): return ERROR("Doktor uygun değil")

    // 4) Hastanın çakışan randevusu var mı?
    IF HasOverlappingAppointment(patientId, baslangic, bitis): return ERROR("Hastanın çakışan randevusu var")

    // 5) Oluştur ve slotu rezerve et (atomik)
    BEGIN_TRANSACTION
        aId = GenerateId()
        Appointments[aId] = Appointment(aId, patientId, doctorId, baslangic, bitis, "AKTIF")
        MarkSlotAsReserved(doctorId, baslangic, bitis)  // ilgili slot aralığını "REZERVE" yap
    COMMIT

    SendNotification(patientId, "Randevunuz oluşturuldu", aId)
    return OK(aId)

FUNCTION IsDoctorAvailable(doctorId, baslangic, bitis):
    FOR EACH slot IN Slots[doctorId]:
        IF slot.durum == "MUSAIT" AND Overlaps(slot.baslangic, slot.bitis, baslangic, bitis) == TRUE:
            // Burada tam kapsama gerekliyse, Overlaps yerine tam içerim kontrolü yapılabilir.
            return TRUE
    return FALSE

FUNCTION HasOverlappingAppointment(patientId, baslangic, bitis):
    FOR EACH a IN Appointments:
        IF a.patientId == patientId AND a.durum == "AKTIF":
            IF Overlaps(a.baslangic, a.bitis, baslangic, bitis): return TRUE
    return FALSE

FUNCTION MarkSlotAsReserved(doctorId, baslangic, bitis):
    // Basit yaklaşım: uygun slotu bul, gerekirse böl
    FOR EACH slot IN Slots[doctorId]:
        IF slot.durum != "MUSAIT": CONTINUE
        IF baslangic >= slot.baslangic AND bitis <= slot.bitis:
            // slotu üçe böl: [baslangicÖncesi] [randevu] [sonrası]
            left  = TimeSlot(doctorId, slot.baslangic, baslangic, "MUSAIT")
            mid   = TimeSlot(doctorId, baslangic, bitis, "REZERVE")
            right = TimeSlot(doctorId, bitis, slot.bitis, "MUSAIT")
            REPLACE slot WITH (left if left süresi>0) + mid + (right if right süresi>0)
            return
    THROW ERROR("Uygun slot bulunamadı")

// =========================
// RANDEVU İPTAL / ERTELEME
// =========================
FUNCTION CancelAppointment(appointmentId, patientId):
    IF appointmentId NOT IN Appointments: return ERROR("Randevu yok")
    a = Appointments[appointmentId]
    IF a.patientId != patientId: return ERROR("Yetkisiz")
    IF a.durum != "AKTIF": return ERROR("İptal edilemez")
    IF (a.baslangic - Now()) < 2 HOURS: return ERROR("2 saatten az kala iptal yapılamaz") // örnek kural

    BEGIN_TRANSACTION
        a.durum = "IPTAL"
        FreeSlot(a.doctorId, a.baslangic, a.bitis)  // slotu yeniden "MUSAIT" birleştir
    COMMIT

    SendNotification(patientId, "Randevunuz iptal edildi", appointmentId)
    return OK()

FUNCTION RescheduleAppointment(appointmentId, patientId, yeniBaslangic, yeniBitis):
    // İptal + Yeni oluştur akışı ama atomik
    IF appointmentId NOT IN Appointments: return ERROR("Randevu yok")
    a = Appointments[appointmentId]
    IF a.patientId != patientId: return ERROR("Yetkisiz")
    IF a.durum != "AKTIF": return ERROR("Taşınamaz")

    BEGIN_TRANSACTION
        // Önce yeni zamanı tutabiliyor muyuz?
        IF NOT IsDoctorAvailable(a.doctorId, yeniBaslangic, yeniBitis):
            ROLLBACK
            return ERROR("Yeni zaman uygun değil")

        // Eskiyi serbest bırak
        a.durum = "IPTAL"
        FreeSlot(a.doctorId, a.baslangic, a.bitis)

        // Yeni randevu oluştur
        newId = CreateAppointment(a.patientId, a.doctorId, yeniBaslangic, yeniBitis)
        IF newId is ERROR:
            // geri al
            a.durum = "AKTIF"
            MarkSlotAsReserved(a.doctorId, a.baslangic, a.bitis)
            ROLLBACK
            return ERROR("Yeniden planlama başarısız")
    COMMIT

    SendNotification(patientId, "Randevunuz yeniden planlandı", newId)
    return OK(newId)

// =========================
// CHECK-IN / GERÇEKLEŞTİRME
// =========================
FUNCTION CheckIn(appointmentId):
    IF appointmentId NOT IN Appointments: return ERROR("Randevu yok")
    a = Appointments[appointmentId]
    IF a.durum != "AKTIF": return ERROR("Geçersiz durum")
    IF Now() < a.baslangic - 30 MIN: return ERROR("Erken check-in") // örnek kural
    MarkAsRealized(appointmentId)
    return OK()

FUNCTION MarkAsRealized(appointmentId):
    a = Appointments[appointmentId]
    a.durum = "GERCEKLESTI"

// =========================
// SLOT SERBEST BIRAKMA ve BİRLEŞTİRME
// =========================
FUNCTION FreeSlot(doctorId, baslangic, bitis):
    // (1) İlgili aralığı "MUSAIT" olarak ekle
    ADD Slots[doctorId], TimeSlot(doctorId, baslangic, bitis, "MUSAIT")
    // (2) Komşu ve çakışan MUSAIT slotları birleştir (merge)
    MergeAdjacentAvailableSlots(doctorId)

FUNCTION MergeAdjacentAvailableSlots(doctorId):
    list = FILTER Slots[doctorId] WHERE durum == "MUSAIT"
    SORT list BY baslangic
    merged = []
    FOR EACH s IN list:
        IF merged EMPTY: APPEND merged, s
        ELSE:
            last = LAST(merged)
            IF last.bitis >= s.baslangic:  // bitiş==başlangıç dahil
                last.bitis = MAX(last.bitis, s.bitis) // birleştir
            ELSE:
                APPEND merged, s
    // REPLACE doctor slots: MUSAIT'leri merged ile güncelle, REZERVE/BLOKELI aynen kalır
    RebuildSlots(doctorId, merged)

// =========================
// BİLDİRİMLER (Sade)
// =========================
FUNCTION SendNotification(patientId, mesaj, refId):
    // SMS/E-posta göndermek yerine sadece loglayalım
    LOG("NOTIFY to patient " + patientId + ": " + mesaj + " [ref=" + refId + "]")

// =========================
// KULLANICI AKIŞ ÖRNEĞİ
// =========================
FLOW Example():
    uid = RegisterPatient("11111111111", "Ada", "Lovelace", "5xx...", "ada@ex.com", "gizli").value
    loginId = Login("11111111111", "gizli").value
    ds = SearchDoctors("Kardiyoloji", {baslangic: "2025-11-01 00:00", bitis: "2025-11-07 23:59"})
    slots = ListAvailableSlots(ds[0].id, {baslangic: "2025-11-03 00:00", bitis: "2025-11-03 23:59"}, 30)
    apptId = CreateAppointment(loginId, ds[0].id, slots[0].start, slots[0].end).value
    // ... sonra iptal veya yeniden planlama yapılabilir

