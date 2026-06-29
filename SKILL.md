---
name: competitor-recon
description: Analizuje eksport widoczności konkurenta (Senuto/Ahrefs/SEMrush — frazy + strony + ruch) i typuje, które STRONY konkurenta są najważniejsze do odtworzenia, by przejąć ich ruch. Nadaje priorytet (P1/P2/P3), main keyword, typ strony (blog/usługa/lokalna) i odsiewa frazy brandowe oraz zbyt generyczne/ambiwalentne (np. „landing", „biogram"). Użyj, gdy user da plik widoczności konkurenta i pyta, co warto odtworzyć / które strony przejąć / jak ukraść ruch konkurencji.
---

# competitor-recon

Z eksportu widoczności konkurenta robi **plan odtworzenia ruchu**: listę jego stron warte
odtworzenia (z najważniejszymi frazami każdej), uszeregowaną wg tego, ile realnego ruchu da się
przejąć. **Domyślnie domenowo-agnostyczny** — nie zakłada żadnej „naszej" strony; ocenia wartość
każdej strony na gruncie samego konkurenta. Filtr pod własną niszę włączasz OPCJONALNIE
(`--our-topics`).

## Filozofia (dlaczego tak)

Surowy wolumen myli. Najwyższe wolumeny to zwykle frazy, na które **nie ma sensu** rankować:
- **brandowe** — `kaman`, `kaman marketing` → cudzego brandu nie przejmiesz, zero wartości;
- **generyczne/ambiwalentne** — `landing` (landing page? nauka angielskiego?), `biogram`
  (życiorys/CV? bio na IG?), `dropship`, `marketing`, `ugc` → 1 słowo, ogromny wolumen,
  rozjechana intencja, konkurent i tak ledwo rankuje (poz. 27–39);
- **obcojęzyczne** — `маркетинг` (cyrylica) → szum.

Dlatego liczymy **qualified / effective traffic** (po odsianiu tych fraz), a nie raw volume.
I dlatego **podział pracy**: skrypt robi mechanikę (agregacja, reguły, scoring), a **Ty (Claude)
robisz osąd semantyczny** — bo „landing" vs „landing page" rozróżni tylko rozumienie znaczenia,
nie reguła.

## Architektura (jak skrypt klasyfikuje dane)

```
eksport.xlsx  ──ingest──►  per-fraza: flaga intencji (reguły)
                           per-strona: agregacja keyword→URL, kandydaci main_kw, typ z URL,
                                       latent_potential (open goals), winsoryzacja CPC
                           ▼
                      worksheet.md (3 pule: ruch + CPC + open-goals) ──►  TY: osąd → judgments.json
                           ▼
                        score  ──►  Opportunity Score → tiery WZGLĘDNE P1/P2/P3 + SKIP + TAIL
                                    report.md + priorities.csv
```

Reguły flagujące frazę (skrypt): `brand` / `foreign` (wykluczone z ruchu) ·
`generic_suspect` (**Ty potwierdzasz**) · `commercial` (CPC≥3 lub money-words) ·
`local` (miasto PL) · `informational` (pytania jak/co/kiedy) · `neutral`.

Composite score = `relevance × (0.6·√ruch + 0.2·szerokość_fraz + 0.2·komercja) × (1 − 0.3·KD)`,
gdzie **relevance to TWÓJ osąd wartości strony DO ODTWORZENIA** (0–1) — czy to realna strona
zarabiająca/autorytetowa konkurenta, czy raczej brand / generyk-śmieć / przynęta na ruch (zła
publiczność). **Domyślnie liczy się sam grunt konkurenta** (nie żadna „nasza" domena). `√ruch` =
malejące zwroty; CPC winsoryzowane (cap 60 zł — wyżej to błąd danych Senuto). Strona-przynęta
z dużym ruchem, ale generyczna/przypadkowa (np. „starodawne gry" w sklepie eventowym) dostaje
niski relevance i ląduje w SKIP — ruch owszem, ale przyciąga złą publiczność.

**Tiery są WZGLĘDNE** — ranking osądzonych stron (P1 = top 20%, P2 do 55%, reszta P3), nie progi
absolutne. Dzięki temu „category = P1" działa też dla małego konkurenta. **Twardy floor:**
relevance < 0.5 → max P3 (traffic-bait nie wejdzie do P1/P2 mimo ruchu); relevance < 0.3 → SKIP.
**TAIL** = strony, których NIE osądziłeś → poza rekomendacjami (nie zaśmiecają P3 brandem/generykiem
z domyślnego relevance). Dlatego **osądź wszystkie strony z worksheet**.

Helper: `.claude/skills/competitor-recon/recon_tools.py`. Odpalaj `/usr/local/bin/python3`
(popsuty venv w repo).

## Workflow

### 1. Ingest
```bash
/usr/local/bin/python3 .claude/skills/competitor-recon/recon_tools.py ingest \
    "<plik.xlsx|csv>" --domain <domena-konkurenta> --top 40 --out <DIR>
```
`--domain` możesz pominąć (auto z pierwszego URL). **Domyślnie bez żadnej „naszej" domeny** —
skill rankuje strony wg wartości do odtworzenia. **OPCJONALNIE**, gdy chcesz filtrować pod
konkretną niszę (np. analizujesz konkurenta swojej/klienta strony), dodaj
**`--our-topics "opis niszy"`** (np. `--our-topics "team building, eventy firmowe, szkolenia"`) —
wtedy strony spoza tej niszy dostaną niższy relevance. `<DIR>` w scratchpadzie albo
`recon/<konkurent>/`. Powstają: `pages.json`, `keywords.csv`, `worksheet.md`.

### 2. Osąd semantyczny → judgments.json  (TO JEST TWOJA ROBOTA)
Przeczytaj **cały `worksheet.md`** i osądź **KAŻDĄ** stronę z niego (worksheet zbiera 3 pule:
📊 duży ruch, 💰 wysoki CPC, 🎯 open-goals = mały ruch dziś, ale duży potencjał). **Nie pomijaj
🎯/💰 z małym obecnym ruchem** — to często najłatwiejsze łupy i money pages, które stary „top wg
ruchu" gubił. Czego nie osądzisz, trafi do TAIL (poza rekomendacjami). Zapisz do
`<DIR>/judgments.json` — obiekt, gdzie **klucz = dokładny URL** ze strony worksheet:

```json
{
  "kamanmarketing.pl/agencja-reklamowa-warszawa-2/": {
    "page_type": "service_local",
    "main_keyword": "agencja marketingowa warszawa",
    "relevance": 0.95,
    "junk_keywords": [],
    "content_type": "Strona usługowa lokalna /agencja-marketingowa-warszawa/",
    "note": "Money page, CPC 11+, my mamy hub /uslugi/. Wysoki priorytet."
  },
  "kamanmarketing.pl/skuteczny-biogram-na-instagramie-w-7-prostych-krokach/": {
    "page_type": "blog",
    "main_keyword": "biogram na instagramie",
    "relevance": 0.4,
    "junk_keywords": ["biogram"],
    "content_type": "Wpis blogowy (poradnik IG bio)",
    "note": "Duży ruch, ale 'biogram' samo w sobie ambiwalentne (CV/życiorys); reszta klastra OK."
  }
}
```

Pola:
- **`page_type`** — finalny typ. Usługowe/contentowe: `homepage` / `service` / `service_local`
  / `blog` / `taxonomy`. Ecommerce: `category` (strona kategorii) / `product` (karta produktu) /
  `region` (lokalizator/oddziały) / `system` (URL parametryczny, `?...`/`.php` — auto-SKIP).
  Skrypt rozpoznaje kody-sufiksy sklepów (`-cNNN`=kategoria, `-pNNN`=produkt, `-rNNN`=region)
  oraz ścieżki (`/kategoria/`, `/produkt/`…). Popraw, gdy się myli.
- **`main_keyword`** — fraza-głowa, którą strona ma celować. Skrypt podaje 2 kandydatów
  (po wolumenie / po ruchu) — **zwykle dobry, ale weryfikuj**: odrzuć fragmenty (`promowania`),
  odwróconą intencję (`jak usunąć konto` przy stronie o zakładaniu) i ambiwalentne 1-słowowce.
  Wybierz najmocniejszą, jednoznaczną, komercyjnie/tematycznie trafną frazę.
- **`relevance`** ∈ 0–1 — **wartość strony DO ODTWORZENIA**. Gate decydujące o tierze.
  **Domyślnie (bez `--our-topics`) oceniaj na gruncie samego konkurenta** — czy to strona, którą
  realnie warto u siebie odtworzyć:
  1. **Business-core vs śmieć** — 0.8–1.0 = realna strona zarabiająca/autorytetowa (kategoria,
     usługa, money page, solidny on-topic poradnik). 0.4–0.6 = peryferyjna/słaba. 0.1–0.3 = brand,
     generyk-śmieć, **przynęta na ruch** (generyczny, wysokowolumenowy temat ściągający złą
     publiczność: „starodawne gry" vol 4400, „atrakcji" vol 450000, „fluo party"), SKU konkurenta,
     system → SKIP. Duży ruch + zerowa intencja + temat przypadkowy ⇒ niski relevance. Ruch ≠ wartość.
  2. **(OPCJONALNIE, tylko gdy podano `--our-topics`) Dopasowanie do TWOJEJ niszy** — dodatkowo
     obniż relevance stronom spoza tematów, które rozwijasz (np. masz sklep z armaturą → blog HR
     konkurenta-eventowca dostaje niżej). Bez `--our-topics` ten krok pomijasz — nie zakładaj
     żadnej konkretnej branży „naszej" strony.

  **Progi (kalibruj świadomie — relevance steruje tierem):** `≥0.5` = kandydat do P1/P2 (im wyżej,
  tym wyżej w rankingu) · `0.3–0.49` = wpadnie max do P3 (twardy floor — tu ląduje traffic-bait
  „luźno pasujący") · `<0.3` = SKIP. Czyli: jeśli coś ma być realnie odpuszczone, daj <0.3; jeśli
  „może kiedyś, ale nie priorytet" — 0.3–0.49.
- **`junk_keywords`** — frazy z klastra do WYKLUCZENIA z ruchu: potwierdzone `generic_suspect`
  (jeśli realnie ambiwalentne) + każda inna brandowa/obca/bez sensu, którą reguła przepuściła.
  Wpisuj frazę 1:1 jak w worksheet. Jeśli `generic_suspect` jest jednak OK w kontekście
  (rzadko) — nie dodawaj.
- **`content_type`** — co konkretnie mamy zbudować (1 zdanie + sugerowany slug, jeśli usługowa).
- **`note`** — krótkie uzasadnienie priorytetu / haczyk / czy mamy już taką stronę.

Worksheet to już wyselekcjonowana pula (3 sygnały) — osądź ją w całości. Strony, których nie
ujmiesz w `judgments.json`, trafią do **TAIL** (długi ogon, poza rekomendacjami) — to celowe,
żeby brand/generyk z domyślnego relevance nie przeciekał do P3. Jeśli po przeczytaniu `report.md`
zobaczysz w TAIL coś z realnym ruchem/potencjałem — dorzuć do judgments i przelicz `score`.

### 3. Score
```bash
/usr/local/bin/python3 .claude/skills/competitor-recon/recon_tools.py score \
    --in <DIR> --judgments <DIR>/judgments.json --out <DIR>
```
Powstają `report.md` (prezentowalny plan w tierach P1/P2/P3/SKIP) i `priorities.csv`.

### 4. Prezentacja
Pokaż userowi **`report.md`** (albo streść top P1). Dla każdej strony P1/P2: main keyword,
typ, ruch do przejęcia, frazy wspierające, co zbudować. Wskaż wprost: które to **money pages**
(usługi/lokalne/kategorie, wysoki CPC — najpierw), które to **🎯 open goals** (mały ruch dziś,
ale konkurent słabo rankuje = najłatwiejszy łup — sekcja „Open goals" w report), które to
**topical-authority blog**, a które **odpuścić** (SKIP) i dlaczego. Zerknij na **rozkład typów**
u góry raportu — brak jakiegoś typu (np. konkurent nie ma bloga) to często niezagospodarowana luka.

## Zasady osądu (skrót, gdy się wahasz)

1. **Brand = zawsze odpuść.** Strona, która żyje tylko z brandu konkurenta → SKIP (relevance niski).
2. **1-słowowy head term z poz. >20 i rozjechaną intencją** (`dropship`, `landing`) → junk,
   nie cel. Konkurent sam nie rankuje — odtwarzanie tym bardziej bez sensu.
3. **Wysoki CPC + money-words + miasto** = priorytet #1 (taki ruch realnie konwertuje/sprzedaje).
4. **Duży ruch + CPC≈0 + intencja informacyjna** = blog topical-authority. Cenny, jeśli to realny
   temat biznesu konkurenta; jeśli przypadkowa przynęta (hobby, sprawy prywatne) → SKIP mimo ruchu.
5. **main_keyword musi DOSŁOWNIE istnieć** jako realna fraza w klastrze — nie wymyślaj.
6. Gdy strona ma frazy z kilku intencji (info + komercja), wybierz main_keyword po **wartości
   biznesowej**, nie po samym wolumenie.

### Ecommerce (gdy w danych są category/product/region)
Logika priorytetu jest inna niż dla agencji/usług:
- **`category` = money page #1.** Strony kategorii rankują na head transakcyjne („baterie
  podtynkowe prysznicowe") — to NAJWIĘKSZY łup. Wysoki relevance, P1. Odtwarzasz strukturę
  kategorii konkurenta, nie pojedyncze produkty.
- **`product` — rozróżnij dwie sytuacje:**
  - karta rankuje na **frazę generyczną/kategorialną** („bateria umywalkowa wodospad",
    „dozownik czarny") → to OKAZJA: u siebie pokryjesz to kategorią lub produktem. relevance wysoki.
  - karta rankuje na **własny model/SKU konkurenta** („br hugo hug buw050c", „ontario satyna",
    „apala mosiądz antyczny") → to de facto **brand produktowy konkurenta**, nieodtwarzalny
    (nie sprzedajesz jego SKU). Traktuj jak brand → relevance 0.1–0.3, junk_keywords z kodami modeli.
- **`region` / `system`** → SKIP (lokalizator dystrybutorów, URL parametryczny — nie są treścią
  do odtworzenia; `system` jest auto-SKIP-owany).
- **Uwaga na ambiwalencję kategorii:** w niektórych niszach head term sklepu jest dwuznaczny
  (np. „baterie" = baterie AA vs baterie łazienkowe; „zamki" = zamki drzwiowe vs budowle).
  Jeśli konkurent rankuje na taką gołą frazę, oceń intencję — często to przypadkowy ruch, nie cel.

## Wejście — obsługiwane formaty
Eksport Senuto (PL), Ahrefs/SEMrush (EN). Skrypt auto-mapuje nagłówki kolumn (keyword, volume,
position, traffic, cpc, kd, url) i auto-wykrywa separator CSV (`,`/`;`). Wymagane minimum:
keyword, volume, position, url. Brak `traffic` → ruch liczony jest z dostępnych kolumn (0 gdy brak).
