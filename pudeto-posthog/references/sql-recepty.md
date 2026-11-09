# HogQL recepty

Dotazy pro `execute-sql`. Do `context` vždy napiš účel a výsledek kontroly katalogu („governed catalog consulted: no match“), pokud žádná schválená metrika nesedí. Před prvním dotazem ověř eventy a properties přes `read-data-schema` (pravidla serveru).

## Parametry (z klientského souboru)

| Parametr | Příklad | Význam |
|---|---|---|
| `{{OD}}`, `{{DO}}` | `'2026-10-27 00:00:00'` | období, `DO` = začátek dne po posledním celém dni |
| `{{TZ}}` | `'Europe/Prague'` | zóna klienta pro denní řezy |
| `{{FILTR_PRODUKT}}` | `properties.park NOT IN ('poukazy')` | vyřazení produktů s jiným postupem; bez vyřazení `1` |
| `{{PRODUKT}}` | `properties.park` | property produktu / parku / kategorie |
| `{{CESTA_OBCHOD}}` | `'^/app/'` | regex cest obchodu |
| `{{CESTA_OBJEDNAVKA}}` | `'^/app/[a-z]+/order$'` | regex stránky objednávky (formulář) |
| `{{CESTA_BRANA}}` | `'/payment-gateway$'` | regex stránky platební brány |
| `{{CESTA_VYBER}}` | `'/(products\|events\|packages)'` | regex stránek výběru |
| `{{POLE}}` | `'firstname\|surname\|email\|telephone'` | `name` polí formuláře |
| `{{SOUHLASY}}` | `'(checkboxLabel)?terms(_[0-9])?'` | `id` zaškrtávátek souhlasu |
| `{{PLATBA}}` | `'order-button'` | `id` tlačítka platby |
| `{{JINA_PLATBA}}` | `'payType-label-(invoice\|voucher)'` | `id` volby platby bez brány |

V tabulce jsou `|` uvnitř regexů escapované kvůli markdownu – do dotazu je piš bez zpětného lomítka.

## Úskalí, na která jsme narazili

- `elements_chain_texts[1]` a podobné indexy padají na interní chybě → prvek vytahuj z `elements_chain` přes `extract(elements_chain, 'attr__id="([^"]+)"')`, `'text="([^"]{1,60})"'`, `'attr__name="([^"]+)"'`.
- `WITH TOTALS` nevrátí řádek součtu → součet spočítej zvlášť nebo sečti ručně a napiš to.
- `toDate()` bere jeden argument: `toDate(toTimeZone(timestamp, {{TZ}}))`.
- Časový filtr vždy ve `WHERE` na `timestamp`, i u bodového dotazu.
- `windowFunnel` pracuje v celých sekundách: `toDateTime(timestamp)`.
- Záznamy: tabulka `raw_session_replay_events`, filtr na `min_first_timestamp`.
- `ILIKE '%…%'` na `elements_chain` je plný sken – jen s úzkým oknem a filtrem na event.
- Celé `properties` nikdy nevybírej.

## Q1 – objem za období

```sql
SELECT count() AS events_total, uniq(person_id) AS persons,
       uniq(properties.$session_id) AS sessions,
       min(timestamp) AS first_event, max(timestamp) AS last_event
FROM events
WHERE timestamp >= toDateTime({{OD}}) AND timestamp < toDateTime({{DO}})
```

Rozpad podle eventu: `SELECT event, count(), uniq(person_id) … GROUP BY event ORDER BY 2 DESC`. Po dnech: `GROUP BY toDate(toTimeZone(timestamp, {{TZ}}))` s `countIf(event = 'purchase')` a `round(sumIf(toFloat(properties.value), event = 'purchase'))`.

## Q2 – e-commerce podle produktu a hostu

```sql
SELECT {{PRODUKT}} AS produkt, properties.$host AS host,
       countIf(event = 'add_to_cart') AS atc, countIf(event = 'view_cart') AS vc,
       countIf(event = 'begin_checkout') AS bc, countIf(event = 'purchase') AS purchases,
       uniqIf(person_id, event = 'add_to_cart') AS atc_persons
FROM events
WHERE timestamp >= toDateTime({{OD}}) AND timestamp < toDateTime({{DO}})
  AND event IN ('add_to_cart', 'view_cart', 'begin_checkout', 'purchase')
GROUP BY produkt, host ORDER BY atc DESC LIMIT 50
```

Produkt s `vc = 0` při nenulovém `atc` má jiný postup → do `{{FILTR_PRODUKT}}`.

## Q3 – mapa cest

```sql
SELECT replaceRegexpAll(properties.$pathname, '[0-9]+', 'N') AS path,
       count() AS views, uniq(person_id) AS persons
FROM events
WHERE timestamp >= toDateTime({{OD}}) AND timestamp < toDateTime({{DO}})
  AND event = '$pageview'
GROUP BY path ORDER BY persons DESC LIMIT 40
```

## Q4 – funnel po osobách podle typu zařízení

```sql
WITH f AS (
  SELECT person_id,
         windowFunnel(86400)(toDateTime(timestamp),
           event = 'add_to_cart', event = 'view_cart', event = 'purchase') AS steps,
         argMinIf(properties.$device_type, timestamp, event = 'add_to_cart') AS device
  FROM events
  WHERE timestamp >= toDateTime({{OD}}) AND timestamp < toDateTime({{DO}})
    AND event IN ('add_to_cart', 'view_cart', 'purchase') AND {{FILTR_PRODUKT}}
  GROUP BY person_id HAVING steps >= 1)
SELECT device, count() AS atc_persons, countIf(steps >= 2) AS reached_cart,
       countIf(steps = 3) AS purchased,
       round(100 * countIf(steps >= 2) / count(), 1) AS atc_to_cart_pct,
       round(100 * countIf(steps = 3) / countIf(steps >= 2), 1) AS cart_to_purchase_pct,
       round(100 * countIf(steps = 3) / count(), 1) AS atc_to_purchase_pct
FROM f GROUP BY device ORDER BY atc_persons DESC
```

Nemá-li obchod `view_cart`, nahraď druhý krok `event = '$pageview' AND match(properties.$pathname, {{CESTA_OBJEDNAVKA}})`.

## Q5 – podle OS, prohlížeče a aplikace

Stejné CTE jako Q4, navíc:

```sql
argMinIf(properties.$os, timestamp, event = 'add_to_cart') AS os,
argMinIf(properties.$browser, timestamp, event = 'add_to_cart') AS browser,
argMinIf(coalesce(properties.$webview_app, ''), timestamp, event = 'add_to_cart') AS webview
```

`GROUP BY os, device, webview, browser HAVING atc_persons >= 15` (do výstupu jen 30+).

## Q6 – prostředí × OS × týden

V CTE přidej `min(timestamp) AS first_ts`, pak:

```sql
SELECT multiIf(webview IN ('Facebook', 'Instagram'), 'in-app FB/IG',
               webview != '', 'in-app jiné', 'běžný prohlížeč') AS prostredi,
       os, toStartOfWeek(first_ts, 1) AS tyden, count() AS atc_persons,
       countIf(steps >= 2) AS reached_cart, countIf(steps = 3) AS purchased
FROM f WHERE os IN ('Android', 'iOS', 'Windows', 'Mac OS X')
GROUP BY prostredi, os, tyden ORDER BY prostredi, os, tyden
```

## Q7 – inventura interakcí na stránce objednávky

```sql
SELECT substring(extract(elements_chain, 'attr__id="([^"]+)"'), 1, 40) AS el_id,
       substring(extract(elements_chain, 'text="([^"]{1,60})"'), 1, 50) AS el_text,
       substring(extract(elements_chain, 'attr__name="([^"]+)"'), 1, 30) AS el_name,
       properties.$event_type AS etype, count() AS n, uniq(person_id) AS persons
FROM events
WHERE timestamp >= toDateTime({{OD}}) AND timestamp < toDateTime({{DO}})
  AND event = '$autocapture' AND match(properties.$pathname, {{CESTA_OBJEDNAVKA}})
GROUP BY el_id, el_text, el_name, etype ORDER BY persons DESC LIMIT 40
```

Jména polí a ID prvků vypiš jen jako názvy prvků – hodnoty polí autocapture neukládá a ty je nikdy nehledej.

## Q8 – mikrofunnel na stránce objednávky

```sql
WITH p AS (
  SELECT person_id,
    windowFunnel(86400)(toDateTime(timestamp),
      event = '$pageview' AND match(properties.$pathname, {{CESTA_OBJEDNAVKA}}),
      event = '$autocapture' AND properties.$event_type = 'change'
        AND match(elements_chain, concat('attr__name="(', {{POLE}}, ')"')),
      event = '$autocapture' AND match(elements_chain, concat('attr__id="', {{SOUHLASY}}, '"')),
      event = '$autocapture' AND match(elements_chain, concat('attr__id="', {{PLATBA}}, '"')),
      event = '$pageview' AND match(properties.$pathname, {{CESTA_BRANA}}),
      event = 'purchase') AS steps,
    argMin(properties.$os, timestamp) AS os,
    argMin(coalesce(properties.$webview_app, ''), timestamp) AS webview,
    argMin(properties.$device_type, timestamp) AS device
  FROM events
  WHERE timestamp >= toDateTime({{OD}}) AND timestamp < toDateTime({{DO}})
    AND ((event IN ('$pageview', '$autocapture')
          AND (match(properties.$pathname, {{CESTA_OBJEDNAVKA}}) OR match(properties.$pathname, {{CESTA_BRANA}})))
         OR (event = 'purchase' AND {{FILTR_PRODUKT}}))
  GROUP BY person_id HAVING steps >= 1)
SELECT multiIf(webview IN ('Facebook', 'Instagram'), 'in-app FB/IG', concat(device, ' ', os)) AS segment,
       count() AS s1_objednavka, countIf(steps >= 2) AS s2_formular, countIf(steps >= 3) AS s3_souhlasy,
       countIf(steps >= 4) AS s4_platba, countIf(steps >= 5) AS s5_brana, countIf(steps >= 6) AS s6_nakup
FROM p GROUP BY segment HAVING s1_objednavka >= 40 ORDER BY s1_objednavka DESC
```

Součet za všechny segmenty spusť zvlášť bez `GROUP BY`.

## Q9 – objednávka → brána → nákup

Jako Q8, jen tři kroky (pageview objednávky, pageview brány, purchase). Rozliší ztrátu na stránce od ztráty na platbě.

## Q10 – kdo nezačne vyplňovat

```sql
WITH per AS (
  SELECT person_id,
    minIf(timestamp, event = '$pageview' AND match(properties.$pathname, {{CESTA_OBJEDNAVKA}})) AS t_order,
    maxIf(timestamp, match(properties.$pathname, {{CESTA_OBJEDNAVKA}}) AND event != '$web_vitals') AS t_order_last,
    countIf(event = '$autocapture' AND properties.$event_type = 'change'
            AND match(elements_chain, concat('attr__name="(', {{POLE}}, ')"'))) AS form_changes,
    countIf(event = 'purchase') AS purchases,
    countIf(event = 'remove_from_cart') AS removes,
    groupArrayIf(timestamp, event = '$pageview' AND match(properties.$pathname, {{CESTA_VYBER}})) AS sel_ts,
    countIf(event = '$autocapture' AND match(properties.$pathname, {{CESTA_OBJEDNAVKA}})
            AND properties.$event_type = 'click') AS order_clicks,
    argMin(properties.$os, timestamp) AS os,
    argMin(coalesce(properties.$webview_app, ''), timestamp) AS webview
  FROM events
  WHERE timestamp >= toDateTime({{OD}}) AND timestamp < toDateTime({{DO}})
    AND match(coalesce(properties.$pathname, ''), {{CESTA_OBCHOD}})
  GROUP BY person_id HAVING t_order > toDateTime('2000-01-01')),
lost AS (
  SELECT *, arrayExists(x -> x > t_order, sel_ts) AS back_to_selection,
         dateDiff('second', t_order, t_order_last) AS secs_on_order
  FROM per WHERE form_changes = 0 AND purchases = 0)
SELECT multiIf(webview IN ('Facebook', 'Instagram'), 'in-app', os) AS segment, count() AS lost_no_form,
       round(100 * countIf(back_to_selection) / count(), 1) AS pct_back_to_selection,
       round(100 * countIf(removes > 0) / count(), 1) AS pct_removed_item,
       round(100 * countIf(order_clicks = 0) / count(), 1) AS pct_no_click,
       round(quantile(0.5)(secs_on_order)) AS median_secs,
       round(100 * countIf(secs_on_order < 10) / count(), 1) AS pct_under_10s
FROM lost GROUP BY segment HAVING lost_no_form >= 10 ORDER BY lost_no_form DESC
```

## Q11 – rage clicky podle prvku

```sql
SELECT replaceRegexpAll(properties.$pathname, '[0-9]+', 'N') AS path,
       substring(extract(elements_chain, 'attr__id="([^"]+)"'), 1, 40) AS el_id,
       substring(extract(elements_chain, 'text="([^"]{1,60})"'), 1, 50) AS el_text,
       substring(splitByChar(';', elements_chain)[1], 1, 80) AS el_first,
       count() AS rageclicks, uniq(person_id) AS persons
FROM events
WHERE timestamp >= toDateTime({{OD}}) AND timestamp < toDateTime({{DO}}) AND event = '$rageclick'
GROUP BY path, el_id, el_text, el_first ORDER BY persons DESC LIMIT 25
```

## Q12 – souhlasy

Po osobě: `countIf(event = '$autocapture' AND properties.$event_type = 'click' AND match(elements_chain, concat('attr__id="', {{SOUHLASY}}, '"')))` jako `terms_taps`, `countIf(event = '$rageclick' AND match(elements_chain, 'terms'))`, nákup; výstup medián klepnutí, podíl nad (počet políček × 2), podíl s rage, podíl s nákupem – po segmentech.

## Q13 – tlačítko platby

Po relaci (`properties.$session_id`) na stránkách objednávky a brány: čas prvního kliku na `{{PLATBA}}`, čas prvního pageview brány, počet kliků, změna `{{JINA_PLATBA}}`. Výstup: podíl s bránou, medián sekund klik → brána, podíl s 2+ kliky, podíl s bránou až po 20 s, podíl s jinou platbou.

## Q14 – opakované úpravy polí

Po relaci s alespoň jednou změnou jména: podíl relací, kde se e-mail / telefon / jméno měnily 2× a víc, a podíl bez brány – po segmentech.

## Q15 – dialogy a chybové hlášky

```sql
SELECT substring(extract(elements_chain, 'text="([^"]{1,80})"'), 1, 60) AS el_text,
       replaceRegexpAll(properties.$pathname, '[0-9]+', 'N') AS path,
       count() AS clicks, uniq(person_id) AS persons
FROM events
WHERE timestamp >= toDateTime({{OD}}) AND timestamp < toDateTime({{DO}})
  AND event = '$autocapture'
  AND (elements_chain ILIKE '%vypršel%' OR elements_chain ILIKE '%Pokračovat%'
       OR elements_chain ILIKE '%smazat%' OR elements_chain ILIKE '%chyba%')
GROUP BY el_text, path ORDER BY persons DESC LIMIT 15
```

Slova přizpůsob jazyku obchodu. Obecné „Pokračovat“ ověř v záznamu, než ho prohlásíš za konkrétní dialog – jinak „odhad“.

## Q16 – web vitals po typu stránky a segmentu

```sql
SELECT multiIf(match(properties.$pathname, {{CESTA_OBJEDNAVKA}}), 'objednávka',
               match(properties.$pathname, {{CESTA_VYBER}}), 'výběr', 'ostatní') AS page,
       multiIf(properties.$webview_app IN ('Facebook', 'Instagram'), 'in-app',
               concat(properties.$device_type, ' ', properties.$os)) AS segment,
       round(quantileIf(0.75)(toFloat(properties.$web_vitals_INP_value), properties.$web_vitals_INP_value IS NOT NULL)) AS inp_p75,
       round(quantileIf(0.90)(toFloat(properties.$web_vitals_INP_value), properties.$web_vitals_INP_value IS NOT NULL)) AS inp_p90,
       countIf(properties.$web_vitals_INP_value IS NOT NULL) AS inp_n,
       round(quantileIf(0.75)(toFloat(properties.$web_vitals_LCP_value), properties.$web_vitals_LCP_value IS NOT NULL)) AS lcp_p75,
       round(quantileIf(0.75)(toFloat(properties.$web_vitals_CLS_value), properties.$web_vitals_CLS_value IS NOT NULL), 3) AS cls_p75
FROM events
WHERE timestamp >= toDateTime({{OD}}) AND timestamp < toDateTime({{DO}})
  AND event = '$web_vitals' AND match(properties.$pathname, {{CESTA_OBCHOD}})
GROUP BY page, segment HAVING inp_n >= 50 ORDER BY page, inp_n DESC
```

## Q17 – rychlost před a po změně nahrávání

Q16 jen pro stránku objednávky, místo `page` sloupec `if(timestamp < toDateTime('<den změny>'), 'před', 'po')`.

## Q18 – podle produktu / parku

Q4 s `argMinIf({{PRODUKT}}, timestamp, event = 'add_to_cart') AS produkt` a výstupem: osoby s košíkem, podíl in-app, konverze celkem, v prohlížeči, v aplikaci.

## Q19 – vzorek relací se záznamem pro Replay Vision

```sql
WITH s AS (
  SELECT properties.$session_id AS sid, any(properties.$os) AS os,
         any(coalesce(properties.$webview_app, '')) AS webview,
         countIf(event = '$autocapture' AND properties.$event_type = 'change'
                 AND match(elements_chain, concat('attr__name="(', {{POLE}}, ')"'))) AS form_changes,
         countIf(event = 'purchase') AS purchases,
         countIf(event = '$pageview' AND match(properties.$pathname, {{CESTA_BRANA}})) AS gw,
         countIf(event = '$pageview' AND match(properties.$pathname, {{CESTA_OBJEDNAVKA}})) AS order_views
  FROM events
  WHERE timestamp >= toDateTime({{OD_ZAZNAMY}}) AND timestamp < toDateTime({{DO_ZAZNAMY}})
    AND match(coalesce(properties.$pathname, ''), {{CESTA_OBCHOD}})
  GROUP BY sid HAVING order_views > 0 AND purchases = 0 AND gw = 0),
r AS (
  SELECT session_id, dateDiff('second', min(min_first_timestamp), max(max_last_timestamp)) AS dur
  FROM raw_session_replay_events
  WHERE min_first_timestamp >= toDateTime({{OD_ZAZNAMY}}) AND min_first_timestamp < toDateTime({{DO_ZAZNAMY}})
  GROUP BY session_id HAVING dur >= 20)
SELECT multiIf(webview IN ('Facebook', 'Instagram'), 'in-app', os) AS segment,
       if(form_changes > 0, 'formulář', 'bez formuláře') AS kind, count() AS n, groupArray(10)(sid) AS sids
FROM s WHERE sid IN (SELECT session_id FROM r)
GROUP BY segment, kind ORDER BY segment, kind
```

`{{OD_ZAZNAMY}}`–`{{DO_ZAZNAMY}}` = období, kdy se nahrávalo (SKILL §6.1). Prázdný výsledek nejdřív ověř denním počtem záznamů, ne změnou dotazu.
