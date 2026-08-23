

## Test skryptu uruchamiającego aparat po otrzymaniu /endpointu z komendą 

<div style="text-align: center;">

```mermaid
graph LR
A(Server = Android) -- payload--> B(Client = Desktop)
    B -- /endpoint zrob zdjecie --> A
```
</div>



!!! note annotate "Przewodnik: Automatyczny Aparat w Termux (Krok po Kroku)"

    Ten skrypt sprawi, że Twój telefon będzie automatycznie robił zdjęcia co kilka sekund i zapisywał je bezpośrednio w pamięci telefonu, dzięki czemu zobaczysz je w zwykłej systemowej Galerii.



### Krok 1: Przygotowanie Telefonu (Uprawnienia)

*Przed rozpoczęciem pracy aplikacja Termux musi dostać uprawnienia do dostępu do aparatu oraz pamięci*

- Wejdź w Ustawienia Androida.
- Znajdź Aplikacje -> Termux.
- Wejdź w Uprawnienia (Permissions).
- Zezwól na Aparat oraz Pliki i multimedia.

### Krok 2: Konfiguracja Termux
f
*Otwórz Termux i wpisz kolejno poniższe komendy (zatwierdzaj każdą klawiszem Enter):*

```
# 1. Połącz Termux z pamięcią telefonu (ważne!)
termux-setup-storage

# 2. Zaktualizuj listę pakietów
pkg update && pkg upgrade

# 3. Zainstaluj Pythona i dodatki Termux API
pkg install python termux-api

# 4. Zainstaluj bibliotekę do obsługi obrazów (opcjonalne, do kompresji)
pip install pillow
```
*Przy komendzie ```termux-setup-storage``` na ekranie wyskoczy okno wiadomości Androida – kliknij Zezwól.*

### Krok 3: Stworzenie skryptu

*Stwórz plik o nazwie ```foto.py``` i dodaj do niego kod:*



#### 1. Utworzenie pliku:

*Najszybciej zrobisz to komendą nano (edytor tekstu):*

``` nano photo1.py```

#### 2. Kod programu:

```
import os, time, datetime, subprocess

# --- USTAWIENIA ---
SEKUNDY = 5                             # Zdjęcie co 5 sekund
FOLDER = os.path.expanduser("~/storage/pictures/TermuxPhotos") 
MAX_ZDJEC = 50                          # Usuwa najstarsze po przekroczeniu 50 sztuk
JAKOSC = 75                             # Kompresja (1-100)

os.makedirs(FOLDER, exist_ok=True)

def rob_zdjecie():
    ts = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
    sciezka = os.path.join(FOLDER, f"IMG_{ts}.jpg")
    
    # Robienie zdjęcia (tylny aparat: -c 0)
    wynik = subprocess.run(["termux-camera-photo", "-c", "0", sciezka], capture_output=True)
    
    if wynik.returncode == 0:
        print(f"[{ts}] Zdjęcie zapisane w Galerii!")
        # Próba kompresji jeśli jest Pillow
        try:
            from PIL import Image
            img = Image.open(sciezka)
            img.save(sciezka, "JPEG", quality=JAKOSC)
        except: pass
    else:
        print(f"Błąd aparatu: {wynik.stderr.decode()}")

def sprzataj():
    pliki = [os.path.join(FOLDER, f) for f in os.listdir(FOLDER)]
    if len(pliki) > MAX_ZDJEC:
        pliki.sort(key=os.path.getmtime)
        for i in range(len(pliki) - MAX_ZDJEC):
            os.remove(pliki[i])

if __name__ == "__main__":
    print(f"Start! Zdjęcia trafiają do folderu Pictures/TermuxPhotos")
    try:
        while True:
            rob_zdjecie()
            sprzataj()
            time.sleep(SEKUNDY)
    except KeyboardInterrupt:
        print("Zatrzymano skrypt.")
```

*Zapisz plik: naciśnij ```CTRL + O```, potem Enter, a na koniec ```CTRL + X```, aby wyjść.*

#### 3. Wyjaśnienie funkcji:

| Zmienna / Funkcja | Wyjaśnienie: | 
| :------------------: | :-------------------------: | 
|SEKUNDY = 5:  |To Twój "zegar". Zmieniając tę liczbę, ustalasz, jak często telefon ma robić zdjęcie.|
|FOLDER = ...:  | Wskazuje systemowi, gdzie dokładnie ma wrzucać pliki. Dzięki przedrostkowi ```storage/pictures```, zdjęcia trafiają do publicznej części pamięci telefonu, a nie zostają ukryte wewnątrz Termuxa. |
|def rob_zdjecie():: | To serce skryptu. Funkcja generuje unikalną nazwę pliku na podstawie aktualnej daty i godziny ```(strftime)```.  Wysyła komendę do systemu Android ```(termux-camera-photo)```, aby ten uruchomił fizyczny aparat i zapisał obraz. ```-c 0:``` Ten parametr oznacza tylny aparat. Jeśli zmienisz go na ```-c 1```, skrypt zacznie robić zdjęcia przednią kamerą (selfie).|
|def sprzataj(): | Funkcja dba o to, by nie zapchać pamięci telefonu. Sprawdza, ile plików jest w folderze i jeśli przekroczysz limit ```MAX_ZDJEC```, automatycznie usuwa najstarsze fotografie.|
|while True: | o nieskończona pętla. Powoduje, że skrypt nie kończy pracy po jednym zdjęciu, ale działa "w kółko", aż sam go nie wyłączysz ```(skrótem CTRL + C)```. |



#### 4. Lokalizacja pliku

Skrypt znajduje się w katalogu domowym Termuxa pod ścieżką:

```/data/data/com.termux/files/home/photo1.py```


### Krok 4: Uruchomienie
Aby wystartować aparat, wpisz:

``` python foto.py ```

!!! note annotate "Ważne"

    Skrypt musi być uruchomiony wewnątrz Termuxa. Dzięki wcześniejszej komendzie ```termux-setup-storage```, skrypt ma uprawnienia do zapisu w folderze dzielonym. Oznacza to, że zdjęcia nie zostają "uwięzione" wewnątrz Termuxa, ale trafiają do pamięci telefonu dostępnej dla każdej aplikacji Androida (np. Twojej systemowej Galerii czy Menadżera Plików).





### Jak przeglądać zdjęcia?

Zdjęcia są zapisywane w pamięci wewnętrznej telefonu. Możesz je znaleźć w następujący sposób:

  1. Przez Galerię: Otwórz aplikację Galeria lub Zdjęcia Google. Szukaj albumu o nazwie **"TermuxPhotos".**
  2. Przez Menadżer Plików: Przejdź do folderu: ```Pamięć wewnętrzna > Pictures > TermuxPhotos```
  3.  Ścieżka systemowa: Wewnątrz Androida jest to lokalizacja ```/storage/emulated/0/Pictures/TermuxPhotos.```

!!! note annotate "Uzasadnienie:"
    W kodzie Python znajduje się linia: ```FOLDER = os.path.expanduser("~/storage/pictures/TermuxPhotos")```. W Termuxie skrót ```~/storage/pictures``` to symboliczne dowiązanie (symlink) właśnie do systemowego folderu ```Pictures```, co sprawia, że zdjęcia są natychmiast indeksowane przez Androida.




### 6. Rozwiązywanie problemów (FAQ)

| Błąd | Zachowanie | 
| :------------------: | :-------------------------: | 
|Jak przerwać? |Naciśnij CTRL + C w Termuxie.|
|Aparat nie działa? | Upewnij się, że masz zainstalowaną aplikację Termux:API ze sklepu (F-Droid lub tego, z którego pobrałeś Termux).|
|Działanie przy zablokowanym ekranie:| Przed uruchomieniem wpisz termux-wake-lock, aby Android nie "uśpił" skryptu po zablokowaniu telefonu.|


