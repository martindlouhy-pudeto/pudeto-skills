# Nasazení PostHogu – poučení z praxe

Samotné nasazení přes GTM řídí skill **pudeto-gtm** (brána, audit, diff, POTVRZUJI ZÁPIS, Preview, protokol, publikuje člověk). Tady je jen to, co je specifické pro PostHog.

## Projekt a organizace

- Každý klient vlastní organizaci (nebo aspoň projekt) v EU cloudu. Do jedné organizace nemíchat klienty – kvóty záznamů jsou na organizaci.
- Projekt pojmenuj „<Klient> – <obchod>“, ne „Default project“. Přejmenování dělá člověk v nastavení.
- Časová zóna projektu: u českých klientů Europe/Prague, jinak se denní řezy musí převádět.

## Rozsah a souhlas

- Měřit jen tam, kde je otázka: obchod nebo jen košík, ne celý web. Výjimka (blokační pravidlo) v GTM: mimo hosty obchodu / mimo typy stránek košíku nic.
- Consent u obou tagů (Init i e-commerce): analytics_storage, ad_storage, ad_user_data, ad_personalization = jen po „Přijmout vše“. Čísla pak budou nižší než v GA4 – napsat do protokolu.
- Po přepnutí na „Jen nezbytné“ na téže stránce posílá už načtené SDK dál až do dalšího načtení. Doplnit tag s opt-out na odvolání souhlasu.

## Šablona v GTM

Oficiální šablona z GitHubu (v době nasazení nebyla v galerii) měla dvě chyby:

1. `copyFromWindow('posthog')` v sandboxu vrací `undefined` (objekt s funkcemi) → e-commerce tag padal na „posthog is not on the page yet“. Oprava: kontrola `copyFromWindow('posthog.__loaded')` + oprávnění access_globals na `posthog.__loaded` (čtení).
2. Pageview jen při načtení → v SPA obchodu (pushState) chyběly obrazovky. Oprava: `capture_pageview: 'history_change'`.

Init tag jako setup tag e-commerce tagu, `oncePerEvent`; hláška „Re-initializing is a no-op“ v konzoli je neškodná. Až bude šablona v galerii s opravami, zvážit přechod.

## Session replay

- **Maskování:** kde je osobní údaj jako text (stránka s adresou, děkovná stránka), nastavit maskování veškerého textu i obrázků; minimálně maskovat všechny vstupy.
- **Kvóta:** zdarma 5 000 záznamů / měsíc na organizaci. Obchod s tisíci návštěv denně ji vyčerpá za pár dní a pak se nenahrává nic. Od začátku nahrávat jen relevantní návštěvy (trigger na URL nebo event – např. jen návštěvy se stránkou objednávky) nebo sampling. Minimální délka záznamu 2 s.
- Záznamy zvyšují zátěž stránky – u pomalých stránek porovnat rychlost s nahráváním a bez.
- Vedle jiného nástroje na záznamy (Smartlook, Hotjar) běží PostHog jen v rozsahu, kde ten druhý nestačí.

## Ověření po nasazení

- Preview: SDK z EU, token projektu, `$pageview` i po změně URL, e-commerce event s hodnotou, měnou a položkami shodnými s dataLayer, nic mimo rozsah, nic po „Jen nezbytné“, sousední tagy beze změny, kontrolní běh na staré verzi.
- `purchase` se v Preview bez platby neověří – zkontrolovat v PostHogu po první objednávce (Activity) a zapsat.
- Testovací eventy operátora v produkčním projektu zapsat do protokolu (čas, host), ať se nepletou do analýz.
