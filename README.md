# Ordo.ai-Architecture-showcase

Note: The source code for this project is maintained in a private repository due to active commercialization efforts. This repository serves strictly as an Architectural Showcase demonstrating the system design, data flow, and technology stack.

**Ordo.ai** to hybrydowy, wydajny system wspierający menedżerów w automatycznym układaniu harmonogramów czasu pracy. Łączy moc elastycznej interpretacji języka naturalnego przez modele LLM (przed-procesor AI) z błyskawicznym, w 100% deterministycznym silnikiem algorytmicznym w Pythonie (**Solver**) oraz transakcyjną bazą danych PostgreSQL.

---

## Główne Cechy i Rozwiązywane Problemy

* **Brak Halucynacji i Determinizm**: Model językowy nie układa grafiku bezpośrednio. Odpowiada jedynie za wyciągnięcie intencji z tekstu. Sam grafik wyliczany jest przez matematyczno-logiczny Solver w Pythonie w czasie `< 0.1s`.
* **Zgodność z Kodeksem Pracy i Regułami Biznesowymi**:
  * Zapobieganie nadmiernemu zmęczeniu (maksymalnie 5 dni pracy z rzędu, potem obowiązkowy reset).
  * Wykluczanie niedziel niehandlowych (automatyczny symbol `'W'`).
  * Uwzględnianie urlopów zatwierdzonych w bazie danych (symbol `'U'`).
  * Gwarancja stałej obsady stanowiskowej (np. dokładnie 2 osoby na zmianę poranną **R** i 2 osoby na popołudniową **P**).
* **Sprawiedliwe Rozkładanie Obciążenia**: Algorytm dynamicznie śledzi liczbę przepracowanych dni w miesiącu i priorytetyzuje przydział pracy dla osób z najmniejszym obciążeniem.
* **Przetwarzanie Mowy Potocznej (AI Preprocessor)**: Menedżer może wpisać dowolne uwagi (np. *"Iwona urlop 10-15 października, Agnieszka w piątek na rano"*), które AI zamienia na strukturalne reguły JSON.
* **Pełny Cykl Życia Danych**: Od wpisania uwag we frontendzie Streamlit, przez walidację Pydantic w FastAPI, zapis transakcyjny w PostgreSQL, aż po wygenerowanie oficjalnego pliku PDF do druku.

---

## Stack Technologiczny

| Warstwa | Technologia / Biblioteka | Rola w Systemie |
| :--- | :--- | :--- |
| **Frontend** | `Streamlit`, `Pandas` | Interfejs użytkownika, wprowadzanie uwag, podgląd tabelaryczny, statystyki obsady |
| **Backend API** | `FastAPI`, `Uvicorn`, `Pydantic` | Asynchroniczny serwer REST API, walidacja typów, orkiestracja przepływu |
| **AI Preprocessor** | `Groq API` (`llama-3.3-70b-versatile`) | Ekstrakcja intencji menedżera z języka naturalnego do czystego JSON |
| **Engine / Solver** | `Python 3.12` (Custom Constraint Engine) | Deterministyczne układanie grafiku, egzekwowanie norm prawnych i obsady |
| **Baza Danych** | `PostgreSQL` / `Supabase`, `psycopg2` | Trwałe przechowywanie pracowników, ról, urlopów oraz zapisanych zmian |
| **Raportowanie PDF** | `Jinja2`, `WeasyPrint`, `HTML5/CSS3` | Renderowanie szablonu HTML i kompilacja do oficjalnego pliku PDF |

---

## Przepływ Danych w Systemie (Data Flow)

```text
[UŻYTKOWNIK / STREAMLIT (frontend.py)]
       │
       ├─► 1. Wpisanie uwag w języku naturalnym + wybór Roku/Miesiąca
       │     │
       │     └─► POST /generate_schedule/ (FastAPI main.py)
       │           │
       │           ├─► A. Wywołanie ai_agent.py (preprocess_input)
       │           │     └─► Groq API (Llama-3.3-70B) ──► Zwraca czysty JSON
       │           │
       │           └─► B. Wywołanie solver.py (generate_schedule)
       │                 ├─► Pobiera pracowników i urlopy z DB (database.py)
       │                 ├─► Nakłada sztywne reguły z AI (forced_shifts)
       │                 └─► Generuje harmonogram w < 0.1s i zwraca słownik
       │
       ├─► 2. Podgląd tabeli DataFrame + Statystyki obsady na ekranie
       │
       ├─► 3. Kliknięcie "💾 Zapisz Grafik w Bazie"
       │     │
       │     └─► POST /save_schedule/ (FastAPI main.py)
       │           └─► Przekładanie symboli ('R' -> 06:00-14:00, 'P' -> 14:00-22:00)
       │                 oraz zapis w transakcji atomowej do PostgreSQL
       │
       └─► 4. Kliknięcie "📄 Pobierz Harmonogram PDF"
             │
             └─► GET /raporty/harmonogram/{year}/{month} (FastAPI main.py)
                   └─► Pobranie zapisanych zmian z DB ──► Jinja2 Template ──► WeasyPrint ──► Plik PDF
```

---

## Wyjaśnienie Architektury Modułowej

### 1. `frontend.py` (Interfejs Streamlit)
Odpowiada za komunikację z menedżerem. Zawiera panel boczny do wyboru roku/miesiąca, pole tekstowe `st.text_area` na niestandardowe uwagi oraz zestaw przycisków akcji. Wyświetla wygenerowany grafik w postaci tabeli `pandas.DataFrame` z automatycznym podliczeniem podsumowania zmian (`R`, `P`, `W`, `U`).

### 2. `main.py` & `schemas.py` (FastAPI + Pydantic)
Serwer REST API zapewniający bezpieczne punkty końcowe:
* **`POST /generate_schedule/`**: Przyjmuje obiekt `ScheduleRequest(year, month, instructions)`. Jeśli podano uwagi, wywołuje przed-procesor AI, a następnie przekazuje dane do Solvera.
* **`POST /save_schedule/`**: Przyjmuje wygenerowany grafik, mapuje symbole na pełne daty i godziny rozpoczęcia/zakończenia pracy, i wykonuje hurtowy zapis do tabeli `zmiana` z obsługą `rollback` w razie błędów.
* **`GET /raporty/harmonogram/{year}/{month}`**: Pobiera z bazy utwalone zmiany, wylicza sumy godzin dla każdego pracownika, generuje strukturę pod szablon HTML i przekazuje ją do `WeasyPrint` w celu wygenerowania pliku PDF.

### 3. `ai_agent.py` & `prompts.py` (Przed-procesor AI)
Lekki moduł interpretujący mowę potoczną. Wykorzystuje surowy prompt systemowy z `prompts.py` wymuszający na modelu Llama-3.3-70B zwrócenie wyłącznie poprawnie sformatowanego obiektu JSON zawierającego dwie tablice:
* `urlopy`: lista obiektów z imieniem i nazwiskiem oraz zakresem dat (`data_start`, `data_koniec`).
* `sztywne_zmiany`: lista obiektów z imieniem, konkretną datą i wymuszoną godziną/symbolem.

### 4. `solver.py` (Deterministyczny Silnik Grafiku)
Główny algorytm aplikacji. Działa w oparciu o trzystopniową logikę:
1. **Inicjalizacja Stanu (`build_employee_tracker`)**: Pobiera dane o pracownikach i urlopach z bazy danych, budując lokalne obiekty śledzące przepracowane dni (`days_working`) oraz ciąg pracy bez przerwy (`workdays_in_a_row`).
2. **Parsowanie Ograniczeń (`forced_shifts`)**: Przekształca reguły z AI na twarde wpisy w słowniku `forced_shifts[(emp_id, date)] = symbol`.
3. **Pętla Dzień po Dniu**:
   * *Krok A*: Aplikuje twarde reguły (forced_shifts, niedziele, urlopy z bazy, limit max 5 dni z rzędu).
   * *Krok B*: Zlicza obsadę już przydzieloną w danym dniu (ile osób ma już poranek 'R' lub popołudnie 'P').
   * *Krok C*: Sortuje pozostałych dostępnych pracowników według liczby przepracowanych dni (sprawiedliwy podział obciążenia) i przydziela brakujące obsady.

### 5. `database.py` (PostgreSQL / Supabase)
Moduł połączenia z bazą danych wykorzystujący `psycopg2` oraz bezpieczne pobieranie zmiennych środowiskowych z pliku `.env`. Zapewnia spójność transakcyjną i bezpieczne zapytania SQL z parametrami chroniącymi przed SQL Injection.

---

## Instrukcja Uruchomienia (Quick Start)

### 1. Wymagania wstępne
* Python 3.12+
* Zainstalowane zależności z pliku `requirements.txt`:
  ```bash
  pip install fastapi uvicorn streamlit psycopg2-binary pandas requests pydantic weasyprint jinja2 groq python-dotenv
  ```
* Działająca baza PostgreSQL (np. na Supabase) z tabelami `pracownik`, `rola`, `urlop`, `zmiana`.
* Klucz API do serwisu Groq zapisany w pliku `.env`:
  ```env
  GROQ_API_KEY=twoj_klucz_groq
  DB_HOST=twoj_host_postgres
  DB_PORT=5432
  DB_NAME=postgres
  DB_USER=postgres
  DB_PASSWORD=twoje_haslo
  ```

### 2. Uruchomienie Serwera API (FastAPI)
W pierwszym terminalu uruchom serwer Uvicorn:
```bash
uvicorn main:app --reload
```
API będzie dostępne pod adresem: `http://127.0.0.1:8000` (Dokumentacja Swagger: `http://127.0.0.1:8000/docs`).

### 3. Uruchomienie Interfejsu (Streamlit)
W drugim terminalu uruchom aplikację frontendową:
```bash
streamlit run frontend.py
```
Aplikacja otworzy się automatycznie w przeglądarce pod adresem `http://localhost:8501`.

---

## Przykładowy Scenariusz Użycia

1. **Otwórz interfejs Streamlit** w przeglądarce.
2. W panelu bocznym wybierz **Rok: 2026** oraz **Miesiąc: 10**.
3. W polu *Niestandardowe uwagi menedżera* wpisz:
   > *"Iwona ma urlop od 12 do 15 października. Agnieszka w piątek 16.10 musi mieć zmianę poranną."*
4. Kliknij przycisk **`🚀 Wygeneruj Grafik (Solver Python)`**.
   * *Co się dzieje*: FastAPI odbiera zapytanie, AI zamienia tekst na JSON, a Solver w 0.05 sekundy układa pełny grafik, uwzględniając proszone urlopy, zmianę Agnieszki, niedziele wolne oraz normę 5 dni z rzędu.
5. Kliknij **`💾 Zapisz Grafik w Bazie PostgreSQL`**.
   * *Co się dzieje*: Symbole 'R' i 'P' zostają przeliczone na godziny (06:00-14:00 i 14:00-22:00) i bezpiecznie wpisane do tabeli `zmiana`.
6. Kliknij **`📄 Pobierz Harmonogram PDF`**.
   * *Co się dzieje*: Pobierany jest gotowy, wyrenderowany raport PDF gotowy do wydruku i powieszenia na tablicy w pracy.
