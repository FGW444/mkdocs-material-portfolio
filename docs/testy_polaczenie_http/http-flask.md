


# Komunikacja HTTP w pythonie z framework flask - old

## Założenia

(diagram z tego zrob)

Stary telefon → okresowe zdjęcia → Termux + Tailscale → odbiorca z nginx/Flask. Zakładam, że Tailscale jest skonfigurowane i oba urządzenia widzą się w sieci.


## 1. Termux — jednorazowa instalacja

```
pkg update && pkg upgrade
pkg install python git termux-api
pip install requests pillow
# włącz uprawnienia: Settings → Apps → Termux → Camera, Storage
```

## 2. Skrypt Termux: send_photo.py 

Zapisz plik w Termux (~/.local/bin lub /data/data/com.termux/files/home/).

```
#!/data/data/com.termux/files/usr/bin/env python3
import os, time, requests, datetime
from subprocess import run

# CONFIG
INTERVAL = 5
DEST_URL = "http://<RECEIVER_TAILSCALE_IP>:5000/upload"
AUTH_TOKEN = "moja_tajna_token"
TMP_DIR = "/data/data/com.termux/files/home/pics"
JPEG_QUALITY = 75

os.makedirs(TMP_DIR, exist_ok=True)

def take_photo(path):
    return run(["termux-camera-photo", path]).returncode == 0

def compress_jpeg(path, quality):
    try:
        from PIL import Image
        img = Image.open(path)
        img.save(path, "JPEG", quality=quality)
    except Exception:
        pass
    return path

while True:
    ts = datetime.datetime.utcnow().strftime("%Y%m%dT%H%M%SZ")
    filename = f"photo_{ts}.jpg"
    path = os.path.join(TMP_DIR, filename)
    if not take_photo(path):
        time.sleep(INTERVAL); continue
    compress_jpeg(path, JPEG_QUALITY)
    with open(path, "rb") as f:
        files = {"file": (filename, f, "image/jpeg")}
        headers = {"Authorization": f"Bearer {AUTH_TOKEN}"}
        try:
            r = requests.post(DEST_URL, files=files, headers=headers, timeout=20)
            if r.status_code == 200:
                os.remove(path)
            else:
                print("Upload failed:", r.status_code)
        except Exception as e:
            print("Upload error:", e)
    time.sleep(INTERVAL)
```

Ustaw prawa i uruchom w tle:

```
chmod +x send_photo.py
termux-wake-lock
# uruchom w tle (np. w tmux/screen)
python3 send_photo.py &
```

Wskazówki:
INTERVAL ≥ 2–5 s (krócej obciąży baterię).
Jeśli Pillow nie działa: pip install pillow.


## 3. Sieć — Tailscale

Zainstaluj Tailscale na obu urządzeniach.
Zaloguj się i sprawdź Tailscale IP odbiorcy; użyj go w DEST_URL.
Tailscale zapewnia prywatną sieć bez NAT/port-forward.


## 4. Odbiorca — prosty Flask (odbiera POST)

Na urządzeniu odbiorczym (telefon / PC) z Pythonem:
Instalacja:

```
pip install flask
```

server.py:

```
from flask import Flask, request, abort
import os, datetime

APP_DIR = os.path.dirname(os.path.abspath(__file__))
SAVE_DIR = os.path.join(APP_DIR, "received")
os.makedirs(SAVE_DIR, exist_ok=True)
AUTH_TOKEN = "moja_tajna_token"

app = Flask(__name__)

@app.route("/upload", methods=["POST"])
def upload():
    auth = request.headers.get("Authorization","")
    if auth != f"Bearer {AUTH_TOKEN}":
        abort(401)
    if "file" not in request.files:
        abort(400)
    f = request.files["file"]
    ts = datetime.datetime.utcnow().strftime("%Y%m%dT%H%M%SZ")
    filename = f"{ts}_{f.filename}"
    f.save(os.path.join(SAVE_DIR, filename))
    return "OK", 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```
Uruchom:

```
python3 server.py
```

Dla stabilności/prod: użyj nginx + gunicorn/uWSGI i SSL.

## 5. Rotacja i monitoring dysku

Dodaj prosty skrypt cron, który usuwa pliki starsze niż N dni:

```
find /path/to/received -type f -mtime +7 -delete
```

## 6. Bezpieczeństwo i optymalizacje

- Użyj tokenu w nagłówku Authorization (jak powyżej).
- Tailscale daje prywatną sieć — nadal rozważ HTTPS jeśli wystawiasz port publicznie.
- Kompresja JPEG i ograniczenie rozdzielczości aby oszczędzać transfer/baterię.
- Zwiększ stabilność: uruchom serwer jako usługa/systemd lub użyj gunicorn.

## 7. Szybkie checklisty (do śledzenia etapów projektu)

 - Zainstalowany Termux, przyznane uprawnienia
 - Zainstalowany Tailscale na obu urządzeniach
 - Skrypt send_photo.py na telefonie z poprawnym DEST_URL i tokenem
 - Skrypt uruchamiany w tle (termux-wake-lock + tmux)
 - Flask/nginx uruchomiony na odbiorcy, endpoint /upload działa
 - Test: ręczne POST wysyła plik i zapisuje na odbiorcy
 - Rotacja plików / monitoring dysku
 - Optymalizacja jakości/INTERVAL według baterii i przepustowości

 bonus

 Jeśli chcesz, przygotuję teraz:

- wersję z adb/scrcpy zamiast termux-camera-photo, lub
- instrukcję konfiguracji nginx + systemd dla serwera odbiorcy.


---

---

---



## Testowy skrypt Termux — lokalne robienie zdjęć co N sekund (bez wysyłania) - OK dziala poprawnie

Poniżej masz kompletny, samodzielny skrypt Python do uruchomienia w Termux, który:

- cyklicznie robi zdjęcia przy użyciu termux-camera-photo,
- zapisuje je w katalogu lokalnym,
- opcjonalnie kompresuje przy pomocy Pillow,
- loguje zdarzenia i błędy (print),
- nie łączy się z odbiorcą — możesz przetestować mechanikę i uprawnienia.

Zapisz plik jako send_photo_local.py w Termux (np. /data/data/com.termux/files/home/send_photo_local.py) i nadaj prawa wykonywalne.


```

#!/usr/bin/env python3
"""
send_photo_local.py
Prosty test: cykliczne robienie zdjęć lokalnie w Termux (bez wysyłania).
- Używa termux-camera-photo (wymaga uprawnień kamery).
- Opcjonalna kompresja z Pillow (pip install pillow).
- Loguje nazwy plików i status.
Konfiguracja: zmień INTERVAL, TMP_DIR, JPEG_QUALITY.
"""

import os
import time
import datetime
from subprocess import run

# ---------- KONFIGURACJA ----------
INTERVAL = 5                    # sekundy między zdjęciami
TMP_DIR = os.path.expanduser("~/storage/shared/Download/pics_test")  # publiczny katalog Download
JPEG_QUALITY = 75               # 1-100 (jeśli Pillow jest zainstalowane)
MAX_FILES = 200                 # opcjonalne ograniczenie (usuń najstarsze po przekroczeniu)
# -----------------------------------

os.makedirs(TMP_DIR, exist_ok=True)

def now_ts():
    return datetime.datetime.utcnow().strftime("%Y%m%dT%H%M%SZ")

def take_photo(path):
    """
    Wywołuje termux-camera-photo <path>, zwraca tuple (ok:bool, stdout:str, stderr:str).
    """
    try:
        res = run(["termux-camera-photo", path], capture_output=True, text=True)
        ok = res.returncode == 0
        return ok, res.stdout, res.stderr
    except FileNotFoundError:
        return False, "", "termux-camera-photo not found"
    except Exception as e:
        return False, "", str(e)

def compress_jpeg(path, quality):
    """
    Jeśli jest Pillow, otwórz, skonwertuj do RGB i zapisz z mniejszą jakością.
    """
    try:
        from PIL import Image
    except Exception:
        return False
    try:
        img = Image.open(path)
        img = img.convert("RGB")
        img.save(path, "JPEG", quality=quality)
        return True
    except Exception as e:
        print("Błąd podczas kompresji:", e)
        return False

def rotate_old_files(directory, max_files):
    try:
        files = [os.path.join(directory, f) for f in os.listdir(directory) if os.path.isfile(os.path.join(directory, f))]
        if len(files) <= max_files:
            return
        files.sort(key=lambda p: os.path.getmtime(p))
        to_delete = files[:len(files) - max_files]
        for p in to_delete:
            try:
                os.remove(p)
                print(f"Usunięto stary plik: {p}")
            except Exception as e:
                print("Błąd usuwania pliku:", p, e)
    except Exception as e:
        print("Błąd w rotate_old_files:", e)

def validate_config():
    global INTERVAL, MAX_FILES
    if not isinstance(INTERVAL, (int, float)) or INTERVAL < 1:
        print("INTERVAL ustawiono na zbyt małą wartość. Ustawiam na 1s.")
        INTERVAL = 1
    if not isinstance(MAX_FILES, int) or MAX_FILES < 1:
        print("MAX_FILES niepoprawne. Ustawiam na 100.")
        MAX_FILES = 100

def main_loop():
    validate_config()
    print(f"[{now_ts()}] Start testu. Zapisywanie zdjęć do: {TMP_DIR}")
    while True:
        ts = now_ts()
        filename = f"photo_{ts}.jpg"
        path = os.path.join(TMP_DIR, filename)

        print(f"[{ts}] Robię zdjęcie -> {filename}")
        ok, out, err = take_photo(path)
        if not ok:
            print(f"[{ts}] termux-camera-photo failed. stderr: {err.strip()}")
            time.sleep(INTERVAL)
            continue

        if not os.path.isfile(path):
            print(f"[{ts}] Plik nie istnieje po wywołaniu termux-camera-photo. stdout: {out.strip()} stderr: {err.strip()}")
            time.sleep(INTERVAL)
            continue

        compressed = compress_jpeg(path, JPEG_QUALITY)
        if compressed:
            print(f"[{ts}] Skompresowano {filename} (quality={JPEG_QUALITY})")
        else:
            print(f"[{ts}] Kompresja pominięta lub nieudana.")

        rotate_old_files(TMP_DIR, MAX_FILES)

        try:
            size = os.path.getsize(path)
            print(f"[{ts}] Zapisano {filename} — {size} bajtów")
        except Exception as e:
            print("Błąd pobierania rozmiaru pliku:", e)

        time.sleep(INTERVAL)

if __name__ == "__main__":
    try:
        main_loop()
    except KeyboardInterrupt:
        print(f"[{now_ts()}] Zatrzymano test (CTRL+C).")

```

### Instrukcja uruchomienia i testowania

- Zainstaluj wymagania w Termux:
 - - pkg install python termux-api
- - (opcjonalnie) pip install pillow
- Nadaj prawa: chmod +x send_photo_local.py
- Nadaj Termux uprawnienia do kamery: Settings → Apps → Termux → Permissions → Camera
- Uruchom:
- - python3 send_photo_local.py
- - Lub w tle: termux-wake-lock; python3 send_photo_local.py & (zalecane użycie tmux/screen)
- Sprawdź katalog TMP_DIR (domyślnie ~/pics_test) — pliki powinny się pojawiać co INTERVAL sekund.

Komentarze w kodzie (co się dzieje)

- take_photo(): wywołuje narzędzie termux-camera-photo, które otwiera domyślną kamerę i zapisuje plik do wskazanej ścieżki.
- compress_jpeg(): próbuje użyć Pillow do zmniejszenia jakości obrazu (oszczędność miejsca i transferu).
- rotate_old_files(): usuwa najstarsze zdjęcia, jeśli katalog przekroczy MAX_FILES — zabezpieczenie przed zapełnieniem pamięci podczas testów.
- Główna pętla tworzy unikalne nazwy z timestampem UTC, obsługuje błędy i loguje wynik.

Jeśli chcesz, mogę:

- przygotować wersję używającą adb/scrcpy (jeśli telefon nie ma termux-api), lub
- dodać prosty skrypt, który wyświetli mini-podsumowanie (liczba zdjęć, średni rozmiar) do dokumentacji. Które preferujesz?

---

---

---

# Testowy skrypt do sprawdzenie dzialania endpointow w flask

Poniżej masz prosty Flask daemon do uruchomienia w Termux, który udostępnia endpointy:

- POST /vibrate — wibruje krótko telefonem,
- GET /bat — zwraca status baterii (JSON),
- POST /torch — body JSON {"action":"on"} lub {"action":"off"} do włączenia/wyłączenia latarki.

Wymagania w Termux:

- pkg install python termux-api
- pip install flask
- Zapisz jako daemon.py i uruchom: python3 daemon.py

(Upewnij się, że Termux ma przyznane uprawnienia do wibracji, baterii i latarki; niektóre funkcje wymagają termux-api.)
daemon.py


#### version 01

```
#!/data/data/com.termux/files/usr/bin/env python3
from flask import Flask, request, jsonify, abort
import subprocess, os, datetime

APP = Flask(__name__)
# prosty token autoryzacji — zmień przed użyciem
AUTH_TOKEN = "moja_tajna_token"

def check_auth():
    auth = request.headers.get("Authorization","")
    if auth != f"Bearer {AUTH_TOKEN}":
        abort(401)

def run_cmd(args, timeout=10):
    try:
        p = subprocess.run(args, capture_output=True, text=True, timeout=timeout)
        return p.returncode, p.stdout.strip(), p.stderr.strip()
    except Exception as e:
        return -1, "", str(e)

@APP.route("/vibrate", methods=["POST"])
def vibrate():
    check_auth()
    # termux-vibrate <duration_ms>
    code, out, err = run_cmd(["termux-vibrate", "300"])
    if code == 0:
        return "OK", 200
    return f"Error: {err or out}", 500

@APP.route("/bat", methods=["GET"])
def battery():
    check_auth()
    # termux-battery-status zwraca JSON
    code, out, err = run_cmd(["termux-battery-status"])
    if code == 0 and out:
        try:
            return out, 200, {"Content-Type":"application/json"}
        except Exception:
            return out, 200
    return f"Error: {err}", 500

@APP.route("/torch", methods=["POST"])
def torch():
    check_auth()
    data = {}
    try:
        data = request.get_json(force=True)
    except Exception:
        pass
    action = (data.get("action") if isinstance(data, dict) else None) or request.args.get("action")
    if not action or action.lower() not in ("on","off"):
        return "Invalid action; use JSON {'action':'on'} or {'action':'off'}", 400
    if action.lower() == "on":
        code, out, err = run_cmd(["termux-torch", "on"])
    else:
        code, out, err = run_cmd(["termux-torch", "off"])
    if code == 0:
        return "OK", 200
    return f"Error: {err or out}", 500

@APP.route("/status", methods=["GET"])
def status():
    check_auth()
    info = {
        "time": datetime.datetime.utcnow().isoformat()+"Z",
        "uptime": run_cmd(["uptime"])[1]
    }
    return jsonify(info)

if __name__ == "__main__":
    # nasłuchaj na wszystkich interfejsach w sieci lokalnej
    APP.run(host="0.0.0.0", port=8080)
```
Jak testować z laptopa (zakładając IP telefonu 192.168.1.50) — użyj curl i dołącz nagłówek Authorization:

- Vibrate: curl -X POST -H "Authorization: Bearer moja_tajna_token" http://192.168.1.50:8080/vibrate
- Battery: curl -H "Authorization: Bearer moja_tajna_token" http://192.168.1.50:8080/bat
- Torch on: curl -X POST -H "Authorization: Bearer moja_tajna_token" -H "Content-Type: application/json" -d '{"action":"on"}' http://192.168.1.50:8080/torch
- Torch off: curl -X POST -H "Authorization: Bearer moja_tajna_token" -H "Content-Type: application/json" -d '{"action":"off"}' http://192.168.1.50:8080/torch
- Status: curl -H "Authorization: Bearer moja_tajna_token" http://192.168.1.50:8080/status

Uwagi i wskazówki:
- Jeśli termux-* komendy nie są dostępne, zainstaluj pakiet termux-api: pkg install termux-api.
- Uprawnienia: w Android Settings przyznaj Termux dostęp do kamery/flash/battery/vibrate jeśli wymagane.
- Nie uruchamiaj tego serwera publicznie bez zabezpieczeń; podczas testów stosuj token i używaj sieci zaufanej (LAN) lub Tailscale.
- Dla trwałego uruchomienia użyj tmux/screen lub uruchom jako usługa; przed długotrwałymi zadaniami użyj termux-wake-lock.

Chcesz, żebym też przygotował wersję, która uruchamia send_photo_network.py przez endpoint /start i zatrzymuje przez /stop?*

##### version v.02

```
#!/data/data/com.termux/files/usr/bin/env python3
from flask import Flask, request, jsonify, abort
import subprocess
import datetime

app = Flask(__name__)

# Zmień token na własny!
AUTH_TOKEN = "moja_tajna_token"

def check_auth():
    """Sprawdza nagłówek Authorization."""
    auth = request.headers.get("Authorization", "")
    if auth != f"Bearer {AUTH_TOKEN}":
        abort(401, description="Brak autoryzacji")

def run_cmd(args, timeout=10):
    """Bezpieczne wywoływanie komend systemowych Termux."""
    try:
        p = subprocess.run(args, capture_output=True, text=True, timeout=timeout)
        return p.returncode, p.stdout.strip(), p.stderr.strip()
    except Exception as e:
        return -1, "", str(e)

@app.before_request
def before_request():
    """Automatycznie sprawdza autoryzację dla każdego endpointu poza ewentualnym /health."""
    check_auth()

@app.route("/vibrate", methods=["POST"])
def vibrate():
    # termux-vibrate: -d to czas w ms
    code, out, err = run_cmd(["termux-vibrate", "-d", "300"])
    if code == 0:
        return jsonify({"status": "ok"}), 200
    return jsonify({"error": err or out}), 500

@app.route("/bat", methods=["GET"])
def battery():
    code, out, err = run_cmd(["termux-battery-status"])
    if code == 0 and out:
        # out jest już stringiem JSON z termux-api, przekazujemy go bezpośrednio
        from flask import Response
        return Response(out, mimetype='application/json')
    return jsonify({"error": err}), 500

@app.route("/torch", methods=["POST"])
def torch():
    # Pobieranie akcji z JSON lub parametrów URL (?action=on)
    data = request.get_json(silent=True) or {}
    action = data.get("action") or request.args.get("action")

    if action not in ("on", "off"):
        return jsonify({"error": "Użyj {'action':'on'} lub {'action':'off'}"}), 400

    code, out, err = run_cmd(["termux-torch", action])
    if code == 0:
        return jsonify({"status": f"torch {action}"}), 200
    return jsonify({"error": err or out}), 500

@app.route("/status", methods=["GET"])
def status():
    # datetime.datetime.now(datetime.timezone.utc) jest zalecane zamiast utcnow()
    info = {
        "time": datetime.datetime.now(datetime.timezone.utc).isoformat(),
        "uptime": run_cmd(["uptime", "-p"])[1] # -p daje ładniejszy format
    }
    return jsonify(info)

if __name__ == "__main__":
    # debug=False na produkcji, ale w Termux pomaga przy testach
    app.run(host="0.0.0.0", port=8080, debug=False)

```

Co poprawiłem:

- Struktura autoryzacji: Dodałem dekorator @app.before_request, dzięki czemu nie musisz ręcznie wpisywać check_auth() w każdej funkcji. Jest czyściej.
- Obsługa JSON: W torch użyłem get_json(silent=True). Zapobiega to wywaleniu błędu 400 przez Flask, jeśli wyślesz pusty body lub zapomnisz nagłówka Content-Type.
- Spójność odpowiedzi: Teraz wszystkie endpointy (oprócz baterii, która zwraca własny obiekt) zwracają jsonify, co jest standardem w API.
- Komenda wibracji: Dodałem flagę -d w termux-vibrate, która w nowszych wersjach API jest wymagana do określenia czasu trwania.
- Bezpieczeństwo: Zmieniłem datetime.utcnow() (które jest odradzane w nowym Pythonie) na nowoczesne podejście z timezone.utc.

Jak teraz testować (przykłady):

- Wibracja pozostaje bez zmian, ale dla latarki możesz teraz wysłać to prościej:
- JSON: curl -X POST -H "Authorization: Bearer moja_tajna_token" -H "Content-Type: application/json" -d '{"action":"on"}' http://IP:8080/torch
- URL (opcjonalnie): curl -X POST -H "Authorization: Bearer moja_tajna_token" "http://IP:8080/torch?action=on"


---

### narazie jeszcze bez tego

---

przetesotanie flasku ze skryptem z pythona

Poniżej masz dwie rzeczy: 1) zmodyfikowany plik send_photo_network.py (Twój skrypt z wysyłaniem zdjęć do Flask na laptopie), z komentarzami; 2) krótka lista poleceń do uruchomienia i testów na laptopie i w Termux.
Zapisz plik jako send_photo_network.py w Termux i dostosuj DEST_URL oraz AUTH_TOKEN przed uruchomieniem.
send_photo_network.py

```

#!/data/data/com.termux/files/usr/bin/env python3
"""
send_photo_network.py
Cykliczne robienie zdjęć w Termux i wysyłanie POSTem do serwera Flask.
- Ustaw DEST_URL na http://<LAPTOP_IP>:5000/upload
- AUTH_TOKEN musi pasować do serwera Flask
- Wymaga: termux-api, python requests, opcjonalnie pillow
"""
import os
import time
import datetime
from subprocess import run
import requests

# ---------- KONFIGURACJA ----------
INTERVAL = 5
TMP_DIR = os.path.expanduser("~/storage/shared/Download/pics_test")
JPEG_QUALITY = 75
MAX_FILES = 200
DEST_URL = "http://192.168.1.100:5000/upload"  # <- zamień na IP laptopa w sieci lokalnej
AUTH_TOKEN = "moja_tajna_token"               # <- ten sam token co we Flask
# -----------------------------------

os.makedirs(TMP_DIR, exist_ok=True)

def now_ts():
    return datetime.datetime.utcnow().strftime("%Y%m%dT%H%M%SZ")

def take_photo(path):
    """
    Wywołuje termux-camera-photo i zwraca (ok, stdout, stderr).
    """
    try:
        res = run(["termux-camera-photo", path], capture_output=True, text=True)
        ok = res.returncode == 0
        return ok, res.stdout, res.stderr
    except FileNotFoundError:
        return False, "", "termux-camera-photo not found"
    except Exception as e:
        return False, "", str(e)

def compress_jpeg(path, quality):
    """
    Jeśli Pillow dostępne — zmniejsza jakość JPEG.
    """
    try:
        from PIL import Image
    except Exception:
        return False
    try:
        img = Image.open(path).convert("RGB")
        img.save(path, "JPEG", quality=quality)
        return True
    except Exception as e:
        print("Błąd kompresji:", e)
        return False

def rotate_old_files(directory, max_files):
    try:
        files = [os.path.join(directory, f) for f in os.listdir(directory) if os.path.isfile(os.path.join(directory, f))]
        if len(files) <= max_files:
            return
        files.sort(key=lambda p: os.path.getmtime(p))
        to_delete = files[:len(files) - max_files]
        for p in to_delete:
            try:
                os.remove(p)
                print(f"Usunięto stary plik: {p}")
            except Exception as e:
                print("Błąd usuwania pliku:", p, e)
    except Exception as e:
        print("Błąd w rotate_old_files:", e)

def send_file(path, token):
    """
    Wysyła plik POSTem multipart/form-data z nagłówkiem Authorization.
    Zwraca tuple (ok:bool, status_code:int, text:str).
    """
    try:
        with open(path, "rb") as fp:
            files = {"file": (os.path.basename(path), fp, "image/jpeg")}
            headers = {"Authorization": f"Bearer {token}"}
            r = requests.post(DEST_URL, files=files, headers=headers, timeout=20)
            return (r.status_code == 200, r.status_code, r.text)
    except Exception as e:
        return (False, None, str(e))

def validate_config():
    global INTERVAL, MAX_FILES
    if not isinstance(INTERVAL, (int, float)) or INTERVAL < 1:
        INTERVAL = 1
    if not isinstance(MAX_FILES, int) or MAX_FILES < 1:
        MAX_FILES = 100

def main_loop():
    validate_config()
    print(f"[{now_ts()}] Start. Zapisywanie do: {TMP_DIR} -> wysyłanie do: {DEST_URL}")
    while True:
        ts = now_ts()
        filename = f"photo_{ts}.jpg"
        path = os.path.join(TMP_DIR, filename)

        print(f"[{ts}] Robię zdjęcie -> {filename}")
        ok, out, err = take_photo(path)
        if not ok:
            print(f"[{ts}] termux-camera-photo failed. stderr: {err.strip()}")
            time.sleep(INTERVAL)
            continue

        if not os.path.isfile(path):
            print(f"[{ts}] Plik nie istnieje po termux-camera-photo. stdout: {out.strip()} stderr: {err.strip()}")
            time.sleep(INTERVAL)
            continue

        if compress_jpeg(path, JPEG_QUALITY):
            print(f"[{ts}] Skompresowano {filename} (quality={JPEG_QUALITY})")
        else:
            print(f"[{ts}] Kompresja pominięta lub nieudana.")

        # Wyślij plik do serwera Flask
        ok_send, code, text = send_file(path, AUTH_TOKEN)
        if ok_send:
            print(f"[{ts}] Upload OK (HTTP {code})")
            # opcjonalnie usuwamy po sukcesie:
            # os.remove(path)
        else:
            print(f"[{ts}] Upload failed (code={code}) text={text}")

        rotate_old_files(TMP_DIR, MAX_FILES)

        try:
            size = os.path.getsize(path)
            print(f"[{ts}] Zapisano {filename} — {size} bajtów")
        except Exception:
            pass

        time.sleep(INTERVAL)

if __name__ == "__main__":
    try:
        main_loop()
    except KeyboardInterrupt:
        print(f"[{now_ts()}] Zatrzymano test (CTRL+C).")


```
Lista poleceń do szybkiego testu (laptop — Linux) i w Termux
Na laptopie (przygotowanie i uruchomienie Flask):

- Utwórz katalog projektu: mkdir ~/photo_server && cd ~/photo_server
(opcjonalnie) virtualenv: python3 -m venv venv source venv/bin/activate

- Zainstaluj Flask: pip install flask
- Stwórz server.py (użyj wcześniej podanego prostego Flask z AUTH_TOKEN = "moja_tajna_token").
- Uruchom serwer: python3 server.py
- Sprawdź IP laptopa w sieci lokalnej: ip a # lub: hostname -I zapamiętaj np. 192.168.1.100

W Termux (na Androidzie):

- Zainstaluj wymagania: pkg update && pkg upgrade pkg install python termux-api pip install requests pillow # pillow opcjonalnie
- Zmień DEST_URL w send_photo_network.py na http://<LAPTOP_IP>:5000/upload i AUTH_TOKEN na ten sam token co we Flask.
- Nadaj prawa i uruchom: chmod +x send_photo_network.py termux-wake-lock python3 send_photo_network.py
- Alternatywny manualny test wysyłki z Termux (po zrobieniu przykładowego zdjęcia): curl -v -F "file=@/storage/emulated/0/Download/pics_test/.jpg" -H "Authorization: - Bearer moja_tajna_token" http://<LAPTOP_IP>:5000/upload

Debug:
- Jeśli curl z Termux nie dociera: sprawdź firewall na laptopie: sudo ufw allow 5000/tcp
- Sprawdź, czy Flask nasłuchuje na 0.0.0.0 i porcie 5000.
- Sprawdź, czy laptop i telefon są w tej samej sieci i możesz pingować laptop z telefonu (w Termux: ping <LAPTOP_IP>).
- To powinno dać Ci działający end-to-end: Termux robi zdjęcia i wysyła je do folderu received na laptopie. Jeśli chcesz, mogę przygotować też ready-to-run server.py (jeśli potrzebujesz przypomnienia) lub pomóc z konfiguracją firewalla/systemd.