# Vyhodnocení košíku – recept

Cíl: seznam překážek, které brání nákupu, seřazený podle dopadu, s daty, důkazem a vlastníkem opravy. Zdroje: PostHog (chování, kliky, rychlost, záznamy) + GA4 (návštěvnost bez souhlasu, tržby, zdroje). Výstup: dokument a slidy pro klienta (`vystupy.md`) + shrnutí do projektu.

Celý běh je čtení. Kredity čerpá jen krok 8 (Replay Vision). Typický rozsah: 25–40 dotazů do PostHogu, 3–4 do GA4, 1 sken 25–30 relací.

Dotazy jsou v `sql-recepty.md` pod čísly (Q1 …). Parametry v `{{…}}` ber z klientského souboru.

## 0. Před startem

- Brána a karta relace (SKILL §1).
- Diagnostika pro uživatele: kolik dotazů, jestli a kolik kreditů Replay Vision, co bude rozhodovat (forma výstupu, publikum). Anketou: jedno schválení pro celý běh, nebo krok po kroku.
- Forma výstupu a publikum anketou, pokud nejsou jasné (klient / interně / dodavatel obchodu).
- Přečti předchozí analýzu klienta (odkaz v klientském souboru nebo v projektu) – navazuj, neopakuj, a porovnej čísla.

## 1. Inventura

1. `project-get` → ID, název, zóna, maskování, nastavení replay.
2. `metric-list` (pravidla serveru) – schválená metrika má přednost.
3. `read-data-schema` events → které e-commerce eventy existují (add_to_cart, view_cart, begin_checkout, add_payment_info, purchase, remove_from_cart). Chybějící zapiš do „Měření“.
4. Q1 – objem: události, osoby, relace, první a poslední event za období.
5. Q2 – rozpad e-commerce eventů podle produktu/parku a hostu → potvrď rozsah a co vyřadit (produkt bez stránky košíku, cizí hosty).
6. Q3 – mapa cest stránek (normalizované pathname) → najdi cestu stránky objednávky, platební brány a stavové stránky. Doplň do klientského souboru, pokud chybí.
7. SKILL §6.1 – denní počet záznamů. Stojí-li nahrávání, plánuj krok 8 na období, kdy běželo.

## 2. Hlavní funnel po osobách

Q4: add_to_cart → stránka objednávky (view_cart nebo pageview objednávky) → purchase, okno 1 den, po osobách, rozpad podle `$device_type`. Zapiš: % z košíku na objednávku, % z objednávky na nákup, % z košíku na nákup.

## 3. Segmenty

1. Q5 – stejný funnel podle `$os` × `$browser` × `$webview_app`, segmenty s 30+ osobami.
2. Q6 – prostředí (in-app Facebook/Instagram / jiná aplikace / běžný prohlížeč) × OS × týden. Ukáže, jestli se problém mění v čase.
3. Podíl in-app na košících spočítej zvlášť – bývá to největší segmentový rozdíl.

Typický obraz z praxe: iPhone v prohlížeči nejlepší, Android Chrome o 8–10 bodů níž, in-app Facebook/Instagram na polovině nebo třetině. Typ zařízení (mobil vs počítač) sám rozhoduje málo.

## 4. Kroky uvnitř stránky objednávky

1. Q7 – inventura interakcí na stránce objednávky (autocapture: id, name, text prvku, typ click/change). Najdi: pole formuláře, souhlasy, tlačítko platby, volbu platby, odkazy ven. Doplň ID do klientského souboru.
2. Q8 – mikrofunnel: na objednávce → začal vyplňovat → souhlasy → klik na platbu → platební brána → nákup, striktně v pořadí, po segmentech.
3. Q9 – objednávka → brána → nákup (3 kroky) po segmentech: odliší ztrátu na stránce od ztráty na platbě.

Kde je největší skok, tam je hlavní problém. V praxi: 40 % lidí na objednávce nezačne vyplňovat, kdo začne, nakoupí ze 3/4, platební brána propustí přes 95 %.

## 5. Kdo nezačne vyplňovat

Q10 – osoby na objednávce bez změny ve formuláři a bez nákupu: podíl s návratem do výběru PO otevření objednávky, s odebráním položky, bez jediného kliknutí, medián času na stránce, podíl pod 10 s. Pozor: návrat do výběru počítej jen po čase otevření objednávky, jinak je to 100 %.

Vysoký podíl rychlých návratů a odebírání = lidé jdou do objednávky zjistit celkovou cenu (poplatky, součet). Ověř v záznamech (krok 8) a ve výběru vstupenek, jestli se celková cena ukazuje dřív.

## 6. Tření

1. Q11 – rage clicky podle prvku a stránky. Prvek z `elements_chain` přes `extract()` (ne `elements_chain_texts[1]` – padá).
2. Q12 – klepnutí na souhlasy na osobu, podíl s víc klepnutími, než je počet políček × 2 (klik na label i input), rage na souhlasech.
3. Q13 – tlačítko platby: podíl relací s opakovaným klikem, s bránou až po 20 s, bez brány, s jinou platbou (převod, poukaz).
4. Q14 – opakované úpravy polí (e-mail, telefon, jméno) po segmentech. Výrazně vyšší u jednoho OS = validace nebo klávesnice.
5. Q15 – kliky na tlačítka dialogů (vypršení rezervace, smazání košíku, chyba) přes `elements_chain ILIKE` s úzkým časovým oknem. Text dialogu ověř v záznamu.

## 7. Rychlost stránek

1. Q16 – INP p75/p90, LCP p75, CLS p75 po typu stránky × segment.
2. Q17 – stejné před a po změně nahrávání záznamů (pokud se v období měnilo) – vyloučí vliv nahrávacího skriptu.

Pomalá jen jedna stránka (objednávka) a jen na jednom OS = konkrétní úkol pro dodavatele obchodu: profilovat stránku na běžném zařízení se zpomaleným procesorem. Kandidáti: pole telefonu s výběrem země, překreslování celého formuláře při psaní, vlastní zaškrtávátka.

## 8. Záznamy (Replay Vision)

`replay-vision.md`. Vzorek 25–30 relací z období s nahráváním: osoby na objednávce bez platby, rozvrstvené podle segmentu (Android s formulářem / bez, iPhone, Windows, in-app). Výsledky roztřiď do kategorií, ověř proti eventům, relace dokončené jinou platbou vyřaď.

## 9. Produkty / parky / zdroje

1. Q18 – funnel podle produktu nebo parku s podílem in-app a konverzí zvlášť v prohlížeči a v aplikaci. Rozliší „park je slabý“ od „park má víc lidí z Facebooku“.
2. GA4 kanály (krok 10) – košík → nákup podle zdroje relace, orientačně.

## 10. GA4 pro srovnání

`run_report` na property z klientského souboru, filtr `hostName` na hosty obchodu a `notExpression` na cesty vyřazených produktů:

- dimenze `deviceCategory`, `operatingSystem`, `eventName` (page_view, add_to_cart, begin_checkout, purchase), metriky `totalUsers`, `eventCount`, `purchaseRevenue` → tabulka po OS s tržbami a průměrnou objednávkou;
- `operatingSystem`, `browser`, `eventName` → prohlížeče (pozor: in-app Facebook na Androidu = „Chrome“, jiné aplikace = „Android Webview“);
- `sessionDefaultChannelGroup`, `sessionSource`, `eventName` → zdroje.

Poměry z GA4 jsou počty uživatelů u jednotlivých eventů, ne striktní funnel – označ „orientačně“. Tablety bývají pod 1 % – stačí věta.

## 11. Seznam problémů

Pro každý nález: co, kolik lidí se týká (objem × rozdíl), čím je doložený (data + záznam), doporučení, kdo opraví (dodavatel obchodu / marketing / klient / agentura). Seřaď podle dopadu. Typické položky:

1. pomalá stránka objednávky na jednom OS,
2. celková cena / poplatek až v objednávce,
3. validace formuláře (telefon, e-mail, obecná hláška bez pole),
4. prohlížeč v aplikaci Facebooku a Instagramu,
5. vlastní zaškrtávátka souhlasu,
6. omezení košíku (nelze kombinovat produkty → „smazat košík“),
7. vypršení rezervace,
8. klik na platbu bez přesměrování.

Odhad potenciálu jen u největšího bodu a jako horní odhad (SKILL §7).

## 12. Měření

Vždy samostatný oddíl: stav nahrávání záznamů, platby bez `purchase`, chybějící eventy, co GA4 nevidí, souhlas.

## 13. Kontrola a výstupy

- Kontrolní součty (SKILL §7) a procenta z uvedených čísel.
- Dokument a slidy podle `vystupy.md`; grafy z dat, ne opsané.
- Shrnutí do projektu: hlavní čísla, seznam problémů, stav měření, ID scanu Replay Vision, odkazy na dokument a slidy.
- Do klientského souboru: nová ID prvků, cesty, odkaz na analýzu, stav kvóty.
- Záznam o relaci (SKILL §10).
