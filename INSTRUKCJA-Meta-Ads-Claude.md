# Grancommerce × Claude: kampanie Meta Ads od A do Z

Instrukcja wdrożenia ekosystemu: **Claude (Claude Code) + Meta Ads + Google Drive + Higgsfield + Canva + CapCut**.

> **Uwaga o statusie informacji.** Instrukcja powstała bez dostępu do treści webinaru, na podstawie opisu i wiedzy o narzędziach. Punkty oznaczone ⚠️ **WERYFIKUJ** dotyczą integracji, które szybko się zmieniają (nazwy serwerów MCP, dostępność w planach, zakres uprawnień). Sprawdź je w dokumentacji przed wdrożeniem u klienta.

---

## 0. Architektura – jak to działa

```
                    ┌────────────────────────┐
                    │  Claude Code (agent)   │  ← Ty + zespół dajecie polecenia
                    │  + CLAUDE.md, skille   │
                    └───────────┬────────────┘
     ┌──────────┬───────────────┼──────────────┬──────────────┐
     ▼          ▼               ▼              ▼              ▼
  Meta Ads   Google Drive    Canva        Higgsfield      CapCut
 (MCP/API)   (konektor)     (konektor)   (MCP/API)      (ręcznie / API)
 kampanie,   brandbook,     grafiki,     wideo AI,      montaż,
 analityka   packshoty,     infografiki, animacje       napisy,
             brief          statyczne                    finalizacja
```

Zasada: **Claude jest mózgiem i orkiestratorem**. Materiały klienta czyta z Drive, kreacje zleca do Canvy/Higgsfield, kampanie zakłada i analizuje przez Meta. **CapCut zwykle zostaje etapem ręcznym** (brak oficjalnej integracji ⚠️ WERYFIKUJ).

---

## 1. Przygotowanie (jednorazowo dla agencji)

### 1.1 Konta i dostępy – checklist
- [ ] **Claude**: plan Pro/Max lub Team/Enterprise (dla zespołu rekomendowany Team – wspólne konektory i zarządzanie dostępami).
- [ ] **Claude Code** zainstalowany (terminal, VS Code lub claude.ai/code).
- [ ] **Meta Business Manager (Business Suite) agencji** z rolą administratora.
- [ ] Dostęp do kont reklamowych klientów **przez Business Manager** (partner access, NIE przez logowanie na dane klienta).
- [ ] **Google Workspace** agencji + współdzielony Dysk (Shared Drive).
- [ ] **Canva** – plan Pro/Teams (Brand Kit i konektor wymagają płatnego planu ⚠️ WERYFIKUJ).
- [ ] **Higgsfield** – płatny plan z dostępem do API/MCP ⚠️ WERYFIKUJ.
- [ ] **CapCut** – konto (CapCut Pro/Business do pracy zespołowej).

### 1.2 Bezpieczeństwo (zrób to PRZED podpięciem Meta)
1. Osobny **użytkownik systemowy w Business Managerze** dla integracji (nie prywatne konto pracownika).
2. **Zasada najmniejszych uprawnień**: na start token tylko do **odczytu** (`ads_read`), zapis (`ads_management`) dopiero po testach.
3. Tokeny trzymaj w menedżerze haseł / zmiennych środowiskowych – **nigdy w repozytorium**. Dodaj `.env` do `.gitignore`.
4. **Reguła bezpieczeństwa kampanii**: wszystkie tworzone przez Claude kampanie/reklamy mają status `PAUSED`. Włączenie = człowiek.
5. Ustal **limity budżetów** po stronie konta reklamowego (spending limit) jako siatkę bezpieczeństwa.
6. Zgoda klienta (umowa/aneks) na przetwarzanie materiałów i danych przez narzędzia AI.

---

## 2. Krok 1 – Połączenie Claude z Meta

Dwie ścieżki. Wybierz jedną.

### Ścieżka A – gotowy serwer MCP dla Meta Ads (szybsza)
⚠️ WERYFIKUJ aktualną, wspieraną opcję: sprawdź oficjalny katalog konektorów Claude (Ustawienia → Konektory) oraz rejestr MCP. Jeśli Meta udostępnia oficjalny serwer MCP dla Ads, użyj go. W przeciwnym razie użyj sprawdzonego serwera społecznościowego (przejrzyj kod i uprawnienia przed użyciem – to dostęp do pieniędzy klienta).

Dodanie serwera MCP w Claude Code (schemat):
```bash
claude mcp add meta-ads --scope project -- <komenda_uruchamiająca_serwer>
# zmienne: META_ACCESS_TOKEN, META_AD_ACCOUNT_ID
claude mcp list        # sprawdź, czy serwer widoczny
```

### Ścieżka B – własna integracja przez Meta Marketing API (pełna kontrola)
1. Wejdź na **developers.facebook.com** → *Moje aplikacje* → **Utwórz aplikację** (typ: Business).
2. Dodaj produkt **Marketing API**.
3. Połącz aplikację z Business Managerem agencji.
4. Utwórz **System User** (Business Settings → Users → System Users) i przypisz mu konta reklamowe klientów.
5. Wygeneruj **token systemowy** z uprawnieniami: `ads_read` (start), potem `ads_management`, `business_management`.
6. Do zaawansowanego dostępu (>25 wywołań/godz. na wielu kontach) złóż **App Review / Advanced Access** ⚠️ WERYFIKUJ wymogi.
7. Napisz (lub zleć Claude'owi napisanie) cienki serwer MCP / skrypty wokół endpointów:
   - `GET /act_<id>/campaigns`, `/adsets`, `/ads`, `/insights`
   - `POST /act_<id>/campaigns`, `/adsets`, `/adcreatives`, `/ads`
8. Zapisz dane w `.env`:
```
META_ACCESS_TOKEN=...
META_APP_ID=...
META_APP_SECRET=...
```

### Test połączenia (obie ścieżki)
W Claude Code wpisz:
> „Pobierz listę kampanii z konta reklamowego [ID] z ostatnich 30 dni i pokaż wydatki, CTR, CPC, ROAS w tabeli.”

Jeśli dostajesz dane – krok 1 zaliczony. Dopiero potem test zapisu: „Utwórz **wstrzymaną** kampanię testową o budżecie 10 zł/dzień.”

---

## 3. Krok 2 – Google Drive (baza wiedzy klienta)

### 3.1 Połączenie
1. Claude (claude.ai) → **Ustawienia → Konektory → Google Drive → Połącz** (OAuth kontem agencji).
2. W Claude Code: dodaj konektor/MCP Drive (`claude mcp add ...` lub konektor z katalogu) ⚠️ WERYFIKUJ.
3. Nadaj dostęp **tylko do folderów klientów**, nie do całego Dysku.

### 3.2 Standardowa struktura folderu klienta (WYMUSZAJ ją dla każdego klienta)
```
/Klienci/[NAZWA KLIENTA]/
├── 00_Brief/               (brief, cele, KPI, budżety, persony)
├── 01_Brandbook/           (logo, kolory, fonty, tone of voice, zasady)
├── 02_Packshoty/           (zdjęcia produktowe – nazwy: SKU_wariant.jpg)
├── 03_Zdjecia_lifestyle/
├── 04_Inspiracje/          (reklamy konkurencji, moodboardy, referencje)
├── 05_Kreacje_gotowe/      (archiwum poprzednich reklam + wyniki)
├── 06_Raporty/             (raporty Claude, eksporty z Meta)
└── 07_Nowe_kampanie/       (wyjście: warianty, teksty, grafiki do akceptacji)
```
**Klucz:** dobre nazwy plików i krótki plik `README.md` w folderze klienta z opisem, co gdzie leży.

### 3.3 Brief-master (najważniejszy plik)
Dla każdego klienta utwórz `00_Brief/brief-master.md`:
- Oferta, USP, grupa docelowa (persony), pain points
- Cele (ROAS/CPA/leady), sezonowość, budżet
- Tone of voice: „mówimy / nie mówimy”
- Zakazy (branżowe, prawne, np. zdrowie/finanse – polityki Meta)
- Linki: sklep, landing page, piksel/CAPI status

### 3.4 Test
> „Przeczytaj folder klienta X na Dysku: brief i brandbook. Streść w 10 punktach markę, grupę docelową i zasady wizualne.”

---

## 4. Krok 3 – Canva (grafiki i infografiki)

1. Claude → Konektory → **Canva → Połącz** (OAuth) ⚠️ WERYFIKUJ dostępność w Twoim planie.
2. W Canvie: skonfiguruj **Brand Kit** klienta (logo, kolory, fonty) – osobny na klienta.
3. Przygotuj **szablony** formatów Meta (biblioteka szablonów agencji):
   - Feed 1:1 (1080×1080), 4:5 (1080×1350)
   - Stories/Reels 9:16 (1080×1920)
   - Karuzela (1080×1080 × 3–10 kart)
4. Polecenie:
> „Na podstawie briefu i brandbooka klienta X stwórz w Canvie 3 warianty grafiki 4:5 dla produktu [SKU] z packshotem z Dysku. Kąty komunikacji: (1) cena/promocja, (2) korzyść, (3) social proof.”
5. Eksport PNG/JPG → wgraj do `07_Nowe_kampanie/` na Dysku.

---

## 5. Krok 4 – Higgsfield (wideo i animacje AI)

1. Załóż konto Higgsfield, wybierz plan z API/MCP ⚠️ WERYFIKUJ; wygeneruj klucz API.
2. Podepnij przez MCP/API do Claude Code (`claude mcp add higgsfield ...` lub skrypt wywołujący API) ⚠️ WERYFIKUJ.
3. Typowe zastosowania w Meta Ads:
   - Animacja packshotu (obrót, ruch kamery, „unboxing”) 5–10 s
   - Wideo lifestyle z produktem
   - Hooki 3-sekundowe do Reels/Stories
4. **Wymogi jakości**: produkt musi wyglądać wiernie (zawsze podawaj packshot jako referencję obrazową); sprawdzaj loga/tekst na opakowaniu – AI potrafi je zniekształcać.
5. Polecenie:
> „Z packshotu [plik] wygeneruj w Higgsfield 3 animacje 9:16, 6 s: (1) powolny zoom, (2) produkt w dłoni, (3) efekt ‘reveal’. Zapisz do `07_Nowe_kampanie/wideo_raw/`.”

---

## 6. Krok 5 – CapCut (montaż)

CapCut nie ma (wg mojej wiedzy) oficjalnej, stabilnej integracji z Claude ⚠️ WERYFIKUJ. Model pracy:
1. Claude przygotowuje **scenariusz i storyboard**, listę ujęć, teksty napisów (PL), CTA, czas trwania.
2. Materiały z Higgsfield pobierasz z Dysku.
3. W CapCut montujesz: napisy, muzyka (licencjonowana), logo, format 9:16/4:5, eksport 1080p.
4. Gotowy plik → `05_Kreacje_gotowe/` na Dysku.
5. Opcja automatyzacji: napisy/montaż sterowane skryptem (FFmpeg) jako zamiennik dla powtarzalnych formatów.

---

## 7. Konfiguracja Claude Code – „mózg agencji”

W repozytorium/folderze roboczym agencji utwórz:

```
grancommerce-ai/
├── CLAUDE.md                    # zasady globalne agencji
├── clients/
│   └── [klient]/CLAUDE.md       # ID konta Meta, folder Drive, brand skrót, zakazy
├── skills/ (lub .claude/skills)
│   ├── nowa-kampania/
│   ├── warianty-kreacji/
│   ├── raport-tygodniowy/
│   ├── audyt-konta/
│   └── kontrola-polityk-meta/
└── .env  (poza gitem)
```

### Przykładowa treść `CLAUDE.md` (agencja)
```
# Zasady Grancommerce
- Język komunikacji i tekstów reklamowych: polski, chyba że klient zdecyduje inaczej.
- NIGDY nie aktywuj kampanii: wszystko tworzysz ze statusem PAUSED.
- NIGDY nie zmieniaj budżetów ani nie usuwaj obiektów bez wyraźnej zgody człowieka.
- Przed tworzeniem kreacji zawsze czytaj: brief-master + brandbook klienta z Dysku.
- Każda kreacja: min. 3 warianty, opis kąta komunikacji i hipotezy testowej.
- Sprawdzaj teksty względem polityk reklamowych Meta (roszczenia zdrowotne, przed/po, dane osobowe).
- Wyniki i raporty zapisuj w 06_Raporty na Dysku klienta.
- Nazewnictwo kampanii: [KLIENT]_[CEL]_[YYYYMM]_[KRAJ]_[TEMAT]
```

---

## 8. Workflow: jak tworzyć kampanię (A→Z)

| # | Etap | Kto | Co robi Claude |
|---|------|-----|----------------|
| 1 | Onboarding klienta | człowiek | Tworzenie struktury folderów, `CLAUDE.md` klienta |
| 2 | Research | Claude | Czyta brief/brandbook/inspiracje, analizuje dotychczasowe wyniki z Meta |
| 3 | Strategia | Claude → człowiek | Propozycja lejka, person, kątów, budżetu, KPI. **Akceptacja** |
| 4 | Copy | Claude | 5–10 wariantów nagłówków, treści głównych, CTA (pod limity Meta) |
| 5 | Kreacje statyczne | Claude + Canva | Warianty grafik/karuzel |
| 6 | Kreacje wideo | Claude + Higgsfield + CapCut | Skrypt, generacja, montaż ręczny |
| 7 | Kontrola jakości | człowiek | Zgodność z marką, polityki Meta, wierność produktu |
| 8 | Budowa kampanii | Claude + Meta | Kampania/zestawy/reklamy jako **PAUSED**, UTM-y, struktura testów |
| 9 | Akceptacja klienta | człowiek | Claude generuje podsumowanie (PDF/doc) do wysyłki |
| 10 | Uruchomienie | człowiek | Włączenie w Meta |
| 11 | Optymalizacja | Claude → człowiek | Codzienny/tygodniowy przegląd, rekomendacje |
| 12 | Raport | Claude | Raport do klienta + wnioski do następnej iteracji |

### Przykładowy prompt startowy (kampania)
> „Klient: [X]. Przeczytaj brief-master i brandbook z Dysku oraz wyniki ostatnich 90 dni z konta Meta. Zaproponuj strategię kampanii sprzedażowej na [produkt/okres]: persony, 4 kąty komunikacji, strukturę kampanii (CBO/ABO), budżet testowy i KPI. Nie twórz jeszcze niczego w Meta.”

Po akceptacji:
> „Wygeneruj 3 zestawy reklam × 3 kreacje. Teksty PL, kreacje w Canvie zgodnie z Brand Kitem, wideo w Higgsfield. Zbuduj kampanię jako PAUSED z UTM: utm_source=meta&utm_medium=paid&utm_campaign=[nazwa].”

---

## 9. Analityka i raportowanie

**Cykl tygodniowy (skill `raport-tygodniowy`):**
1. Pobranie insights: wydatki, zasięg, częstotliwość, CTR (link), CPC, CPM, CPA, ROAS, koszt dodania do koszyka.
2. Porównanie tydzień do tygodnia + względem celów.
3. Analiza kreacji: zwycięzcy/przegrani, zmęczenie kreacji (rosnąca częstotliwość + spadający CTR).
4. Rekomendacje: co skalować, co wyłączyć, jakie nowe testy.
5. Zapis raportu do `06_Raporty/` + krótkie podsumowanie dla klienta.

**Ważne:** Claude rekomenduje, człowiek wprowadza zmiany budżetowe (na początku). Zaufanie i autonomię zwiększajcie stopniowo.

---

## 10. Plan wdrożenia w agencji

### Faza 1 (tydzień 1) – Fundament
- Konta, uprawnienia, bezpieczeństwo (sekcja 1).
- Połączenie Meta w trybie **tylko odczyt**.
- Połączenie Drive; wzorcowa struktura folderów.

### Faza 2 (tydzień 2) – Pilot na 1 kliencie
- Wybierz klienta z dobrze uporządkowanymi materiałami.
- Uzupełnij brief-master i `CLAUDE.md` klienta.
- Uruchom: audyt konta + raport tygodniowy (bez zapisu do Meta).

### Faza 3 (tydzień 3) – Kreacje
- Podłącz Canvę i Higgsfield; przygotuj szablony i Brand Kit.
- Wygeneruj pierwszy pakiet wariantów, przejdź kontrolę jakości.

### Faza 4 (tydzień 4) – Zapis w Meta
- Nadaj `ads_management`; twórz kampanie jako PAUSED.
- Człowiek włącza; zmierz czas i koszt vs. dotychczasowy proces.

### Faza 5 (miesiąc 2+) – Skalowanie
- Skille (`nowa-kampania`, `audyt-konta`, itd.) dopracowane na podstawie pilota.
- Onboarding kolejnych klientów wg szablonu.
- Szkolenie zespołu, wewnętrzna „biblioteka promptów”.
- Comiesięczny przegląd: co działa, koszty narzędzi, ryzyka.

---

## 11. Ryzyka i dobre praktyki

| Ryzyko | Zabezpieczenie |
|---|---|
| Przypadkowe wydanie budżetu | PAUSED domyślnie, spending limit, zapis dopiero po akceptacji |
| Odrzucenie reklam / blokada konta | Skill kontroli polityk, brak roszczeń zdrowotnych/finansowych bez weryfikacji |
| Halucynacje danych | Każda liczba w raporcie z API; pytaj o źródło i zakres dat |
| Zniekształcony produkt w AI | Packshot jako referencja, kontrola człowieka |
| Wyciek danych klienta | Osobne foldery, minimalne uprawnienia, zgody w umowach |
| Prawa autorskie (muzyka, fonty, AI) | Licencjonowane zasoby, sprawdzenie warunków narzędzi |
| Oznaczanie treści AI | Sprawdź aktualne wymogi Meta dot. oznaczania treści generowanych/edytowanych przez AI |
| Uzależnienie od jednego narzędzia | Dokumentuj procesy w skillach i `CLAUDE.md`, nie w głowach |

---

## 12. Co zrobić jako pierwsze (dziś)

1. Utwórz System Usera i wygeneruj token **tylko do odczytu**.
2. Podepnij Meta do Claude Code, wykonaj test „lista kampanii”.
3. Podepnij Drive, ustandaryzuj folder jednego klienta pilotażowego.
4. Napisz `CLAUDE.md` agencji i klienta pilotażowego.
5. Uruchom pierwszy audyt konta i raport.

Dalsze kroki (Canva → Higgsfield → zapis w Meta) dopiero po zaliczeniu powyższych.
