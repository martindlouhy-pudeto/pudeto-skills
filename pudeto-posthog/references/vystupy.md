# Výstupy pro klienta

## Jazyk

Klient není analytik. Místo technických pojmů:

| Ne | Ano |
|---|---|
| INP p75 | odezva na ťuknutí (u 3 ze 4 návštěv kratší než …) |
| rage click | opakované, naštvané klikání |
| in-app webview | prohlížeč uvnitř aplikace Facebook / Instagram |
| funnel, drop-off | kroky nákupu, kolik lidí v kroku odpadne |
| conversion rate | podíl lidí, kteří nakoupili |
| event chybí v dataLayer | obchod tuto událost neposílá |

Procenta s mezerou (`41 %`), čísla s mezerou po tisících (`3 185`), data `27. 10. – 8. 11. 2026`. Každá tabulka a graf má zdroj, období a omezení (souhlas).

## Claude dokument

Nadpis = otázka klienta („Proč lidé nedokončí nákup – <obchod>“). Oddíly:

1. **Shrnutí** – hlavní závěr ve dvou větách s čísly, pak číslovaný seznam překážek podle dopadu (tučný název + jedna věta s číslem), kdo opravuje. Tohle je to, co klient přečte.
2. **Data a období** – tabulka zdrojů (co měří, proč se čísla liší), filtry, okno funnelu, datum stažení.
3. **Kde lidé odpadají** – graf kroků na stránce objednávky (sloupce úměrné počtu, zvýrazněný největší úbytek) + věta, kolik z košíku dojde na objednávku a kolik z těch, kdo začnou vyplňovat, nakoupí.
4. **Zařízení a operační systémy** – vodorovné sloupce konverze podle OS × prohlížeč × aplikace (aplikace zvýrazněné, průměr jako značka), pod tím tabulka GA4 po OS s tržbami; věta o tabletech.
5. **Seznam problémů** – u každého: co, čísla (tabulka, když jde o srovnání segmentů), důkaz ze záznamů, **Doporučení (kdo)**.
6. **Produkty / parky a zdroje** – tabulka s podílem in-app a konverzí v prohlížeči a v aplikaci; zdroje z GA4 orientačně.
7. **Co jsme viděli v záznamech** – tabulka kategorií, poznámka o vzorku, odkazy pro tým.
8. **Měření** – díry v datech a řešení.
9. **Doporučený postup** – tabulka # | co | kdo | priorita | problém.

Graf vždy z řádků dat (ne opsaná čísla), po vykreslení se na něj podívej (screenshot) a oprav překryvy popisků.

## Slidy (Claude Slides)

Design systém klienta nebo agentury podle nastavení; jinak barvy agentury z `config.md`. Kostra (11 slidů):

1. Titul – otázka, obchod, období, „data PostHog a GA4“.
2. Hlavní zjištění – věta + tři velká čísla (konverze košíku, největší úbytek, platba).
3. Kroky na objednávce – sloupce, zvýrazněný úbytek.
4. Zařízení – sloupce podle OS × prohlížeč, aplikace zvýrazněné.
5. Největší problém do hloubky (např. rychlost na Androidu) – velké číslo + tabulka + potenciál jako horní odhad.
6. Seznam překážek – tabulka 8 řádků, top 4 zvýrazněné.
7.–9. Detail problémů (karty: číslo, co se děje, návrh).
10. Měření – tmavý slide, dvě karty.
11. Co udělat a kdo – tabulka s prioritou.

Zdroj na každém slidu v patičce, podrobnosti do poznámek řečníka. Osobní údaje ani snímky záznamů na slidy nedávej.

## Shrnutí do projektu

`claude/vyhodnoceni-<tema>-<RRRR-MM-DD>.md`: odkazy na dokument a slidy, zdroje a filtry, hlavní čísla (funnel, segmenty, GA4 po OS, rychlost), seznam překážek, stav měření, `scan_id` Replay Vision. Další analýza z něj vychází a porovnává.
