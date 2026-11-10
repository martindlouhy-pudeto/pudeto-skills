# Replay Vision – rozbor záznamů

Replay Vision nechá model projít záznamy a odpovědět na otázku. Spotřebovává kredity (měsíční limit organizace). Používej ho na vzorek relací, které čísla ukázala jako problémové – ne na prohlížení „co se děje“.

## Postup

1. **Kvóta a cena.** `vision-quota-get` → limit, spotřeba, konec období. `vision-scanners-estimate` → `credits_per_observation` pro zvolený model; počet relací × cena proti zbývajícím kreditům (SKILL §6.4). Do karty relace.
2. **Záznamy existují?** Denní počet záznamů (SKILL §6.1). Vzorek ber jen z dnů, kdy se nahrávalo.
3. **Vzorek.** `sql-recepty.md` Q19: relace, které došly na stránku objednávky a nezaplatily, se záznamem delším než 20 s. Rozvrstvi 25–30 relací:
   - Android s rozpracovaným formulářem 8, Android bez formuláře 8,
   - iPhone s formulářem 4–6,
   - Windows s formulářem 4,
   - in-app Facebook/Instagram 6.
   Poměr uprav podle toho, kde jsou v datech největší rozdíly.
4. **Oznámení / schválení.** Počet relací a účel; nad limit z `config.md` rekapitulace a ano (SKILL §6.2).
5. **Sken.** `vision-scanners-inline-scan` se `session_ids` a promptem níže, `scanner_type` monitor. Nikdy `vision-scanners-create`.
6. **Výsledky.** `vision-scanners-observations-list` se `scanner_id` = `scan_id`. Čekej, dokud stavy nejsou `succeeded` / `failed` / `ineligible`; pár běžících nech být a napiš, že chybí.
7. **Třídění.** Každou relaci zařaď do jedné kategorie (níže). Relace, které skončily objednávkou jinou platbou (převod, poukaz), vyřaď – nejsou to odchody, ale díra v měření.
8. **Ověření.** U technických nálezů zkontroluj eventy té relace (klik na platbu bez pageview brány, remove_from_cart po zobrazení ceny). Model se plete hlavně v tom, zda tlačítko „nereagovalo“ – bez eventu to označ jako „podle záznamu“.
9. **Hodnocení pozorování** (palce) jen s verdiktem člověka, nikdy podle sebe.

## Prompt (šablona)

Uprav popis obchodu a názvy kroků; otázky a kategorie drž stejné, aby šly analýzy porovnat v čase.

```
This is a ticket shop (<platforma>, Czech). Flow: ticket selection (<cesty výběru>) -> order page (<cesta objednávky>, contact form with first name, surname, e-mail, phone, consent checkboxes, payment method, '<text tlačítka platby>' button) -> payment gateway. In this session the visitor reached the order page but did NOT reach payment. Watch only what is visible on screen and answer: (1) What did the visitor do on the order page (scroll, fill fields, tap checkboxes, change quantities, remove tickets, go back)? (2) Did any control respond slowly or not at all to taps (e.g. checkbox needing several taps, delayed input, frozen screen)? Quote what you saw. (3) Was there any error or validation message visible? Quote its text. (4) Did the visitor see a total price, a fee, or a price change right before leaving? (5) Most likely reason the visitor did not proceed to payment, picked from: price/fee surprise, comparing options or browsing, technical problem/slow page, form or validation friction, consent checkbox friction, cart emptied/changed mind, left the app/in-app browser issue, inconclusive. Give on-screen evidence for the reason; answer 'inconclusive' if the recording does not show it. Do not read out or repeat any personal data typed into the form. Answer briefly in Czech.
```

## Kategorie do výstupu

| Kategorie | Patří sem |
|---|---|
| Prohlížení a porovnávání | po zobrazení celkové ceny návrat do výběru, odebrání položek, přechod na jiný produkt, menu |
| Technická překážka | validační hláška, která zastavila nákup; klik na platbu bez přesměrování; vypršení rezervace; zamrzlá stránka |
| Nejasné | formulář rozpracovaný nebo hotový, pak nečinnost a konec |
| Mimo odchody | objednávka dokončena jinou platbou → do „Měření“ |

Do dokumentu: tabulka kategorie × počet × anonymní příklad („vstupenka 330 Kč + poplatek 20 Kč, hned ji odebral“) a 3–5 odkazů na záznamy pro tým. Vždy věta, že vzorek ukazuje typy problémů, ne jejich podíl.

## Co si zapsat

`scan_id`, počet relací, období vzorku, rozdělení po segmentech, spotřebu kreditů (rozdíl `credits_used` před a po, přepočtený na relaci) – do shrnutí v projektu a do klientského souboru (historie analýz). Ke každému nálezu ze záznamu slovní popis toho, co bylo vidět: záznamy ve free tarifu po 30 dnech mizí. Stejný prompt na stejné relace se příště nezpoplatní.
