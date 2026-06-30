# Changelog — competitor-recon

## 2026-06-29 — v1 (start)
Skill typujący strony konkurenta do odtworzenia z eksportu widoczności (Senuto/Ahrefs/SEMrush).

- `recon_tools.py ingest` — auto-mapowanie kolumn, derywacja brandu z domeny, regułowa
  klasyfikacja fraz (brand/foreign/generic_suspect/commercial/local/informational/neutral),
  agregacja keyword→URL, kandydaci main_kw (po wolumenie / po ruchu), zgadywanie typu z URL,
  generuje `worksheet.md` do osądu Claude'a.
- Krok osądu (Claude → `judgments.json`): finalny typ, main_keyword, relevance do DD,
  junk_keywords (potwierdzenie ambiwalentnych jak „landing"/„biogram"), content_type, note.
- `recon_tools.py score` — composite Opportunity Score
  `relevance × (0.6·sqrt(ruch) + 0.2·szerokość + 0.2·CPC) × (1−0.3·KD)`, tiery P1/P2/P3/SKIP,
  `report.md` + `priorities.csv`.

### Kalibracja (z boju na kamanmarketing.pl, 2978 fraz / 246 stron)
- Ruch normalizowany przez **sqrt** — jeden off-topowy outlier o dużym ruchu (blog IG 1302)
  nie spłaszcza money pages. Progi tierów P1=45 / P2=25 dobrane do realnego sufitu skali.
- Walidacja: money pages (agencja warszawa/katowice, ile kosztuje pozycjonowanie) → P1;
  blog IG (duży ruch, relevance 0.5) → P2; montaż filmów (327 ruchu, relevance 0.1) → SKIP.
  Reguła poprawnie odsiała brand (kaman*), cyrylicę i 1-słowowe generyki (dropship/landing/biogram).

## 2026-06-29 — v1.1 (drugi boj: projektefektywny.pl, nisza eventowa)
Stress-test na konkurencie z INNEJ branży (team building/eventy) ujawnił poprawki:

- **Typowanie stron przepisane.** Polskie długie slugi usług (`organizacja-imprez-integracyjnych-dla-firm`)
  myliły regułę „≥5 słów = blog". Teraz: tokeny ofertowe/usługowe w slugu (oferta/cennik/organizacj/
  dla-firm/obsluga…) → service; jawne `/blog/` → blog; `unknown` dociągany sygnałem komercyjnym
  (CPC≥5 + udział fraz commercial/local). Usunięto flip blog→service (mylił poradniki) i wąsko
  przycięto token `realizacj` (łapał blogowe „omówienie realizacji").
- **Generalizacja poza DD.** `--our-domain` = domena, którą rozwijamy (DD lub klient) + nowy
  `--our-topics "opis niszy"`. Relevance liczone względem właściwej niszy, nie zawsze DD.
- **Soczewka business-core vs traffic-bait** dopisana do osądu: nawet on-niche, generyczne
  wysokowolumenowe przynęty (events: „starodawne gry" 4400, „atrakcji" 450000; DD: „montaż filmów")
  → relevance 0.2–0.4 → SKIP. Walidacja: 3 strony ofertowe → P1, „starodawne gry" → SKIP.
- Brand: wariant spacjowany łapany przez `kw.replace(' ','')` (np. „projekt efektywny" = brand).

## 2026-06-29 — v1.2 (trzeci boj: bluewater.pl, ecommerce)
Stress-test na sklepie (baterie/armatura, flat-URL ze sufiksami Shoper/IdoSell) — nowe typy stron:

- **Ecommerce page types.** Detekcja kodów-sufiksów: `-cNNN`=category, `-pNNN`=product,
  `-rNNN`=region, oraz ścieżek (`/kategoria/`, `/produkt/`, `/c/`, `/p/`…). Walidacja na
  bluewater: 48 category, 62 product, 28 service_local (landingi miastowe), 25 region, 32 blog.
- **URL systemowy/parametryczny** (`files.php?…`, `index.php?producent=…`, `?sort=`) →
  typ `system`, **auto-SKIP w score** (force tier SKIP, niezależnie od osądu).
- **Wytyczne osądu ecom w SKILL.md:** category = money page #1 (head transakcyjny → P1);
  product rozdzielony na generyk (okazja) vs własne SKU/model konkurenta („br hugo buw050c" →
  brand-like, SKIP); region/system → SKIP; uwaga na ambiwalentne head („baterie" = AA vs łazienkowe).
- Walidacja: category „baterie podtynkowe prysznicowe" (772 ruchu) → P1; homepage/region/files.php → SKIP.

### Uniwersalność (stan)
Skill przetestowany na 3 modelach: agencja (kaman), usługi/eventy (projektefektywny), ecommerce
(bluewater). Wejście = eksport Senuto + `--our-domain`/`--our-topics`. Następny krok (do zrobienia):
elastyczne wejście „podaj domenę / branżę / eksport" — na razie wymagany eksport widoczności.

## 2026-06-29 — v2 (przelot przez skill-creator: 6 evali with/baseline na 3 niszach)
Eval-loop ujawnił wady, których 3 rundy walidacji nie złapały. Naprawione:

- **TAIL — koniec zalewania P3.** Nieosądzone strony dostawały default relevance 0.6 (≥ próg SKIP)
  i przeciekały do P3 jako rekomendacje z surowym main_kw — w eval-0 raport polecał `kaman beauty`,
  `co to znaczy kaman`, `dropship`, `ugc`. Teraz nieosądzone → tier **TAIL** (poza P1/P2/P3), osobna
  sekcja „długi ogon" w report. Przeciek brand/generyk w rekomendacjach: 5 → **0**.
- **Tiery WZGLĘDNE** (percentyl wśród osądzonych: P1=top20%, P2=do55%) zamiast progów absolutnych.
  Naprawia kompresję małego konkurenta: bluewater `baterie podtynkowe prysznicowe` (max 773 ruchu)
  z P2 → **P1**; reguła „category=P1" działa niezależnie od wielkości domeny.
- **Twardy floor relevance** (<0.5 → max P3, <0.3 → SKIP), udokumentowany. Traffic-bait
  „kiedy publikować na IG" (rel 0.45, ruch 1302) już nie wchodzi do P2 — ląduje w P3.
- **Worksheet = 3 pule** (📊ruch + 💰cpc + 🎯open-goal) zamiast samego top-wg-ruchu. Łapie
  wysokoCPC-owe money pages z małym OBECNYM ruchem, które stary worksheet gubił (eventy:
  `/organizacja-piknikow/` CPC 123, `/eventy-firmowe/`). `latent_potential` = wolumen fraz, gdzie
  konkurent słabo rankuje (poz≥8, vol≥50); `open_goal` gdy latent≥1500. Sekcja „🎯 Open goals" w report.
- **Winsoryzacja CPC** (cap 60 zł) — outliery Senuto (995/202/128) nie zawyżają komponentu komercyjnego;
  flaga ⚠cpc w worksheet. **Rozkład typów** w nagłówku report (widać luki, np. brak bloga).
- Drobne: wzór w doc poprawiony na `√ruch`; worksheet kompaktowy (9 fraz, limit Read); pole
  „qualified_traffic"→„ruch_now"; +30 miast (Koło, regiony). Benchmark v1: with-skill 90.5% vs baseline 84.9%.

## 2026-06-29 — v2.1 (iteracja 2: świeże agenty na przepisanym SKILL.md)
3 świeże agenty (po jednym na niszę) osądziły 55–57 stron każdy. **Jednogłośny** sygnał + bugi:

- **Latent wpięty do score** (`opportunity = ruch + 0.12·latent`). Wszystkie 3 agenty zgłosiły, że
  `√ruch` chowa open-goale z zerowym OBECNYM ruchem, ale dużym popytem. Po zmianie: ecommerce
  `baterie z wyciąganą wylewką` (ruch 18, latent 11560) → **P1 #1**; events `gry integracyjne dla firm`
  (ruch 34) → P1. Crown jewel z dużym ruchem dalej P1 — ruch i latent zbalansowane.
- **Latent liczony PO usunięciu junk** (w score, nie ingest) — fix buga, gdzie generyk „atrakcji"
  (vol 450k, oznaczony junk) zawyżał open-goals do 450120.
- **Open-goals = tylko P1/P2** (nie P3) — open-goal z niskim relevance to szum, nie „szybki łup".
- **Pule worksheet hojniejsze** (open=max(18,top/2), cpc=max(12,top/3)) — mniej ręcznego ratowania z TAIL.
- **Nagłówek raportu** foregrounduje `--our-topics` (nie myli „double-digital.pl" przy analizie klienta).

Benchmark iter2 vs iter1 (with-skill, ten sam grader): **19/20 (95%) vs 18/20 (90%)** — wzrost z
naprawy przecieku brandu (TAIL). Nauka: grader liczył TAIL jako rekomendacje → poprawiony
(rekomendacje = tylko P1/P2/P3). Pozostały 1 fail = brzegowy „ugc" P3 z osądu agenta (kalibracja).

## 2026-06-29 — v2.2 (domenowo-agnostyczny domyślnie)
Skill nie zakłada już żadnej „naszej" strony (wcześniej `--our-domain` domyślnie `double-digital.pl`
→ filtrował relevance pod DD nawet bez kontekstu).

- `--our-domain` i `--our-topics` są teraz **opcjonalne** (default None). Bez nich skill rankuje
  strony **wg wartości do odtworzenia na gruncie samego konkurenta** (business-core vs brand/
  generyk/traffic-bait) — wypluwa listę stron + najważniejsze frazy każdej, bez założeń o branży.
- `--our-topics "nisza"` włącza OPCJONALNY filtr dopasowania pod własną niszę.
- Nagłówki report/worksheet: „Tryb: ranking wg wartości strony" (bez niszy) lub „Filtr niszy: …".
- SKILL.md przeramowane: relevance = „wartość strony do odtworzenia", nie „dopasowanie do DD";
  usunięte twarde odniesienia do double-digital.pl z zasad osądu i przykładów.

## 2026-06-30 — v2.3 (report.md czytelny dla nie-eksperta)
Raport był gęsty i żargonowy (score 72.4, latent, KD, CPC, wzór ze √) + setki rozwlekłych kart
P3/SKIP. Przepisany `report.md` (twarde liczby zostają w `priorities.csv` — dla profesjonalisty):

- **Plain-language** wstęp + „Jak to czytać" (🔴 zrób najpierw / 🟠 potem / 🟡 później / 🎯 łatwy łup
  / ⏭️ odpuść) + sekcja **„⭐ W skrócie"** (ile stron, dominujący typ, największa okazja, luki).
- **Żargon przetłumaczony:** `latent` → „popyt, którego konkurent nie łapie"; `KD` → trudność
  słowna (bardzo niska…bardzo wysoka); typy stron po polsku („kategoria sklepu", „wpis blogowy");
  score ukryty (numeracja w sekcji); CPC schowane do CSV.
- **Karty P1/P2** = plain zdanie „co zrobić + dlaczego" + 1 linia skali + frazy + wzór URL + opcjonalny
  💡 komentarz eksperta. **P3/SKIP/TAIL zwinięte** do jednolinijkowców (z setek linii → ~300).
- Sekcja **„🎯 Najszybsze łupy"** zbiorczo. Footer: „pełne liczby w priorities.csv".
