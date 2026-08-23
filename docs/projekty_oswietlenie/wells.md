

# W.E.L.L.S - Bezprzewodowy System Monitorowania Oświetlenia Awaryjnego

<div style="text-align: center; margin: 20px 0;"><img src="/img-and-mp4/projekty_praca/wells-topologia.png" alt="Zdjecie" style="width: 100%; max-width: 90%; height: auto; display: inline-block;"></div>





!!! note "Czym jest WELLS i do czego służy?"
    - To bezprzewodowy system monitorowania oświetlenia awaryjnego w budynkach. Umożliwia zaplanowanie harmonogramu testów dla wszystkich połączonych z nim opraw oświetlenia.

    - System pozwala na zarządzanie stanem pracy oprawy: (stan wstrzymania, stan spoczynku, stan testowy, stan awarii) oraz otrzymywanie raportów o stanie opraw





<div style="text-align: center; margin: 20px 0;"><img src="/img-and-mp4/projekty_praca/wells-front.png" alt="Zdjecie" style="width: 100%; max-width: 60%; height: auto; display: inline-block;"></div>


## Centrala Sterująca Systemu WELLS:

Centrala wyposażona jest w intuicyjny panel dotykowy, który stanowi serce zarządzania bezprzewodowym systemem oświetlenia awaryjnego. Komunikacja z oprawami odbywa się radiowo poprzez moduł IQRF. Urządzenie umożliwia szybkie parowanie opraw z centralą, elastyczne przypisywanie ich do grup oraz pełną automatyzację procesów kontrolnych.


- **TEST (Funkcjonalny / Autonomii):** Zdalne wyzwalanie obowiązkowych testów sprawności źródła światła oraz czasu pracy baterii.

- **REST (Tryb spoczynkowy):** Wyłączenie oświetlenia awaryjnego podczas planowanych przerw w użytkowaniu budynku (oszczędność akumulatorów).

- **INHIBIT (Blokada):** Tymczasowe zablokowanie przejścia opraw w tryb awaryjny (np. na czas prac remontowych).

- **AUTONOMIA:** Monitorowanie i weryfikacja projektowego czasu świecenia oprawy z baterii (np. 1h/2h/3h).

- **AWARIA / STATUS:** Natychmiastowa identyfikacja błędów (uszkodzenie źródła LED, awaria ładowania, zużyty akumulator).

- **HARMONOGRAM:** Automatyczne planowanie testów zgodnie z przepisami prawa budowlanego.



---

## Na czym polegałą moja rola w tym projekcie:


*Za co byłem odpowiedzialny:*


- **Konfiguracja i administracja systemem:** Zdalne zarządzanie jednostkami centralnymi na bazie linux poprzez protokół SSH, w tym konfiguracja uprawnień użytkowników i zarządzanie dostępem do haseł,

- **Bezpieczeństwo i utrzymanie sieci (Cybersecurity & Maintenance):** Zabezpieczanie komunikacji sieciowej poprzez konfigurację firewalla (UFW – zarządzanie portami) oraz regularne wykonywanie aktualizacji oprogramowania układowego i systemowego,

- **Testowanie i walidacja komunikacji (QA / Testy):** Opracowywanie, wdrażanie oraz realizacja procedur testowych (kart testowych) weryfikujących poprawność bezprzewodowej komunikacji radiowej między oprawami a centralą systemu,

- **Integracja urządzeń i automatyzacja:** Przeprowadzanie procedur parowania opraw oświetleniowych z centralą, w tym tworzenie sekwencji testowych oraz przygotowywanie kompletnej dokumentacji technicznej i wykonawczej,

- **Prace wdrożeniowe i serwisowe (Field Engineering):**  Prowadzenie operacji serwisowych i uruchomień systemów komunikacji oświetlenia awaryjnego bezpośrednio na obiektach budowlanych.
