# Projekt: Detekcja ruchu Android + Termux + Flask + Bearer Token

## Cel projektu

Mamy dwa telefony z Androidem w tej samej sieci lokalnej:

1. **TELEFON A — telefon z kamerą**
    * Działa na nim Termux z rozszerzeniami `Termux:API` oraz uprawnieniami do pamięci (`termux-setup-storage`).
    * Uruchamia serwer Flask sterujący procesem detekcji.
    * Robi zdjęcia tylną kamerą, analizuje ruch, wykonuje serię zdjęć alarmowych i składa je w plik MP4 za pomocą `ffmpeg`.
    * Zapisuje zdarzenia w logu tekstowym `motion.log` i przesyła pliki na Telefon B.

2. **TELEFON B — telefon sterujący / odbierający pliki**
    * Działa na nim Termux z serwerem Flask.
    * Odbiera pliki z Telefonu A i zapisuje je w katalogu `Downloads/motion_files`.
    * Automatycznie czyści stare pliki (starsze niż 7 dni) za pomocą skryptu cron, chroniąc pamięć urządzenia przed przepełnieniem.

## Zabezpieczenie komunikacji

Komunikacja HTTP jest zabezpieczona tokenem typu Bearer Token.

Przykład nagłówka:
```http
Authorization: Bearer MOJ_SUPER_TAJNY_TOKEN
```

!!! warning "Ważne uwagi dotyczące bezpieczeństwa"
    * To nie jest szyfrowanie HTTPS (ruch jest jawny w sieci Wi-Fi).
    * Token chroni przed przypadkowym wywołaniem endpointów przez inne urządzenia w sieci domowej.
    * Najlepiej używać tego rozwiązania wyłącznie w zaufanej sieci Wi-Fi.

## Architektura systemu

```text
HTTP Shortcuts / przeglądarka / curl


        |
        |  Bearer Token
        v
TELEFON A: motion_camera_server.py  ---> Pisze do: motion.log

        |
        |  wykrycie ruchu -> zdjęcia -> składanie MP4 przez ffmpeg
        |

        |  Bearer Token (POST /upload)
        v
TELEFON B: receiver_server.py <--- Skrypt czyszczący: clean_motion.sh
        |
        v
Pamięć telefonu: Downloads/motion_files
```

---

## Część 1 — Instalacja na Telefonie A (Kamera)

### 1. Zainstaluj aplikacje
* **Termux** (zalecana wersja z F-Droid)
* **Termux:API** (zalecana wersja z F-Droid)

### 2. Konfiguracja środowiska Termux
Uruchom aplikację Termux na **Telefonie A** i wykonaj poniższe polecenia w podanej kolejności. 

!!! note "Ważna uwaga dotycząca OpenCV"
    Duże biblioteki graficzne (OpenCV) i matematyczne (NumPy) instalujemy za pomocą systemowego menedżera `pkg`, a nie przez `pip`. Zapobiega to wielogodzinnej, nieudanej kompilacji kodu na procesorze telefonu.

```bash
# Aktualizacja bazy pakietów Termux
pkg update && pkg upgrade -y

# Dodanie repozytorium X11 (wymagane dla pakietu OpenCV)
pkg install x11-repo -y

# Instalacja gotowego, skompilowanego pakietu OpenCV dla Androida
pkg install opencv-python -y

# ROZWIĄZANIE BŁĘDÓW DEPENDENCJI (ImportError: dlopen failed):
# Pakiet OpenCV wymaga ręcznego doinstalowania kodeków graficznych oraz szyny DBus:
pkg install libavif dbus -y

# Instalacja pozostałych narzędzi systemowych oraz lekkich bibliotek Pythona
pkg install termux-api ffmpeg -y
pip install flask requests

# Przyznanie Termuxowi dostępu do pamięci wewnętrznej telefonu
termux-setup-storage
```

#### Weryfikacja poprawności instalacji OpenCV:
Przed uruchomieniem serwera upewnij się, że wszystkie biblioteki systemowe ładują się poprawnie. Wpisz w konsoli:
```bash
python -c "import cv2; print('OpenCV działa poprawnie!')"
```
Jeśli zobaczysz komunikat `OpenCV działa poprawnie!`, środowisko jest w 100% gotowe do pracy.


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
Zanotuj adres IP (np. `192.168.1.50`), aby wpisać go w konfiguracji Telefonu A.

---

## Część 3 — Wybór tokenu

Ustal jeden wspólny token autoryzacyjny dla obu urządzeń, np.:
`MOJ_SUPER_TAJNY_TOKEN_123456`

---

## Część 4 — Kod dla Telefonu B: `receiver_server.py`

Uruchom na Telefonie B. Odbiera pliki z Telefonu A i zapisuje je w `Downloads/motion_files`.

### Utwórz plik
```bash
nano receiver_server.py
```

### Kod źródłowy
```python
from flask import Flask, request, jsonify
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
        "message": "Receiver działa",
        "save_dir": str(SAVE_DIR)
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

### POTWIERDZENIE ZMIAN W LOGICE:
1. **Wdrożenie modułu `logging`**: Zamiast standardowych instrukcji `print()`, aplikacja inicjuje rotacyjny logger zapisujący zdarzenia z precyzyjnym czasem oraz poziomem ważności (`INFO`, `ERROR`) do pliku `motion.log`.
2. **Pełna implementacja funkcji `handle_motion_event`**: Uzupełniono urwany uprzednio kod wysyłania plików graficznych i wideo (MP4) na Telefon B wraz z obsługą wyjątków sieciowych.

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
from pathlib import Path
from datetime import datetime
from functools import wraps
import cv2
import requests
from flask import Flask, jsonify, request

app = Flask(__name__)

# ==========================================================
# KONFIGURACJA SYSTEMOWA
# ==========================================================
API_TOKEN = "MOJ_SUPER_TAJNY_TOKEN_123456"
RECEIVER_URL = "http://192.168.8.81:7000/upload"  # Podmień na IP Telefonu B
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

LOW_DIR.mkdir(parents=True, exist_ok=True)
EVENT_DIR.mkdir(parents=True, exist_ok=True)

# ==========================================================
# KONFIGURACJA LOGOWANIA (motion.log)
# ==========================================================
# Plik logów zapisywany jest bezpośrednio w katalogu bazowym projektu
LOG_FILE_PATH = BASE_DIR / "motion.log"

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S',
    handlers=[
        logging.FileHandler(LOG_FILE_PATH, encoding='utf-8'),
        logging.StreamHandler()  # Jednoczesny podgląd w konsoli Termux
    ]
)

logging.info("--- Inicjalizacja systemu detekcji ruchu ---")
logging.info(f"Plik logów dostępny w: {LOG_FILE_PATH}")


# ==========================================================
# STAN PROGRAMU I SYNCHRONIZACJA WĄTKÓW
# ==========================================================
state = {
    "running": False,
    "busy_event": False,
    "last_motion": None,
    "last_error": None,
    "last_event_dir": None,
    "frames_checked": 0,
    "last_changed_pixels": None
}

worker_thread = None
stop_event = threading.Event()


# ==========================================================
# DEKORATOR AUTORYZACJI
# ==========================================================
def require_bearer_token(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        auth_header = request.headers.get("Authorization", "")
        if auth_header != f"Bearer {API_TOKEN}":
            logging.warning(f"Odrzucono nieautoryzowaną próbę dostępu z adresu: {request.remote_addr}")
            return jsonify({"ok": False, "error": "Brak autoryzacji"}), 401
        return func(*args, **kwargs)
    return wrapper


# ==========================================================
# FUNKCJE POMOCNICZE I OBSŁUGA SPRZĘTU (Termux:API / OpenCV / FFmpeg)
# ==========================================================
def now_string():
    return datetime.now().strftime("%Y%m%d_%H%M%S")


def run_command(command, timeout=30):
    """Wykonuje bezpiecznie proces zewnętrzny w systemie operacyjnym."""
    logging.debug(f"Wywoływanie komendy: {' '.join(command)}")
    result = subprocess.run(
        command,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        text=True,
        timeout=timeout
    )
    if result.returncode != 0:
        raise RuntimeError(f"Błąd procesu. STDERR: {result.stderr.strip()}")
    return result


def take_photo(path):
    """Pobiera klatkę z aparatu wywołując natywne narzędzie Termux:API."""
    path = Path(path)
    path.parent.mkdir(parents=True, exist_ok=True)

    # Wywołanie termux-camera-photo powoduje fizyczne otwarcie sterownika aparatu
    run_command([
        "termux-camera-photo",
        "-c",
        CAMERA_ID,
        str(path)
    ], timeout=40)
    return path


def load_small_gray_image(path):
    """Konwertuje obraz wejściowy, aby zmniejszyć obciążenie procesora."""
    image = cv2.imread(str(path))
    if image is None:
        raise ValueError(f"Brak pliku lub uszkodzony plik obrazu: {path}")

    # Obliczanie proporcji w celu zachowania aspektu przy zmniejszaniu szerokości do ANALYSIS_WIDTH
    height, width = image.shape[:2]
    scale = ANALYSIS_WIDTH / float(width)
    new_height = int(height * scale)

    resized = cv2.resize(image, (ANALYSIS_WIDTH, new_height))
    gray = cv2.cvtColor(resized, cv2.COLOR_BGR2GRAY)
    
    # GaussianBlur eliminuje drobny szum cyfrowy matrycy mogący generować fałszywy ruch
    gray = cv2.GaussianBlur(gray, (21, 21), 0)
    return gray


def detect_motion(previous_gray, current_gray):
    """Realizuje detekcję różnicową pomiędzy dwoma kolejnymi obrazami."""
    # Wyznaczenie bezwzględnej różnicy pikseli (Background Subtraction)
    frame_delta = cv2.absdiff(previous_gray, current_gray)

    # Binaryzacja - piksele o zmianie jasności powyżej 25 stają się białe (255)
    threshold = cv2.threshold(frame_delta, 25, 255, cv2.THRESH_BINARY)[1]
    
    # Dylatacja (pogrubienie) obiektów w celu zamknięcia luk w maskach ruchu
    threshold = cv2.dilate(threshold, None, iterations=2)

    # Zliczanie zmienionych pikseli
    changed_pixels = cv2.countNonZero(threshold)
    motion_detected = changed_pixels > MOTION_THRESHOLD_PIXELS

    return motion_detected, changed_pixels


def upload_file(path):
    """Wysyła plik binarny na Telefon B metodą POST Multipart/Form-Data."""
    path = Path(path)
    headers = {"Authorization": f"Bearer {API_TOKEN}"}

    logging.info(f"Rozpoczynanie wysyłania pliku: {path.name}")
    with path.open("rb") as f:
        response = requests.post(
            RECEIVER_URL,
            headers=headers,
            files={"file": (path.name, f)},
            timeout=90
        )
    response.raise_for_status()
    logging.info(f"Pomyślnie wysłano: {path.name}")
    return response.json()


def create_video_from_images(images_dir, output_video_path, fps=1):
    """Konwertuje sekwencję zdjęć o uporządkowanych nazwach do formatu MP4 za pomocą FFmpeg."""
    images_dir = Path(images_dir)
    output_video_path = Path(output_video_path)

    logging.info(f"Kompilacja wideo MP4 w katalogu: {images_dir.name}")
    run_command([
        "ffmpeg",
        "-y",
        "-framerate",
        str(fps),
        "-i",
        str(images_dir / "frame_%04d.jpg"),
        "-c:v",
        "mpeg4",
        "-pix_fmt",
        "yuv420p",
        str(output_video_path)
    ], timeout=180)
    logging.info(f"Wygenerowano plik wideo: {output_video_path.name}")
    return output_video_path


def handle_motion_event():
    """Wykonuje całą sekwencję alarmową po wykryciu zdarzenia ruchu."""
    state["busy_event"] = True
    logging.info("--- WYKRYTO RUCH: Uruchamianie procedury alarmowej ---")

    try:
        event_name = f"event_{now_string()}"
        event_dir = EVENT_DIR / event_name
        photos_dir = event_dir / "photos"
        frames_dir = event_dir / "frames"

        photos_dir.mkdir(parents=True, exist_ok=True)
        frames_dir.mkdir(parents=True, exist_ok=True)

        state["last_event_dir"] = str(event_dir)
        state["last_motion"] = datetime.now().isoformat(timespec="seconds")

        photo_paths = []

        # KROK 1: Przechwycenie serii zdjęć wysokiej rozdzielczości (Burst)
        for i in range(BURST_PHOTOS):
            if stop_event.is_set():
                return
            photo_path = photos_dir / f"photo_{i + 1:02d}.jpg"
            logging.info(f"Przechwytywanie zdjęcia burst ({i+1}/{BURST_PHOTOS})")
            take_photo(photo_path)
            photo_paths.append(photo_path)
            time.sleep(0.4)

        # KROK 2: Przechwycenie klatek w stałym interwale do pliku wideo
        total_frames = RECORD_SECONDS * RECORD_FPS
        for i in range(total_frames):
            if stop_event.is_set():
                return
            frame_path = frames_dir / f"frame_{i + 1:04d}.jpg"
            logging.info(f"Przechwytywanie klatki wideo ({i+1}/{total_frames})")
            take_photo(frame_path)
            time.sleep(max(0, 1.0 / RECORD_FPS))

        # KROK 3: Renderowanie MP4 ze zgromadzonych klatek
        video_path = event_dir / f"{event_name}.mp4"
        try:
            create_video_from_images(frames_dir, video_path, fps=RECORD_FPS)
        except Exception as exc:
            logging.error(f"Renderowanie wideo nie powiodło się: {exc}")
            state["last_error"] = str(exc)
            video_path = None

        # KROK 4: Transfer danych na serwer odbiorczy (Telefon B)
        for photo_path in photo_paths:
            try:
                upload_file(photo_path)
            except Exception as exc:
                logging.error(f"Nieudany upload zdjęcia {photo_path.name}: {exc}")

        if video_path and video_path.exists():
            try:
                upload_file(video_path)
            except Exception as exc:
                logging.error(f"Nieudany upload wideo {video_path.name}: {exc}")

    except Exception as g_exc:
        logging.error(f"Krytyczny błąd podczas przetwarzania zdarzenia: {g_exc}")
        state["last_error"] = str(g_exc)
    finally:
        state["busy_event"] = False
        logging.info("--- Zakończono przetwarzanie zdarzenia. Powrót do czuwania ---")


# ==========================================================
# WĄTEK ROBOCZY (MONITORING CIĄGŁY)
# ==========================================================
def motion_worker():
    """Pętla nadzorująca zmiany obrazu pracująca w tle."""
    previous_gray = None
    logging.info("Wątek roboczy detekcji został uruchomiony.")
    
    while not stop_event.is_set():
        if state["busy_event"]:
            # Zawieszenie sprawdzania tła, gdy realizowane jest nagrywanie zdarzenia
            time.sleep(CHECK_INTERVAL_SECONDS)
            continue
            
        try:
            current_low_path = LOW_DIR / "current_check.jpg"
            take_photo(current_low_path)
            current_gray = load_small_gray_image(current_low_path)
            state["frames_checked"] += 1
            
            if previous_gray is not None:
                detected, pixels = detect_motion(previous_gray, current_gray)
                state["last_changed_pixels"] = pixels
                
                if detected:
                    logging.info(f"Wykryto zmianę struktury klatki! Zmienione piksele: {pixels}")
                    # Wywołanie obsługi w osobnym wątku unika zablokowania pętli głównej
                    threading.Thread(target=handle_motion_event, daemon=True).start()
                    
            previous_gray = current_gray
        except Exception as err:
            logging.error(f"Błąd w pętli monitorującej: {err}")
            state["last_error"] = str(err)
            
        time.sleep(CHECK_INTERVAL_SECONDS)


# ==========================================================
# ENDPOINTY FLASK (STEROWANIE INTERFEJSEM)
# ==========================================================p
@app.route("/status", methods=["GET"])
@require_bearer_token
def get_status():
    return jsonify({"ok": True, "state": state})


@app.route("/start", methods=["POST"])
@require_bearer_token
def start_detection():
    global worker_thread
    if state["running"]:
        return jsonify({"ok": True, "message": "System już pracuje."})
        
    logging.info("Żądanie HTTP: Uruchomienie detekcji.")
    state["running"] = True
    stop_event.clear()
    worker_thread = threading.Thread(target=motion_worker, daemon=True)
    worker_thread.start()
    return jsonify({"ok": True, "message": "Detekcja uruchomiona."})


@app.route("/stop", methods=["POST"])
@require_bearer_token
def stop_detection():
    global worker_thread
    if not state["running"]:
        return jsonify({"ok": True, "message": "System nie jest uruchomiony."})
        
    logging.info("Żądanie HTTP: Zatrzymanie detekcji.")
    state["running"] = False
    stop_event.set()
    if worker_thread:
        worker_thread.join(timeout=5)
    return jsonify({"ok": True, "message": "Detekcja zatrzymana."})


@app.route("/test-photo", methods=["POST"])
@require_bearer_token
def test_photo():
    logging.info("Żądanie HTTP: Wymuszenie zdjęcia testowego.")
    try:
        test_path = BASE_DIR / f"test_{now_string()}.jpg"
        take_photo(test_path) 
        upload_result = upload_file(test_path)
        return jsonify({"ok": True, "message": "Zdjęcie testowe przetworzone", "result": upload_result})
    except Exception as err:
        logging.error(f"Błąd wymuszenia zdjęcia testowego: {err}")
        return jsonify({"ok": False, "error": str(err)}), 500


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

---

## Część 6 — Automatyczne czyszczenie dysku na Telefonie B

### POTWIERDZENIE LOGIKI:
Skrypt wykorzystuje natywne narzędzie Unix `find`. Filtruje i usuwa pliki o rozszerzeniach `.jpg` oraz `.mp4`, których data ostatniej modyfikacji przekracza 7 dni (`-mtime +7`). Zadanie automatyzowane jest za pomocą pakietu `cronie` dedykowanego dla Termux.

### 1. Tworzenie skryptu czyszczącego
Na **Telefonie B** utwórz katalog na skrypty i utwórz plik basha:
```bash
mkdir -p ~/scripts
nano ~/scripts/clean_motion.sh
```

Wklej poniższą zawartość:
```bash
#!/usr/bin/env bash

# ==============================================================================
# SKRYPT AUTOMATYCZNEGO CZYSZCZENIA STARYCH PLIKÓW DETEKCJI
# ==============================================================================

# Definicja katalogu docelowego - bezwzględna ścieżka środowiska Termux
TARGET_DIR="$HOME/storage/downloads/motion_files"

# Weryfikacja czy katalog istnieje, aby zapobiec błędom wykonania find
if [ -d "$TARGET_DIR" ]; then
    # find wyszukuje pliki regularne (-type f) zmodyfikowane dawniej niż 7 dni temu (-mtime +7)
    # i usuwa je strukturalnie (-delete)
    find "$TARGET_DIR" -type f \( -name "*.jpg" -o -name "*.mp4" \) -mtime +7 -delete
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] Czyszczenie wykonane pomyślnie." >> "$TARGET_DIR/clean_log.txt"
else
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] BŁĄD: Katalog docelowy nie istnieje." >> "$HOME/clean_error.txt"
fi
```

Nadaj uprawnienia do wykonywania skryptu:
```bash
chmod +x ~/scripts/clean_motion.sh
```

### 2. Automatyzacja za pomocą harmonogramu Cron
W celu cyklicznego uruchamiania skryptu codziennie o północy, skonfiguruj demona cron w Termuxie:

Otwórz edytor tabeli cron:
```bash
crontab -e
```

Wklej poniższą regułę (używaj wyłącznie pełnych, bezwzględnych ścieżek):
```text
0 0 * * * /data/data/com.termux/files/home/scripts/clean_motion.sh
```

Upewnij się, że usługa cron jest włączona i działa w tle. Jeśli menedżer usług zgłasza błąd braku katalogu (`file does not exist`), wymuś odświeżenie profili środowiskowych Termuxa przed aktywacją:
```bash
# Wczytanie skryptu startowego usług Termux (ładowanie brakujących punktów montowania)
source $PREFIX/etc/profile.d/start-services.sh

# Włączenie i uruchomienie demona cron w tle
sv-enable crond
sv up crond
```

Weryfikację poprawności działania demona możesz przeprowadzić za pomocą polecenia:
```bash
sv status crond
```


---

## Część 7 — Uruchomienie systemu i zdalne sterowanie (HTTP Shortcuts / Terminal)

Systemem możesz sterować na dwa sposoby: poprzez aplikację graficzną **HTTP Shortcuts** lub bezpośrednio z konsoli **Telefonu B** za pomocą poleceń tekstowych `curl`.

### 1. Procedura startowa na Telefonie A (Kamera)

Samo uruchomienie skryptu Pythona na Telefonie A włącza serwer, ale **nie uruchamia jeszcze aparatu ani analizy ruchu** (system czeka w bezpiecznym trybie uśpienia). Aby wystartować cały proces:

1. Otwórz Termux na **Telefonie A** i uruchom serwer:
   ```bash
   python motion_camera_server.py
   ```
2. Serwer zacznie działać w tle i wyświetli log o inicjalizacji.
3. System jest teraz w gotowości. Aby aparat zaczął fizycznie analizować obraz, musisz wysłać komendę **START** z Telefonu B (za pomocą HTTP Shortcuts lub komendy w konsoli, patrz sekcje poniżej).

---

### 2. Alternatywa: Sterowanie z poziomu konsoli Telefonu B (Komendy curl)

Jeżeli nie masz pod ręką aplikacji HTTP Shortcuts, możesz zarządzać pracą Telefonu A bezpośrednio z poziomu wiersza poleceń **Telefonu B**. 

Otwórz nową sesję Termux na **Telefonie B** i użyj poniższych poleceń (przy założeniu, że Telefon A ma adres IP `192.168.8.246` i port `8000`):

* **URUCHOMIENIE analizy ruchu i aparatu (START):**
  ```bash
  curl -X POST -H "Authorization: Bearer MOJ_SUPER_TAJNY_TOKEN_123456" http://192.168.8.246:8000/start
  ```
  *Serwer A odpowie komunikatem: `{"message":"Detekcja uruchomiona.","ok":true}` i rozpocznie monitorowanie pikseli.*

* **ZATRZYMANIE analizy ruchu i uśpienie aparatu (STOP):**
  ```bash
  curl -X POST -H "Authorization: Bearer MOJ_SUPER_TAJNY_TOKEN_123456" http://192.168.8.246:8000/stop
  ```
  *Serwer A wyłączy aparat, przerwie pętlę monitorującą i przejdzie w stan czuwania.*

* **Wymuszenie wykonania natychmiastowego zdjęcia testowego:**
  ```bash
  curl -X POST -H "Authorization: Bearer MOJ_SUPER_TAJNY_TOKEN_123456" http://192.168.8.246:8000/test-photo
  ```
  *Zmusza Telefon A do zrobienia jednej fotki i natychmiastowego przesłania jej do Telefonu B.*

* **Podgląd pełnych statystyk i stanu pamięci (STATUS):**
  ```bash
  curl -X GET -H "Authorization: Bearer MOJ_SUPER_TAJNY_TOKEN_123456" http://192.168.8.246:8000/status
  ```
  *Zwraca szczegółowy obiekt JSON zawierający liczbę sprawdzonych klatek, ostatnie błędy oraz informację, czy system aktualnie nagrywa.*

---

### 3. Integracja z aplikacją HTTP Shortcuts (Szybki import)

Możesz skopiować dokładnie te same polecenia `curl` z Sekcji 2 i wkleić je w aplikacji za pomocą opcji **Import ze składni cURL (Import from cURL)**, aby automatycznie wygenerować przyciski na pulpicie Androida.

---

### 4. Struktura konfiguracji przycisku w formacie JSON

Poniżej znajduje się oficjalny, kompletny format obiektu JSON dla przycisku **START**, który możesz zaimportować do aplikacji HTTP Shortcuts. Zawiera on pełny adres URL z końcówką `/start` oraz automatyczną obsługę wyskakujących powiadomień systemowych (Toast):

```json
{
  "name": "Uruchom Detekcję",
  "method": "POST",
  "url": "http://192.168.1",
  "headers": [
    {
      "key": "Authorization",
      "value": "Bearer MOJ_SUPER_TAJNY_TOKEN_123456"
    }
  ],
  "executionType": "APP",
  "feedback": "TOAST",
  "timeout": 10000,
  "responseHandling": {
    "responseDisplayOutput": "BODY",
    "responseSuccessType": "TOAST",
    "responseErrorType": "TOAST_ERROR"
  }
}
```

---
---
---
prompt do debuggingu

Oto jeden, skondensowany prompt (streszczenie kontekstu), który możesz skopiować i zapisać w pliku tekstowym. W przyszłości wystarczy, że wkleisz go na początku nowej rozmowy z AI, a model natychmiast odtworzy całą architekturę, konfigurację sieciową, kody oraz rozwiązane problemy techniczne.

Działamy w kontekście projektu: "Detekcja ruchu Android + Termux + Flask + Bearer Token". Odtwórz pełny kontekst systemu składającego się z dwóch telefonów w sieci lokalnej:
```
1. ARCHITEKTURA I ENPOINTY:
- TELEFON A (Kamera, IP domyślne: 192.168.1.40): Uruchamia serwer Flask na porcie 8000. Za pomocą OpenCV (analiza różnicowa pikseli o progu MOTION_THRESHOLD_PIXELS = 2500 i szerokości analizy 320px) wykrywa ruch. Po wykryciu wykonuje BURST_PHOTOS = 4 oraz sekwencję zdjęć przez RECORD_SECONDS = 15 z prędkością RECORD_FPS = 1, składa je przez FFmpeg w plik MP4 i wysyła na Telefon B. Endpointy sterujące na Telefonie A (zabezpieczone Bearer Tokenem): POST /start, POST /stop, POST /test-photo, GET /status. Wszystkie operacje loguje do pliku 'motion.log' przez moduł logging.
- TELEFON B (Odbiornik, IP domyślne: 192.168.1.50): Uruchamia serwer Flask na porcie 7000. Posiada endpoint POST /upload przyjmujący pliki multipart/form-data i zapisujący je w katalogu 'Downloads/motion_files'. Działa na nim demon cron (pakiet cronie), który codziennie o północy wykonuje skrypt bash '~/scripts/clean_motion.sh' usuwający pliki .jpg/.mp4 starsze niż 7 dni (-mtime +7).

2. BEZPIECZEŃSTWO:
- Komunikacja jest zabezpieczona nagłówkiem "Authorization: Bearer MOJ_SUPER_TAJNY_TOKEN_123456" weryfikowanym przez dekorator @require_bearer_token na obu telefonach.

3. ROZWIĄZANE PROBLEMY TECHNICZNE (Kluczowe dla stabilności środowiska):
- Aby uniknąć wiecznej kompilacji NumPy/OpenCV w pip na architekturze ARM, pakiety te instalujemy przez systemowe repozytorium: `pkg install x11-repo && pkg install opencv-python`.
- Brakujące biblioteki współdzielone (.so) dla OpenCV w Termuxie naprawiliśmy poprzez: `pkg install libavif dbus -y`. Pozostałe zależności to `pkg install termux-api ffmpeg -y` oraz `pip install flask requests`.
- Błąd rejestracji usługi cron 'file does not exist' w menedżerze sv/runit na Telefonie B został rozwiązany przez wymuszenie przeładowania profilu usług poleceniem: `source $PREFIX/etc/profile.d/start-services.sh` przed wywołaniem sv-enable/sv up.
- Błędy HTTP 404 w curl/HTTP Shortcuts wynikają z pominięcia numeru portu (:8000) lub braku ukośnika przed endpointem (prawidłowy format to np. http://192.168.1).

Potwierdź, że w pełni rozumiesz tę architekturę, logikę kodu, strukturę katalogów oraz konfigurację sieciową/zależności systemowe Termuxa. Od teraz odpowiadaj w kontekście tego projektu.
```
