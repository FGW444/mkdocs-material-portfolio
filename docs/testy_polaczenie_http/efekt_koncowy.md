## Projekt: Detekcja ruchu i zdalna telemetria Android + Termux + Flask + Bearer Token

## Cel projektu

Mamy dwa telefony z Androidem w tej samej sieci lokalnej (lub spięte bezpiecznym tunelem VPN, np. Tailscale):

1. **TELEFON A — telefon z kamerą, sensorami i elementami wykonawczymi**
    * Działa na nim Termux z rozszerzeniami `Termux:API` oraz uprawnieniami do pamięci (`termux-setup-storage`).
    * Uruchamia serwer Flask sterujący procesem detekcji, obsługujący zdalną diagnostykę oraz urządzenia wykonawcze.
    * Robi zdjęcia tylną kamerą, analizuje ruch, wykonuje serię zdjęć alarmowych i składa je w plik MP4 za pomocą `ffmpeg`.
    * Zarządza fizycznymi zasobami sprzętowymi urządzenia: latarka, silnik wibracyjny, mikrofon oraz moduł lokalizacji GPS.
    * Zapisuje zdarzenia w logu tekstowym `motion.log` i przesyła pliki na Telefon B.

2. **TELEFON B — telefon sterujący / odbiornik plików**
    * Działa na nim Termux z serwerem Flask.
    * Odbiera pliki z Telefonu A i zapisuje je w katalogu `Downloads/motion_files`.
    * Automatycznie czyści stare pliki (starsze niż 7 dni) za pomocą skryptu cron, chroniąc pamięć urządzenia przed przepełnieniem.

## Zabezpieczenie komunikacji

Komunikacja HTTP jest bezwzględnie zabezpieczona tokenem typu Bearer Token na obu urządzeniach.

Przykład nagłówka:
```http
Authorization: Bearer MOJ_SUPER_TAJNY_TOKEN_123456
```

!!! warning "Ważne uwagi dotyczące bezpieczeństwa"
    * Ruch HTTP w lokalnej sieci Wi-Fi jest jawny. W przypadku komunikacji przez internet należy bezwzględnie kierować ruch przez szyfrowany tunel, np. **Tailscale** (wykorzystując adresy `100.x.y.z`).
    * Token chroni przed przypadkowym wywołaniem endpointów przez inne urządzenia w sieci.

## Architektura systemu

```text
HTTP Shortcuts / przeglądarka / curl / skrypty zewnętrzne


                          |
                          |  Bearer Token (GET / POST)
                          v
+-----------------------------------------------------------------+

| TELEFON A (Kamera + Urządzenie Wykonawcze):                     |
| motion_camera_server.py                                         |
|                                                                 |
| API Telemetrii i Kontroli Sprzętowej:                           |
|  -> GET  /status          (Stan systemu, bateria, pamięć)       |
|  -> GET  /location        (Koordynaty GPS z Termux:API)         |
|  -> POST /vibrate         (Wywołanie wibracji, parametr ?ms=)   |
|  -> POST /torch           (Włączenie/wyłączenie latarki)         |
|  -> GET  /voice           (Nagranie z mikrofonu, parametr ?sec=)|
|                                                                 |
| Logika detekcji ruchu w tle:                                    |
|  -> Analiza klatek -> Wykrycie -> Zdjęcia -> ffmpeg (MP4)       |
+-----------------------------------------------------------------+

                          |
                          |  Bearer Token (POST /upload)
                          v
+-----------------------------------------------------------------+

| TELEFON B (Odbiornik + Koordynator):                            |
| receiver_server.py <--- Skrypt czyszczący: clean_motion.sh       |
|                                                                 |
| Endpoints:                                                      |
|  -> GET  /status          (Weryfikacja dostępności odbiornika)   |
|  -> POST /upload          (Zapis przysyłanych plików MP4)       |
|                                                                 |
| Pamięć telefonu: Downloads/motion_files                         |
+-----------------------------------------------------------------+
```

---

## Część 1 — Instalacja na Telefonie A (Kamera)

### 1. Zainstaluj aplikacje
* **Termux** (zalecana wersja z F-Droid)
* **Termux:API** (zalecana wersja z F-Droid)

### 2. Konfiguracja środowiska Termux
Uruchom Termux na **Telefonie A** i wykonaj poniższe polecenia:

```bash
# Aktualizacja bazy pakietów Termux
pkg update && pkg upgrade -y

# Dodanie repozytorium X11 (wymagane dla pakietu OpenCV)
pkg install x11-repo -y

# Instalacja gotowego, skompilowanego pakietu OpenCV dla Androida
pkg install opencv-python -y

# Rozwiązanie błędów dependencji OpenCV
pkg install libavif dbus -y

# Instalacja narzędzi systemowych Termux:API, ffmpeg oraz jq do obsługi JSON
pkg install termux-api ffmpeg jq -y
pip install flask requests

# Przyznanie Termuxowi dostępu do pamięci wewnętrznej telefonu
termux-setup-storage
```

!!! note "Ważna uwaga dotycząca uprawnień Androida"
    Wejdź w systemowe ustawienia aplikacji systemu Android i upewnij się, że **Termux** oraz **Termux:API** posiadają uprawnienia do: **Aparatu, Mikrofonu, Lokalizacji oraz Pamięci**.

#### Weryfikacja poprawności instalacji OpenCV:
```bash
python -c "import cv2; print('OpenCV działa poprawnie!')"
```

---

## Część 2 — Instalacja na Telefonie B (Odbiornik)

### 1. Konfiguracja środowiska Termux
Wykonaj poniższe polecenia w konsoli Telefonu B:
```bash
pkg update && pkg upgrade -y
pkg install python cronie termux-services -y
pip install flask
termux-setup-storage
```

### 2. Sprawdź IP telefonu B
```bash
ip addr show wlan0
```
*(Jeśli używasz Tailscale, pobierz IP za pomocą komendy `tailscale ip -4`)*.

---

## Część 3 — Wybór tokenu

Ustal jeden wspólny token autoryzacyjny dla obu urządzeń, np.:
`MOJ_SUPER_TAJNY_TOKEN_123456`

---

## Część 4 — Kod dla Telefonu B: `receiver_server.py`

### Utwórz plik
```bash
nano receiver_server.py
```

### Kod źródłowy
```python
from flask import Flask, request, jsonify, abort
from pathlib import Path
from werkzeug.utils import secure_filename
from datetime import datetime
from functools import wraps

app = Flask(__name__)

# ==========================================================
# KONFIGURACJA
# ==========================================================
API_TOKEN = "MOJ_SUPER_TAJNY_TOKEN_123456"

# Ścieżka do zapisu plików w pamięci współdzielonej telefonu Android
SAVE_DIR = Path.home() / "storage" / "downloads" / "motion_files"
SAVE_DIR.mkdir(parents=True, exist_ok=True)


# ==========================================================
# AUTORYZACJA BEARER TOKEN
# ==========================================================
def require_bearer_token(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        auth_header = request.headers.get("Authorization", "")
        expected_header = f"Bearer {API_TOKEN}"

        if auth_header != expected_header:
            return jsonify({
                "ok": False,
                "error": "Brak autoryzacji lub błędny Bearer Token"
            }), 401

        return func(*args, **kwargs)
    return wrapper


# ==========================================================
# ENDPOINTY
# ==========================================================
@app.route("/status", methods=["GET"])
@require_bearer_token
def status():
    return jsonify({
        "ok": True,
        "message": "Receiver działa poprawnie",
        "save_dir": str(SAVE_DIR),
        "timestamp": datetime.now().isoformat()
    })


@app.route("/upload", methods=["POST"])
@require_bearer_token
def upload():
    if "file" not in request.files:
        return jsonify({"ok": False, "error": "Brak pola file"}), 400

    uploaded_file = request.files["file"]

    if uploaded_file.filename == "":
        return jsonify({"ok": False, "error": "Pusta nazwa pliku"}), 400

    safe_name = secure_filename(uploaded_file.filename)
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")

    final_name = f"{timestamp}_{safe_name}"
    final_path = SAVE_DIR / final_name

    uploaded_file.save(final_path)

    return jsonify({
        "ok": True,
        "saved_as": str(final_path)
    })


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=7000)
```

---

## Część 5 — Kod dla Telefonu A: `motion_camera_server.py`

### Utwórz plik
```bash
nano motion_camera_server.py
```

### Kod źródłowy
```python
import time
import threading
import subprocess
import logging
import json
import os
from pathlib import Path
from datetime import datetime
from functools import wraps
import cv2
import requests
from flask import Flask, jsonify, request, send_file

app = Flask(__name__)

# ==========================================================
# KONFIGURACJA SYSTEMOWA
# ==========================================================
API_TOKEN = "MOJ_SUPER_TAJNY_TOKEN_123456"
RECEIVER_URL = "http://192.168.8"  # Podmień na IP Telefonu B (lub IP Tailscale)
CAMERA_ID = "0"                                    # ID kamery z termux-camera-info

# Parametry analizy i detekcji ruchu
CHECK_INTERVAL_SECONDS = 2
ANALYSIS_WIDTH = 320
MOTION_THRESHOLD_PIXELS = 2500

# Parametry zapisu po wykryciu zdarzenia
BURST_PHOTOS = 4
RECORD_SECONDS = 15
RECORD_FPS = 1

# Struktura katalogów roboczych w Termux
BASE_DIR = Path.home() / "storage" / "Downloads" / "motion_camera"
LOW_DIR = BASE_DIR / "low"
EVENT_DIR = BASE_DIR / "events"
TEMP_DIR = Path.home() / "storage" / "Downloads" / "motion_camera" / "tmp"

for directory in [LOW_DIR, EVENT_DIR, TEMP_DIR]:
    directory.mkdir(parents=True, exist_ok=True)

# ==========================================================
# KONFIGURACJA LOGOWANIA (motion.log)
# ==========================================================
LOG_FILE = BASE_DIR / "motion.log"
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    handlers=[
        logging.FileHandler(LOG_FILE, encoding='utf-8'),
        logging.StreamHandler()
    ]
)

# ==========================================================
# AUTORYZACJA BEARER TOKEN
# ==========================================================
def require_bearer_token(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        auth_header = request.headers.get("Authorization", "")
        expected_header = f"Bearer {API_TOKEN}"

        if auth_header != expected_header:
            logging.warning("Nieautoryzowana próba dostępu do endpointu.")
            return jsonify({
                "ok": False,
                "error": "Brak autoryzacji lub błędny Bearer Token"
            }), 401
        return func(*args, **kwargs)
    return wrapper


# ==========================================================
# POMOCNICZE FUNKCJE SYSTEMOWE (TERMUX:API)
# ==========================================================
def run_termux_command(args):
    """Bezpieczne wykonywanie poleceń powłoki systemowej Termux."""
    try:
        result = subprocess.run(args, capture_output=True, text=True, check=True)
        return True, result.stdout.strip()
    except subprocess.CalledProcessError as e:
        logging.error(f"Błąd polecenia Termux {args}: {e.stderr}")
        return False, e.stderr
    except Exception as e:
        logging.error(f"Wyjątek krytyczny przy uruchamianiu {args}: {str(e)}")
        return False, str(e)


# ==========================================================
# ENDPOINTY DIAGNOSTYCZNO-WYKONAWCZE
# ==========================================================
@app.route("/status", methods=["GET"])
@require_bearer_token
def status():
    """Zwraca zunifikowane parametry operacyjne telefonu-kamery (W tym stan baterii)."""
    status_data = {
        "ok": True,
        "device_role": "motion_camera",
        "timestamp": datetime.now().isoformat(),
        "battery": {"percentage": "UNKNOWN", "status": "UNKNOWN"},
        "storage": {"free_bytes": "UNKNOWN"}
    }
    
    # Pobieranie statusu baterii przez Termux API
    success, output = run_termux_command(["termux-battery-status"])
    if success:
        try:
            status_data["battery"] = json.loads(output)
        except Exception:
            pass

    # Pobieranie informacji o przestrzeni dyskowej
    try:
        usage = Path.home().stat()
        status_data["storage"]["free_bytes"] = getattr(usage, 'st_blocks', 'Dostępne')
    except Exception:
        pass

    return jsonify(status_data)


@app.route("/location", methods=["GET"])
@require_bearer_token
def location():
    """Pobiera aktualną pozycję GPS urządzenia przy użyciu Termux:API."""
    success, output = run_termux_command(["termux-location", "-p", "network", "-r", "once"])
    
    if not success:
        return jsonify({
            "ok": False, 
            "error": "Nie udało się pobrać lokalizacji", 
            "details": output
        }), 500
        
    try:
        location_json = json.loads(output)
        return jsonify({
            "ok": True,
            "location": location_json
        })
    except json.JSONDecodeError:
        return jsonify({
            "ok": False,
            "error": "Błąd dekodowania danych GPS z Termux:API",
            "raw_output": output
        }), 502


@app.route("/vibrate", methods=["POST"])
@require_bearer_token
def vibrate():
    """Wymusza wibracje urządzenia. Obsługuje query string (?ms=) oraz JSON body."""
    # Wsparcie dla parametru z query string (?ms=) lub JSON dla zachowania kompatybilności
    ms_param = request.args.get("ms") or (request.get_json(silent=True) or {}).get("duration") or "1000"
    
    try:
        duration_ms = int(ms_param)
        if duration_ms <= 0:
            raise ValueError
    except ValueError:
        return jsonify({"ok": False, "error": "Nieprawidłowy czas wibracji (musi być liczbą > 0)"}), 400

    success, output = run_termux_command(["termux-vibrate", "-d", str(duration_ms)])
    
    if success:
        logging.info(f"Wywołano wibracje sprzętowe: {duration_ms}ms")
        return jsonify({"ok": True, "message": f"Telefon wibrował przez {duration_ms}ms"})
    else:
        return jsonify({"ok": False, "error": "Błąd wywołania silnika wibracyjnego", "details": output}), 500


@app.route("/torch", methods=["POST"])
@require_bearer_token
def torch():
    """Zdalne sterowanie wbudowaną latarką LED."""
    data = request.get_json(silent=True) or {}
    action = data.get("action", "").lower()
    
    if action not in ["on", "off"]:
        return jsonify({"ok": False, "error": "Dozwolone akcje w polu 'action' to: 'on' lub 'off'"}), 400

    state = "on" if action == "on" else "off"
    success, output = run_termux_command(["termux-torch", state])
    
    if success:
        logging.info(f"Stan latarki zmieniony na: {state}")
        return jsonify({"ok": True, "message": f"Latarka została ustawiona w stan: {state}"})
    else:
        return jsonify({"ok": False, "error": f"Nie udało się zmienić stanu latarki na {state}", "details": output}), 500


@app.route("/voice", methods=["GET"])
@require_bearer_token
def voice():
    """Nagrywa dźwięk z mikrofonu przez określony czas i zwraca plik binarny M4A."""
    seconds_param = request.args.get("sec", "5")
    
    try:
        seconds = int(seconds_param)
        if seconds <= 0 or seconds > 60:  # Zabezpieczenie przed zbyt długim blokowaniem wątku
            raise ValueError
    except ValueError:
        return jsonify({"ok": False, "error": "Parametr 'sec' musi być liczbą całkowitą z przedziału 1-60"}), 400
        
    path = TEMP_DIR / "audio_record.m4a"
    
    # Czyszczenie starego nagrania jeśli istnieje
    if path.exists():
        try:
            path.unlink()
        except Exception:
            pass

    logging.info(f"Uruchomienie nagrywania dźwięku: {seconds}s")
    success, output = run_termux_command(["termux-microphone-record", "-l", str(seconds), "-f", str(path)])
    
    if not success:
        return jsonify({"ok": False, "error": "Nie udało się zainicjować mikrofonu", "details": output}), 500
        
    # Oczekiwanie na finalizację zapisu pliku przez system Android
    time.sleep(seconds + 1)
    
    if path.exists() and path.stat().st_size > 0:
        return send_file(str(path), mimetype='audio/mp4', as_attachment=True, download_name=f"record_{int(time.time())}.m4a")
    
    return jsonify({"ok": False, "error": "Plik audio nie został wygenerowany poprawnie"}), 500


# ==========================================================
# LOGIKA PRZETWARZANIA OBRAZU (Makieta procesów tła)
# ==========================================================
def handle_motion_event():
    """Funkcja odpowiedzialna za rejestrację zdarzenia ruchu oraz wysyłkę plików."""
    logging.info("Rozpoczęto procedurę przechwytywania zdarzenia ruchu...")


if __name__ == "__main__":
    logging.info("Uruchamianie zunifikowanego serwera detekcji ruchu na porcie 5000...")
    app.run(host="0.0.0.0", port=5000, debug=False)
```

**Uruchomienie skryptu w tle w celu ochrony przed ubiciem przez Androida:**
```bash
termux-wake-lock
python3 motion_camera_server.py &
```

---

## Co dokładnie zostało zmienione i dlaczego? (Uzasadnienie)

1. **Ujednolicenie struktury odpowiedzi JSON**: Druga dokumentacja zwracała dane chaotycznie (raz surowy string tekstowy, raz słownik `{"status": ...}`). Wszystkie endpointy zostały ustandaryzowane tak, aby zwracały słownik z kluczem `"ok": True/False` oraz odpowiednimi komunikatami, co ułatwia automatyczną parsowalność (np. w aplikacjach typu *HTTP Shortcuts* lub skryptach klienckich).
2. **Dodanie endpointu `/voice` (Nagrywanie mikrofonu)**: Funkcja ta została przeniesiona bezpośrednio do Telefonu A (Kamera). Zgodnie z logiką systemową, to telefon rejestrujący obraz powinien nagrywać otaczający go dźwięk. Zastosowano bezpieczne przesyłanie pliku za pomocą funkcji `send_file` prosto do strumienia odpowiedzi HTTP z odpowiednim nagłówkiem MIME (`audio/mp4`), dzięki czemu klient od razu pobiera gotowy plik `.m4a`.
3. **Zabezpieczenie przed blokowaniem wątków w `/voice`**: Wprowadzono `time.sleep(seconds + 1)` po wywołaniu procesu asynchronicznego Androida, aby zapobiec sytuacji, w której Flask próbuje odczytać plik, który system jeszcze fizycznie zapisuje na karcie pamięci. Dodatkowo ograniczono maksymalny czas nagrania do 60 sekund, by zapobiec zawieszeniu wątku serwera.
4. **Wielowariantowość parametru w `/vibrate`**: Druga dokumentacja oczekiwała czasu wibracji jako query parameter (`?ms=2000`), a pierwsza opierała się na formacie obiektów JSON (`{"duration": 500}`). Endpoint został zaprojektowany hybrydowo: najpierw szuka parametru w adresie URL, jeśli go nie ma – sprawdza body JSON, a w ostateczności przyjmuje bezpieczną wartość domyślną (`1000`).
5. **Wspólny dekorator autoryzacji `@require_bearer_token`**: W drugiej dokumentacji funkcja uwierzytelniania była realizowana poprzez ręczne wywoływanie `check_auth()` wewnątrz każdej funkcji, co skutkowało powtarzalnością kodu. Wszystkie nowe funkcje zostały wdrożone za pomocą dekoratora, co gwarantuje pełną, bezwyjątkową ochronę całego API.
6. **Zintegrowany katalog tymczasowy `TEMP_DIR`**: Zamiast tworzyć pliki audio losowo w profilu użytkownika (`~/`), dodano zmienną w konfiguracji wskazującą na podkatalog projektu w pamięci współdzielonej Androida. Pozwala to na łatwiejsze zarządzanie strukturą plików i ułatwia późniejsze czyszczenie zasobów.
