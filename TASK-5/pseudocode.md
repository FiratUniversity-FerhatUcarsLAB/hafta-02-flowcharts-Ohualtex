// ===============================
// KONFİGÜRASYON
// ===============================
CONST ENTRY_DELAY_MS   = 20000     // giriş gecikmesi
CONST EXIT_DELAY_MS    = 30000     // çıkış gecikmesi
CONST ALARM_DURATION_MS= 300000    // siren süresi (5 dk)
CONST HEARTBEAT_MS     = 5000
CONST MAX_PIN_TRIES    = 5

STRUCT Zone {
    id: INT
    name: STRING
    type: ENUM{INSTANT, ENTRY_EXIT, INTERIOR, SMOKE, WATER, TAMPER}
    enabled_when_armed_away: BOOL
    enabled_when_armed_stay: BOOL
    sensor_read: FUNC -> BOOL // true: tetik
}

STRUCT Event {
    ts: TIME
    type: ENUM{ARM, DISARM, ALARM, SENSOR_TRIP, TAMPER, POWER_FAIL, NET_FAIL}
    detail: STRING
}

GLOBAL state = ENUM{DISARMED, EXIT_DELAY, ARMED_STAY, ARMED_AWAY, ENTRY_DELAY, ALARM}
GLOBAL pin_fail_count = 0
GLOBAL zones = ARRAY<Zone>
GLOBAL alarm_timer = 0
GLOBAL entry_timer = 0
GLOBAL exit_timer = 0
GLOBAL siren_on = FALSE

// Bildirim/kayıt arayüzleri
FUNC Log(e: Event) { persist(e) }
FUNC Notify(msg: STRING) { push_to_app(msg); if(net_down()) queue_for_retry(msg) }

// Donanım soyutlamaları
FUNC Siren(on: BOOL) { gpio_write(SIREN_PIN, on) }
FUNC Strobe(on: BOOL) { gpio_write(STROBE_PIN, on) }
FUNC LockAllDoors() { relay_all_locks(LOCK) }
FUNC UnlockAllDoors() { relay_all_locks(UNLOCK) }
FUNC CaptureClip(seconds: INT) { camera_record(seconds) }
FUNC Now() -> TIME { return system_time() }

// ===============================
// YARDIMCI FONKSİYONLAR
// ===============================
FUNC AnyTriggered(zset: ARRAY<Zone>) -> Zone|NULL {
    FOR z IN zset:
        IF z.sensor_read() == TRUE:
            RETURN z
    RETURN NULL
}

FUNC ActiveZonesFor(mode) -> ARRAY<Zone> {
    IF mode == ARMED_AWAY:
        RETURN [z FOR z IN zones WHERE z.enabled_when_armed_away]
    IF mode == ARMED_STAY:
        RETURN [z FOR z IN zones WHERE z.enabled_when_armed_stay]
    RETURN [] // disarmed
}

FUNC IsEntryExit(z: Zone) -> BOOL { return z.type == ENTRY_EXIT }
FUNC IsInstant(z: Zone) -> BOOL { return z.type == INSTANT }
FUNC IsLifeSafety(z: Zone) -> BOOL { return z.type IN {SMOKE, WATER} }
FUNC IsTamper(z: Zone) -> BOOL { return z.type == TAMPER }

// ===============================
// DURUM GEÇİŞLERİ
// ===============================
PROC ArmStay():
    IF state != DISARMED: RETURN
    exit_timer = EXIT_DELAY_MS
    state = EXIT_DELAY
    mode_after_exit = ARMED_STAY
    Notify("Çıkış gecikmesi başladı (Stay).")
    Log(Event{Now(), ARM, "stay"})

PROC ArmAway():
    IF state != DISARMED: RETURN
    exit_timer = EXIT_DELAY_MS
    state = EXIT_DELAY
    mode_after_exit = ARMED_AWAY
    Notify("Çıkış gecikmesi başladı (Away).")
    Log(Event{Now(), ARM, "away"})

PROC Disarm(pin: STRING):
    IF verify_pin(pin) == FALSE:
        pin_fail_count += 1
        IF pin_fail_count >= MAX_PIN_TRIES:
            Notify("PIN çok sayıda hatalı girildi.")
            Log(Event{Now(), TAMPER, "pin brute-force"})
        RETURN
    pin_fail_count = 0
    state = DISARMED
    alarm_timer = 0
    entry_timer = 0
    Siren(FALSE); Strobe(FALSE)
    Notify("Sistem devre dışı.")
    Log(Event{Now(), DISARM, ""})

PROC TriggerAlarm(reason: STRING):
    state = ALARM
    alarm_timer = ALARM_DURATION_MS
    Siren(TRUE); Strobe(TRUE)
    CaptureClip(30)
    LockAllDoors()
    Notify("ALARM: " + reason)
    Log(Event{Now(), ALARM, reason})

// ===============================
// ANA DÖNGÜ
// ===============================
LOOP forever:
    // 1) Yaşam güvenliği her zaman aktif
    FOR z IN zones:
        IF IsLifeSafety(z) AND z.sensor_read():
            TriggerAlarm("Yaşam güvenliği: " + z.name)

    SWITCH state:

        CASE DISARMED:
            // isteğe bağlı: kapı/çıkış açık uyarıları
            pass

        CASE EXIT_DELAY:
            IF exit_timer > 0:
                exit_timer -= tick()
            ELSE:
                state = mode_after_exit
                Notify("Sistem kuruldu: " + to_string(state))

        CASE ARMED_STAY:
            az = ActiveZonesFor(ARMED_STAY)
            t = AnyTriggered(az)
            IF t != NULL:
                IF IsEntryExit(t):
                    entry_timer = ENTRY_DELAY_MS
                    state = ENTRY_DELAY
                    Notify("Giriş gecikmesi başladı.")
                ELSE IF IsInstant(t) OR IsTamper(t):
                    TriggerAlarm("Bölge: " + t.name)

        CASE ARMED_AWAY:
            az = ActiveZonesFor(ARMED_AWAY)
            t = AnyTriggered(az)
            IF t != NULL:
                IF IsEntryExit(t):
                    entry_timer = ENTRY_DELAY_MS
                    state = ENTRY_DELAY
                    Notify("Giriş gecikmesi başladı.")
                ELSE:
                    TriggerAlarm("Bölge: " + t.name)

        CASE ENTRY_DELAY:
            IF entry_timer > 0:
                entry_timer -= tick()
                // Bu sırada panelden geçerli PIN gelirse Disarm çağrılır
            ELSE:
                TriggerAlarm("Giriş gecikmesi aşıldı.")

        CASE ALARM:
            IF alarm_timer > 0:
                alarm_timer -= tick()
            ELSE:
                // alarm süresi bitti, tekrar kurulu kalır
                Siren(FALSE); Strobe(FALSE)
                Notify("Alarm süresi doldu.")
                // tercihen kurulu mod korunur
                IF previous_mode IN {ARMED_STAY, ARMED_AWAY}:
                    state = previous_mode
                ELSE:
                    state = DISARMED

    // 2) Sağlık kontrolleri
    IF power_fail_detected():
        Notify("Güç kesintisi algılandı.")
        Log(Event{Now(), POWER_FAIL, ""})
    IF network_problem_detected():
        Notify("Ağ problemi algılandı.")
        Log(Event{Now(), NET_FAIL, ""})

    // 3) Heartbeat ve minimal bakım
    EVERY HEARTBEAT_MS:
        send_heartbeat_summary(state)
        rotate_logs_if_needed()

END LOOP

// ===============================
// OLAY TETİKÇİLERİ (ISR/EVENTS)
// ===============================
ON keypad_pin_entered(pin):
    IF state IN {ENTRY_DELAY, ARMED_STAY, ARMED_AWAY, ALARM}:
        Disarm(pin)
    ELSE:
        // hızlı kurulum kısayolları
        IF pin == quick_arm_stay_code(): ArmStay()
        IF pin == quick_arm_away_code(): ArmAway()

ON mobile_command(cmd):
    SWITCH cmd.type:
        CASE "ARM_STAY": ArmStay()
        CASE "ARM_AWAY": ArmAway()
        CASE "DISARM": Disarm(cmd.pin)
        CASE "BYPASS_ZONE": disable_zone(cmd.zone_id)
        CASE "LOCK_ALL": LockAllDoors()
        CASE "UNLOCK_ALL": UnlockAllDoors()

ON zone_tamper_detected(zid):
    TriggerAlarm("Müdahale: " + get_zone(zid).name)

// ===============================
// ZONE TANIM ÖRNEKLERİ
// ===============================
INIT zones = [
    Zone{1,"Giriş Kapısı", ENTRY_EXIT, true, true, read_contact_1},
    Zone{2,"Salon PIR",    INTERIOR,  true, false, read_pir_living},
    Zone{3,"Pencere-1",    INSTANT,   true, true,  read_contact_2},
    Zone{4,"Duman",        SMOKE,     true, true,  read_smoke},
    Zone{5,"Su Kaçağı",    WATER,     true, true,  read_water},
    Zone{6,"Panel Kapağı", TAMPER,    true, true,  read_tamper}
]

// ===============================
// İSTEĞE BAĞLI MODÜLLER (kısa)
// ===============================

// Akıllı sahne: alarm sırasında ışıkları yakıp flaşla
PROC AlarmLightingScene():
    IF state == ALARM:
        smart_lights_flash_all()

// Basit öğrenme: yanlış alarm yapan bölgeyi raporla
PROC FalseAlarmTracker(z: Zone, user_marked_false: BOOL):
    IF user_marked_false:
        increment_false_count(z)
        IF false_count(z) > THRESHOLD:
            Suggest("Bölge " + z.name + " hassasiyet ayarı düşürülsün.")

// Gece modu (Stay kurulu iken bazı sensörleri devre dışı)
SCHEDULE every 23:00 -> enable_night_profile()
SCHEDULE every 06:30 -> disable_night_profile()

