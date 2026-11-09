# pudeto-skills

Skilly pro [Claude](https://claude.ai), které používáme v agentuře [Pudeto](https://pudeto.cz) při práci v marketingových a analytických nástrojích klientů přes MCP konektory. Každý skill je složka se souborem `SKILL.md` a referencemi.

| Skill | K čemu |
| --- | --- |
| [pudeto-posthog](pudeto-posthog/) | Bezpečná práce v PostHogu přes MCP (jen čtení jako výchozí, správný projekt, osobní údaje, kvóty) a recept na vyhodnocení košíku: funnel, zařízení a OS, prohlížeč v aplikaci, rychlost stránek, záznamy, srovnání s GA4, výstup pro klienta. |

Další skilly (bezpečnostní pravidla, Google Tag Manager, Mergado) přibudou.

## Jak skill použít

1. Stáhněte složku skillu (např. `pudeto-posthog`).
2. Zkopírujte `config.example.md` na `config.md` a vyplňte hodnoty své agentury nebo firmy.
3. Pro každého klienta založte `clients/<domena>.md` ze šablony `clients/_template.md`.
4. Složku zabalte do zipu tak, aby byla v jeho kořeni, a nahrajte ji do Claude jako vlastní skill.

Popis skillu v `SKILL.md` má do 200 znaků, aby prošel nahráním do Claude.

## Co v repu není

`config.md` a klientské soubory zůstávají mimo git (`.gitignore`). Ve veřejných souborech nejsou jména klientů ani dodavatelů, ID účtů ani firemní údaje – jen poučení z praxe.

## Licence

[MIT](LICENSE)
