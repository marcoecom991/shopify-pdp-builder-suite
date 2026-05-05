# Shopify PDP Builder

Claude Code plugin che automatizza la creazione di template Shopify (PDP + Funnel + clone competitor integrale) da zero. Distribuibile a tutti i membri del team: ognuno lo installa una volta, configura i propri store, e lancia uno dei comandi disponibili.

## Comandi disponibili

### `/create-new-pdp`
Crea una nuova PDP duplicando un template prodotto esistente.

Flusso:
1. **Store selection** — scegli su quale store lavorare (Nimea, Glowria, o altri configurati).
2. **Auth + scelta tema** — verifica Theme Access token, mostra temi disponibili, permette di lavorare anche su tema dev/unpublished.
3. **Duplicazione template + liquidify automatica** — sceglie un template PDP esistente, lo duplica con nuovo nome, duplica tutte le sezioni con prefisso derivato. **Ogni sezione duplicata viene resa editabile dal theme editor**: se era già editabile (schema completo) resta com'è; se era hardcoded (tipo GemPages legacy) la skill estrae testi/immagini/CTA dal markup del duplicato e li sposta nello schema come `settings`/`blocks`. Il template sorgente non viene mai toccato.
4. **Push selettivo** — pubblica solo i file nuovi sul tema.
5. **Raccolta materiali** — competitor PDP, transcript, research prodotto, angle, brand assets.
6. **Riscrittura testi** — modalità sezione-per-sezione, batch, o misto. I testi del prodotto base vengono riscritti per il nuovo prodotto modificando i `default` dello schema (non il markup), mantenendo layout/CSS/JS identici.
7. **Guida immagini** — brief per sezione con dimensioni/ratio corretti. Le immagini si caricano **direttamente dal theme editor** (campi `image_picker`), senza passare da URL CDN incollati in chat.
8. **Verifica finale** — checklist mobile/desktop, URL live.

### `/create-new-funnel`
Crea una nuova pagina funnel (advertorial, listicle, quiz, custom) **da zero**.

Flusso:
1. **Store selection** — stesso meccanismo di PDP.
2. **Auth + scelta tema** — come PDP.
3. **Brand identity discovery** — la skill chiede quale PDP usare come riferimento, estrae palette colori + tipografia dal tema, chiede conferma.
4. **Creazione template** — nome, layout (chromeless senza header/footer, o `theme` con chrome), prefisso sezioni. Push del template scheletro.
5. **Tipo di funnel** — advertorial / listicle / quiz / altro. Carica la struttura base di partenza.
6. **Raccolta materiali** — esempi competitor del tipo scelto, research prodotto, **URL PDP** del prodotto (per le CTA), angolo marketing, brand assets extra.
7. **Costruzione sezioni** — modalità sezione-per-sezione / batch / misto. Claude crea sezioni responsive mobile+desktop usando palette e font del brand, testo dal research, bottoni CTA → PDP. **Ogni sezione nasce editabile dal theme editor**: testi, immagini (`image_picker`) e link sono nello schema, mai hardcoded.
8. **Guida immagini** — brief per sezione + upload diretto dal theme editor.

### `/clone-competitor-store`
Replica integrale di uno store competitor (di solito US) sul tuo tema (mercato IT). Diversa dalle altre due perché orchestra la creazione di **tutte le pagine necessarie** (funnel + PDP + home + extras) in un'unica sessione, con brand discovery globale fatta UNA volta dai URL competitor.

Flusso:
1. **Store selection** — come le altre skill.
2. **Auth + scelta tema** — come le altre.
3. **Definizione scope** — selezione multipla: Funnel (advertorial/listicle/quiz) / PDP / Home. Le pagine extra (contattaci, traccia ordine, FAQ, about, blog, collection) vengono chieste alla fine. Out of scope sempre: cart e checkout.
4. **Brand discovery globale** — l'utente manda gli URL competitor di tutte le pagine selezionate, la skill estrae automaticamente palette colori, font, logo, tono di voce via WebFetch + analisi CSS. Una sola volta, riusato per tutte le pagine successive.
5. **Identità nuovo brand** — nome brand, logo (URL CDN da Shopify Files), email assistenza, telefono, indirizzo legale. Lingua testi: IT (la skill traduce ENG → IT mantenendo tono di voce).
6. **Costruzione iterativa** in ordine vincolante **Funnel → PDP → Home**: per ogni pagina screenshot mobile+desktop, mappatura sezioni, creazione template, costruzione sezioni 1:1 (modalità one-shot o sezione-per-sezione), istruzioni a creare Product/Page in Admin e assegnare il template.
7. **Pagine extra opzionali** — contattaci, traccia ordine, FAQ, about, blog, collection (multi-select).
8. **Verifica finale** — checklist mobile/desktop/CTA/editabilità + handoff.

**Hard rule editabilità (CRITICO)**: ogni sezione nasce 100% editabile dal theme editor — testi/immagini/link/colori sempre come `settings`/`blocks` nello schema, mai hardcoded. Le sezioni PDP includono `{"type": "@app"}` nei blocks per permettere di inserire **Katching Bundles, subscription pickers, recensioni Judge.me/Loox** dal theme editor senza toccare il liquid.

**Replica visiva 1:1**: ogni sezione è costruita da zero (no riuso sezioni esistenti del tema), HTML+CSS scritti per matchare lo screenshot/markup competitor pixel-perfect, con classi scoped al prefisso `<store-slug>-<page>-<NN>-<role>`.

## Setup (primo utilizzo, una volta per membro)

### 1. Installa il plugin

```bash
/plugin install https://github.com/<org>/shopify-pdp-builder.git
```

(Se il plugin è distribuito via zip, scompatta in `~/.claude/plugins/shopify-pdp-builder/`.)

### 2. Crea la cartella locale degli store

Fuori dal repo del plugin, in un posto tuo privato (es. `~/Desktop/Shopify-Stores/`):

```
~/Desktop/Shopify-Stores/
├── Nimea/
│   └── workdir/        # ← `shopify theme pull` qui
└── Glowria/
    └── workdir/
```

Per ciascuno store, pull del tema:

```bash
cd ~/Desktop/Shopify-Stores/Nimea/workdir
npx @shopify/cli@latest theme pull --theme <THEME_ID> --store <handle>.myshopify.com
```

### 3. Crea il `.env` per ogni store

Usa lo script helper:

```bash
cd ~/.claude/plugins/shopify-pdp-builder  # o dove hai installato il plugin
./scripts/setup-store.sh
```

Ti chiede interattivamente path workdir, token, handle — scrive `.env` con permessi 600 e testa la connessione.

**Come generare il Theme Access token**:
1. Shopify Admin → Apps → cerca **"Theme Access"** (installala se non c'è, è gratis).
2. Dentro l'app → **Create password** → compila nome + email.
3. Shopify ti manda un'email con un link → aprilo → copi il token (`shptka_...`).
4. Lo incolli quando `setup-store.sh` lo chiede.

**Il token è visibile una sola volta. Se lo perdi, rigeneralo.**

### 4. Compila `config/stores.json`

Copia `config/stores.example.json` → `config/stores.json` e metti i path reali dei tuoi workdir:

```json
{
  "stores": {
    "nimea": {
      "label": "Nimea",
      "workdir": "/Users/<tu>/Desktop/Shopify-Stores/Nimea/workdir",
      "env": "/Users/<tu>/Desktop/Shopify-Stores/Nimea/workdir/.env",
      "theme_id": "195368288591",
      "store_handle": "qncxve-5r.myshopify.com"
    },
    "glowria": {
      "label": "Glowria",
      "workdir": "/Users/<tu>/Desktop/Shopify-Stores/Glowria/workdir",
      "env": "/Users/<tu>/Desktop/Shopify-Stores/Glowria/workdir/.env",
      "theme_id": "196773773653",
      "store_handle": "2hw0d3-wu.myshopify.com"
    }
  }
}
```

`config/stores.json` è nel `.gitignore`, non viene mai committato.

### 5. Testa

In una nuova conversazione Claude Code:

```
/create-new-pdp              # per una nuova pagina prodotto
/create-new-funnel           # per un funnel (advertorial/listicle/quiz)
/clone-competitor-store      # per replicare integralmente uno store competitor
```

Dovrebbero partire tutti dal prompt di scelta store.

## Aggiornamenti

```bash
/plugin update shopify-pdp-builder
```

(Oppure, per distribuzione zip: ri-scompatta la versione aggiornata.)

## Separazione secrets

- **Il plugin** (questa cartella) è pubblicabile/condivisibile. Non contiene mai token o path personali.
- **I `.env`** stanno nei workdir degli store, fuori dal plugin. Mai committati.
- **`config/stores.json`** è per-membro, nel `.gitignore`. Solo `stores.example.json` è committato.

Ogni membro genera il **proprio** Theme Access token — audit Shopify traccia ogni modifica all'utente corretto, e se qualcuno lascia il team basta revocare il suo token.

## Struttura repo

```
shopify-pdp-builder/
├── .claude-plugin/plugin.json     # Manifest plugin
├── skills/
│   ├── create-new-pdp/            # Comando PDP
│   │   ├── SKILL.md
│   │   └── references/ (6 file)
│   ├── create-new-funnel/         # Comando Funnel
│   │   ├── SKILL.md
│   │   └── references/ (8 file)
│   └── clone-competitor-store/    # Comando clone integrale store
│       ├── SKILL.md
│       └── references/ (6 file specifici + riusa references delle altre skill via path relativo)
├── config/
│   ├── stores.example.json        # Template (committato)
│   └── stores.json                # Reale (NON committato)
├── scripts/setup-store.sh
├── .env.example
├── .gitignore
└── README.md
```

Reference files per skill:

`skills/create-new-pdp/references/` (6 file):
- `workflow-faithful-rebuild.md` — regola "non riscrivere markup, solo testi"
- `section-schema-patterns.md` — liquidify + pattern schema editabile (tipi setting, blocks, edge case)
- `auth-pattern.md`, `selective-push.md`, `section-naming.md` — infrastruttura condivisa
- `image-specs-per-section.md` — specs immagini PDP

`skills/create-new-funnel/references/` (8 file):
- `funnel-types.md` — struttura advertorial/listicle/quiz
- `brand-identity-discovery.md` — estrazione colori/font da una PDP esistente
- `page-template-layout.md` — chromeless vs theme layout
- `funnel-image-specs.md` — specs immagini per funnel
- `section-schema-patterns.md` — pattern schema editabile (condiviso con PDP)
- `auth-pattern.md`, `selective-push.md`, `section-naming.md` — copie infrastruttura

`skills/clone-competitor-store/references/` (6 file specifici):
- `competitor-discovery.md` — WebFetch + estrazione palette/font/logo/tono dai URL competitor
- `visual-replication.md` — regole per ricostruire design 1:1 da screenshot mobile + desktop, breakpoint, classi scoped
- `editability-and-app-blocks.md` — hard rule editabilità schema + supporto `{"type": "@app"}` per Katching Bundles, subscription, recensioni
- `localization-it.md` — traduzione ENG→IT, claim regulatory IT (cosmetici, integratori), adattamento valute/unità/formati
- `home-template.md` — gestione `index.<slug>.json` + assignment manuale come homepage
- `page-types-extras.md` — template per contattaci, traccia ordine, FAQ, about, blog, collection (chiamate alla fine del flusso principale)

Inoltre la skill riusa via path relativo (`../create-new-pdp/references/...` e `../create-new-funnel/references/...`) i file di infrastruttura comune (auth, naming, push selettivo, schema patterns, brand discovery, funnel types).

## Troubleshooting

| Problema                                                       | Soluzione                                                                            |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `/create-new-pdp` (o gli altri comandi) non esistono           | Hai installato il plugin? Apri nuova sessione Claude Code dopo `/plugin install`.    |
| `401 Unauthorized` durante push                                | Token scaduto/revocato. Rigenera da Theme Access e rilancia `setup-store.sh`.        |
| "Store non trovato in config/stores.json"                      | Crea il file da `stores.example.json` e aggiungi il tuo store.                       |
| Push rifiutato con "Theme is live"                             | Aggiungi `--allow-live` (già incluso nei template della skill).                      |
| `command not found: npx`                                       | Installa Node.js (≥ 18) da nodejs.org, poi riprova.                                  |
| Sezione sbagliata dopo push                                    | La skill ha modificato un file diverso dal prefisso atteso? Controlla `--only`.      |
| `/clone-competitor-store` — Katching Bundles non si inserisce  | Verifica che la sezione PDP main abbia `{"type": "@app"}` nello schema blocks.       |
| `/clone-competitor-store` — WebFetch ritorna pagina vuota      | Il sito competitor è JS-rendered. Fornisci screenshot + inspect element manualmente. |

## Roadmap

- v2: creazione Product/Page via Admin API (ora è manuale in Admin).
- v2: upload immagini via Files API (ora le carichi manualmente).
- v2: logging audit `logs/<timestamp>-<pdp>.md`.

## Licenza

Privato — uso interno team.
