# Changelog

## 1.1.0 – 2026-11-10

Cíl: dostat z free tarifu PostHogu maximum hned od nasazení.

- Limity free tarifu jako rozpočet (§6): tabulka limitů, skutečná spotřeba z `billing-usage-get` s náhradním odhadem z dat, denní strop a tempo záznamů, datum vyčerpání, konkrétní hodnota vzorkování nebo URL triggeru, rozpočet událostí.
- Povinná kontrola 24 hodin po publikaci nasazení (§9.1): objem, úplnost e-commerce událostí, nákup podle stránky, rozsah, záznamy, rozpočet, verdikt a zápis. Vznikla z praxe: událost nákupu chodila s textem „false“ místo čísla objednávky a bez hodnoty.
- Cena rozboru záznamů předem přes `vision-scanners-estimate` a skutečná spotřeba kreditů do klientského souboru (§6.4).
- Důkazy ze záznamů zapisovat slovy hned – záznamy ve free tarifu po 30 dnech mizí (§5).
- Připravenost dat před analýzou: aspoň 7 celých dní a 30 osob ve skupině, jinak nabídnout počkat nebo sloučit skupiny (§7).
- Čtení spotřeby v billingu povoleno, změny v billingu dál zakázané (§2, §4).
- Šablona klienta: období kvóty, denní strop, tempo, vzorkování, kontrola po nasazení, kredity na relaci.

## 1.0.0 – 2026-11-09

- První verze: pravidla práce v PostHogu přes MCP (brána, karta relace, úrovně nástrojů, absolutní zákazy, osobní údaje, kvóty, výklad čísel, záznam o relaci).
- Zápis povolen jen pro insighty, dashboardy, notebooky a anotace po rekapitulaci a ano; nastavení projektu, replay, mazání, sdílení a automatizace zakázané.
- Recept na vyhodnocení košíku (PostHog + GA4) ze dvou analýz obchodu se vstupenkami na podzim 2026, 19 HogQL dotazů, postup Replay Vision, kostra dokumentu a slidů.
- Poučení z nasazení přes GTM (oprava šablony, consent, maskování, kvóta záznamů).
- Rozhodnutí správce: automatické načítání při práci v PostHogu, limit Replay Vision 30 relací na běh, kohorty a akce zakázané.
