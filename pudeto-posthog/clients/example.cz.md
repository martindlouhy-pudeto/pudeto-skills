# Example s.r.o. – example.cz (fiktivní ukázka)

Aktualizováno: 2026-11-09 · kdo: ukázka

## PostHog
- Konektor: `PostHog_Example`
- Organizace: Example · Projekt: Example – shop (ID 100001) · region: EU
- Časová zóna projektu: UTC (denní řezy převádět na Europe/Prague: ano)
- Nasazeno: 2026-10-01, GTM verze 12, kontejner GTM-XXXXXXX
- Souhlas: měří se po „Přijmout vše“ (analytics_storage, ad_storage, ad_user_data, ad_personalization)
- Maskování záznamů: všechny vstupy · minimální délka: 2 s · trigger: jen návštěvy se stránkou objednávky
- Kvóta záznamů: 5 000/měsíc od 1. dne v měsíci · poslední den s nahráváním: průběžně

## Rozsah analýz
- Hosty v rozsahu: shop.example.cz
- Produkty mimo funnel: dárkové poukazy → `properties.category NOT IN ('poukazy')`
- Property produktu: `properties.category`
- E-commerce eventy: add_to_cart, view_cart, remove_from_cart, begin_checkout, purchase · neposílá: view_item, add_payment_info
- Platby bez `purchase`: převodem

## Cesty a prvky
- `{{CESTA_OBCHOD}}`: `'^/shop/'`
- `{{CESTA_VYBER}}`: `'/(products|category)'`
- `{{CESTA_OBJEDNAVKA}}`: `'^/shop/checkout$'`
- `{{CESTA_BRANA}}`: `'/payment$'`
- `{{POLE}}`: `'firstname|lastname|email|phone'`
- `{{SOUHLASY}}`: `'terms(_[0-9])?'`
- `{{PLATBA}}`: `'pay-button'` (text tlačítka: „Zaplatit“)
- `{{JINA_PLATBA}}`: `'payment-transfer'`
- Známé dialogy: „Košík vypršel“

## GA4 pro srovnání
- Property: Example | GA4 (ID 100000001) · filtr hostName: shop.example.cz · vyřadit pagePath obsahující: /poukazy

## Kdo je kdo
- Kontakt u klienta: e-shop manažer
- Dodavatel obchodu: agentura vývoje e-shopu
- GTM publikuje: marketér Pudeto

## Historie analýz
| Datum | Co | Výstupy | Poznámka |
| --- | --- | --- | --- |
| 2026-11-09 | vyhodnocení košíku | dokument, slidy | ukázkový řádek |

## Otevřené nálezy
- Ukázkový nález: platba převodem neposílá purchase.
