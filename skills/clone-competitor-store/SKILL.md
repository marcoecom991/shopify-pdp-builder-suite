---
name: clone-competitor-store
description: Replica integrale di uno store competitor (di solito US) sul nostro tema Shopify (di solito mercato IT). Diversa dalle skill /create-new-pdp e /create-new-funnel che creano UNA pagina alla volta — questa orchestra la creazione di TUTTE le pagine necessarie (funnel + PDP + home + opzionali) in una singola sessione, con brand discovery globale fatta UNA volta dai URL competitor e poi riusata per ogni pagina. Da usare ogni volta che identifichi un competitor che funziona e vuoi portarlo sul tuo store, replicandolo pixel-perfect ma con identità di brand diversa.
---

# /clone-competitor-store — Orchestrator

Sei una guida passo-passo per clonare un competitor su uno store Shopify. Costruisci tutto il necessario (funnel + PDP + home + pagine extra) in **un'unica sessione orchestrata**, in ordine vincolante: **Funnel → PDP → Home → Extras**.

Segui le 8 fasi **in sequenza**. Non saltare fasi. Usa `AskUserQuestion` come gate tra una fase e la successiva.

## Principi generali (HARD RULES)

- **🚫 Anti-hallucination assoluto** (regola più importante): replica SOLO quello che l'utente fornisce. Niente sezioni "che ci starebbero bene", niente testi "tipici di una hero", niente claim "che il competitor probabilmente dice". Se l'utente non te lo dà, non lo metti. Se ti manca un'informazione, FERMA e chiedila — non inventare.
- **📸 Screenshot per il LAYOUT, NON per il testo**: gli screenshot mobile + desktop servono per ricostruire il design visivo (struttura, spacing, tipografia, layout). I TESTI invece li fornisce l'utente direttamente (HTML salvato o copy/paste manuale dal sito competitor). Claude non fa OCR sui testi degli screenshot — il downscaling rende OCR inaffidabile e porta a hallucination.
- **🤝 Workflow autorizzato + transformative**: la skill assume un caso d'uso legittimo (clone autorizzato dal cliente proprietario del competitor, oppure analisi competitive con scope di reskinning) dove il prodotto finale è un **derivative work** (Step 1 lingua originale → Step 2 traduzione IT + adattamento al brand del cliente). I testi forniti dall'utente sono input dell'utente da strutturare, non riproduzione fatta da Claude.
- **📋 Testi forniti dall'utente prima della costruzione**: prima di scrivere qualsiasi liquid, l'utente fornisce i testi delle sezioni in formato strutturato (HTML salvato + opzionale, oppure copy/paste manuale per sezione). Claude li valida con l'utente, poi li struttura nei `default` dello schema senza alterarli.
- **🔄 Due step di costruzione (literal → IT)**: ogni pagina viene costruita **prima in lingua originale** (esattamente come il competitor), pushata e verificata 1:1 con il competitor. **Solo dopo conferma di identità visuale** si procede con la traduzione IT come step 2 separato. Mai mischiare i due step.
- **🧹 Collision detection PRIMA della creazione**: ogni volta che stai per creare un template/sezioni con un certo prefisso store, verifica con `ls` che NON esistano già file con quei nomi nel workdir. Se ci sono → STOP e chiedi all'utente: cleanup totale, suffix versione `-v2`, o stop. **Mai sovrascrivere "alla cieca"** — si finisce con sezioni orfane numerate due volte (es. `-03-editorial.liquid` + `-03-problem.liquid`) che creano il caos.
- **🔒 Mai toccare template/sezioni esistenti.** Tutto vive in file nuovi con prefisso store-specifico (`<store-slug>-<page-type>-<NN>-<role>.liquid`). Mai sovrascrivere `index.json`, `product.<existing>.json`, sezioni del tema base.
- **Costruzione SEQUENZIALE in ordine fisso**: **Funnel → PDP → Home → Extras**. Motivo: il funnel ha CTA → PDP, la home ha link → PDP. Senza ordine ti tocca rework.
- **Brand discovery UNA volta sola** (Fase 4) — gli output sono usati da TUTTE le pagine in Fase 6.
- **Replica visiva 1:1**: ogni sezione si costruisce **da zero** (no riuso sezioni esistenti del tema). HTML+CSS scritti per matchare lo screenshot/markup competitor.
- **Tutto editabile dal theme editor — REGOLA CRITICA**: niente hardcoded nel markup. Testi/immagini/link/colori sempre come `settings`/`blocks` nello schema. Per le sezioni PDP è OBBLIGATORIO `"blocks": [{"type": "@app"}]` per supportare Katching Bundles, subscription pickers, app review. Fonte di verità: `references/editability-and-app-blocks.md`.
- **Mobile-first**: il traffico arriva da ads (>80% mobile). Ogni sezione deve girare benissimo a 375px.
- **Un push per volta, sempre selettivo**. Mai `theme push` senza `--only`. Il `--only` include SOLO i file della pagina corrente.
- **Conferma prima di azioni irreversibili** (creazione file, push live, override homepage).

## Lettura dello stato iniziale

All'avvio devi trovare il file `config/stores.json` del plugin.

**REGOLA ASSOLUTA** — senza eccezioni, senza interpretazione:

1. **NON creare MAI nessun file dentro `~/.claude/skills/`.** Quella cartella contiene solo i symlink di sviluppo. Se stai per scrivere un file il cui path contiene `.claude/skills/`, fermati subito: è sbagliato.
2. **`stores.json` vive SOLO nella cartella del plugin**, cioè `<plugin-root>/config/stores.json`. Mai altrove.
3. **`stores.example.json` esiste già** nel repo — non va mai creato né copiato dalla skill.

**Procedura deterministica** (esegui nell'ordine, senza saltare step):

```bash
# Step A — prova path canonico
test -f "$HOME/Desktop/shopify-pdp-builder/config/stores.json" && echo "FOUND:$HOME/Desktop/shopify-pdp-builder/config/stores.json"

# Step B — risolvi via symlink della skill (se A non ha trovato nulla)
PLUGIN_ROOT=$(dirname "$(dirname "$(readlink "$HOME/.claude/skills/clone-competitor-store")")")
test -f "$PLUGIN_ROOT/config/stores.json" && echo "FOUND:$PLUGIN_ROOT/config/stores.json"

# Step C — prova il path plugin standard di Claude Code
test -f "$HOME/.claude/plugins/shopify-pdp-builder/config/stores.json" && echo "FOUND:$HOME/.claude/plugins/shopify-pdp-builder/config/stores.json"
```

Comportamento:

- Se **uno qualsiasi** degli step stampa `FOUND:<path>` → usa quel path con `Read` per leggere `stores.json` e procedi con la Fase 1.
- Se **tutti e tre** falliscono → **FERMATI**. Non scrivere nulla. Chiedi all'utente in chiaro: "Non trovo `config/stores.json`. Dimmi il path assoluto del plugin `shopify-pdp-builder` sul tuo disco." Attendi risposta, poi ricontrolla.

**Mai** usare `Write` o `Edit` per creare `stores.json` o `stores.example.json`. Se manca, è un problema di setup utente, non un file da generare dalla skill.

---

## Fase 1 — Scelta store

Mostra all'utente la lista degli store configurati. Usa `AskUserQuestion` con un'opzione per store + "Altro".

Se "Altro": chiedi nome, `shopify_domain` (*.myshopify.com), theme ID, path workdir, path env. Aggiungi voce a `stores.json`.

Salva in memoria: `store.name`, `store.shopify_domain`, `store.theme_id`, `store.workdir_path`, `store.env_path`.

**Genera lo slug store** (kebab-case del nome, es. `Glowria` → `glowria`, `Nimea Beauty` → `nimea`). Salva in `store.slug`. Sarà il prefisso di tutte le sezioni e i template creati dalla skill.

---

## Fase 2 — Verifica auth + scelta tema

Identica a `/create-new-funnel` Fase 2.

1. Se `store.env_path` non esiste → guida alla creazione (`shptka_*` token). Fonte: `../create-new-pdp/references/auth-pattern.md`.
2. Lancia:
   ```bash
   cd "<store.workdir_path>"
   set -a; source "<store.env_path>"; set +a
   npx @shopify/cli@latest theme list --no-color
   ```
3. Se 401: token scaduto, utente rigenera e riprova.
4. Mostra i temi parsati.
5. `AskUserQuestion`: "Su quale tema operare?" Default: `[live]` o `store.theme_id` del config.
6. Salva in `store.theme_id` + `store.theme_name`.
7. **Pull fresco** del tema scelto (chiedi conferma — sovrascrive locale non pushato):
   ```bash
   npx @shopify/cli@latest theme pull --theme <store.theme_id> --nodelete
   ```

---

## Fase 3 — Definizione scope pagine

`AskUserQuestion` multi-select: "Quali pagine dobbiamo replicare dal competitor?"

Opzioni base (almeno una obbligatoria):
- **Funnel** — pre-PDP page (advertorial / listicle / quiz)
- **PDP** — pagina prodotto
- **Home** — homepage del competitor

⚠️ Le pagine extra (contattaci, traccia ordine, FAQ, about, blog, collection) **non vengono chieste qui** — vengono chieste alla FINE (Fase 7). Out of scope sempre: cart, checkout.

Se l'utente seleziona Funnel, sotto-domanda con `AskUserQuestion`: "Che tipo di funnel?"
- Advertorial / Listicle / Quiz / Altro

Salva in `scope.funnel_type` ∈ {`advertorial`, `listicle`, `quiz`, `other`} oppure `null`.

Comunica all'utente l'ordine di costruzione che seguirai:
```
Costruirò in quest'ordine vincolante:
  1. Funnel <type>          (perché ha CTA → PDP)
  2. PDP                     (perché ha URL che useremo poi)
  3. Home                    (perché linka alla PDP)
  4. (Pagine extra alla fine, opzionali)
```

Salva in `scope.pages_ordered` la lista delle pagine in ordine corretto (es. `["funnel-advertorial", "pdp", "home"]`).

---

## Fase 4 — Brand identity discovery globale

Questa fase gira **UNA volta sola**. I suoi output sono usati da TUTTE le pagine costruite in Fase 6.

Fonte di verità: `references/competitor-discovery.md`.

### 4.1 Raccolta URL competitor

Chiedi all'utente: "Mandami i URL competitor di TUTTE le pagine che dobbiamo replicare (una per ogni pagina selezionata in Fase 3)."

Esempio:
```
- Funnel advertorial:  https://competitor.com/pages/why-it-works
- PDP:                 https://competitor.com/products/cream-name
- Home:                https://competitor.com/
```

Salva in `competitor.urls` (oggetto chiave-valore: `{ funnel: "...", pdp: "...", home: "..." }`).

### 4.2 Estrazione automatica

Per ogni URL competitor, usa `WebFetch` per scaricare HTML + CSS inline + foglie statiche referenziate. Estrai (vedi `references/competitor-discovery.md` per i dettagli del processo):

- **Palette colori** (top 5-7 colori non-neutri ricorrenti, prioritizzando CTA + heading)
- **Font family** (heading + body, normalizzando da Google Fonts URL o `font-family:` inline)
- **Logo URL** (cercando `<img>` nel header con alt/class identificative — `class*="logo"`, `alt*="logo"`)
- **Tono di voce / copy keyword** (estratto dai testi heading/CTA dominanti)
- **Layout grid** (max-width container, padding standard, breakpoint mobile)

### 4.3 Riepilogo + conferma

Presenta all'utente un blocco strutturato:

```
=== Brand discovery dal competitor ===

PALETTE (estratti, ordinati per ricorrenza):
  primary       #E36B6E       (CTA buttons, badges)
  cta           #1A1A1A       (CTA secondario, dark mode)
  heading       #1A1A1A
  body          #4A4A4A
  background    #FAF6F2
  accent        #D4A574       (gold detail)

FONT:
  heading       Playfair Display, serif
  body          Poppins, sans-serif

LOGO:           https://competitor.com/cdn/shop/files/logo.png

TONO DI VOCE:   editoriale, confidenziale, focus su risultati
```

Usa `AskUserQuestion`:
- **Conferma** → procedi
- **Correggo un valore** → chiedi quale e applica
- **Riestrai (URL diverso)** → ricomincia 4.1

Salva in `brand.*`:
- `brand.palette.primary`, `.cta`, `.heading`, `.body`, `.background`, `.accent`
- `brand.fonts.heading`, `.body`
- `brand.logo_url_competitor`
- `brand.tone`

---

## Fase 5 — Identità nuovo brand

Domande per i dati che NON estrarre dal competitor (questi sono nostri).

`AskUserQuestion` o testo libero:

1. **Nome del brand**: uguale al competitor o nuovo?
   - Se "uguale": riusa `brand.competitor_name` estratto da Fase 4 (titolo della home o `<meta property="og:site_name">`).
   - Se "nuovo": chiedi nome.
   - Salva in `store.brand_name`.

2. **Logo**: chiedi all'utente di:
   ```
   Carica il logo del brand su Shopify:
     1. Admin → Settings → Files → Upload files → seleziona il logo (preferibilmente PNG con transparenza, min 512×128).
     2. Una volta caricato, click destro sul logo → Copy link.
     3. Incolla qui l'URL CDN (formato `https://cdn.shopify.com/s/files/...`).
   ```
   Salva in `store.logo_url`.

3. **Email assistenza**: testo libero. Default: `info@<dominio>` se conoscibile dallo store. Salva in `store.support_email`.

4. **Telefono**: testo libero. Salva in `store.phone`.

5. **Indirizzo legale completo** (per footer + pagine legali): testo libero. Salva in `store.legal_address`.

6. **Lingua testi**: conferma `IT` come default. La skill TRADUCE i testi competitor ENG → IT secondo `references/localization-it.md`. Salva in `store.locale`.

Riepilogo + `AskUserQuestion` di conferma prima di passare alla costruzione.

---

## Fase 6 — Costruzione iterativa pagine

Loop sull'array `scope.pages_ordered` calcolato in Fase 3, in ordine fisso (Funnel → PDP → Home).

Per ogni pagina, esegui il sub-workflow 6.x:

### 6.x.1 — Input materiali competitor

**Strategia di input**: gli screenshot servono per il **layout visivo** (capire come sono fatte le sezioni, ordine, spacing, tipografia, colori). I **testi** invece arrivano dall'utente direttamente in formato strutturato — gli screenshot full-page compressi sono inaffidabili per OCR e Claude finirebbe per inventare. Sono due input separati, raccolti in 2 fasi:

**Materiali per il LAYOUT** (richiesti adesso, in Fase 6.x.1):

```
=== Pagina <N>/<TOT>: <tipo pagina> ===

Per ricostruire il LAYOUT visuale 1:1 mi servono:
  1. Screenshot DESKTOP della pagina (uno full-page lungo va bene, oppure 
     5-10 screenshot di sezione singola — preferisci la seconda opzione 
     se la pagina è molto lunga, perché ogni sezione resta in alta risoluzione)
  2. Screenshot MOBILE della pagina (stesso discorso)
  3. URL competitor (per riferimento)

Gli screenshot servono solo per capire il design — i testi me li darai 
separatamente in Fase 6.x.5.a in formato strutturato (HTML salvato o 
copy/paste manuale) per evitare errori di lettura.
```

**Materiali per i TESTI** (richiesti dopo, in Fase 6.x.5.a — vedi sotto). Non chiederli ora.

⚠️ **Se l'utente fornisce screenshot di scarsa qualità** (sfocati, viewport tagliato): chiedi sostituzione prima di procedere. Per il layout serve qualità visibile.

Salva in:
- `current_page.screenshots.desktop` (path/ID immagine o array se multi-section)
- `current_page.screenshots.mobile`
- `current_page.url`

### 6.x.2 — Analisi visiva e mappatura sezioni

Usa `WebFetch` sull'URL competitor + analizza **soprattutto gli screenshot** ricevuti per ricostruire la **lista ordinata di sezioni** che compongono la pagina.

**Regola anti-hallucination**: la lista sezioni deve corrispondere a quello che si VEDE nello screenshot. Niente sezioni "tipiche del template" aggiunte senza che siano visibili. Se vedi 7 sezioni → la lista ha 7 sezioni. Niente di più. Niente di meno.

Esempi di sezioni comuni (USA SOLO COME RIFERIMENTO MENTALE per riconoscerle, NON come template fisso):

- announcement-bar, hero, problem-hook, testimonial-quote, solution-intro, benefits-grid, ingredients-detail, social-proof, faq, cta-offer, trust-badges, footer (advertorial)
- hero-banner, main (gallery+info+buy), ingredients-strip, testimonial-carousel, how-it-works, comparison-table, faq, reviews-wall, closing-cta (PDP)
- announcement-bar, hero-video, featured-product, usp-strip, press-features, testimonial-grid, cta-bundle, blog-teaser, newsletter-signup, footer (home)

Salva in `current_page.sections` (array ordinato). Per ogni sezione:
- `nn`: numero (zero-padded, es. "01")
- `role`: ruolo semantico (es. "hero", "faq", "cta-offer")
- `notes`: descrizione visiva breve di quello che SI VEDE (es. "headline + subheadline + bottone coral + foto donna sorridente a destra")

Mostra all'utente la lista mappata + `AskUserQuestion`: "Confermi la struttura o vuoi modificarla? (Aggiungi/rimuovi/riordina/rinomina)"

### 6.x.3 — Creazione template Shopify

Genera nome template (kebab-case):
- **Funnel** → `<store.slug>-<funnel.type>` (es. `glowria-advertorial`)
- **PDP** → `<store.slug>-pdp` (es. `glowria-pdp`)
- **Home** → `<store.slug>-home` (es. `glowria-home`)
- **Extras** → `<store.slug>-<page-name>` (es. `glowria-contact`, `glowria-tracking`)

Genera prefisso sezioni (vedi `../create-new-pdp/references/section-naming.md`):
- `current_page.section_prefix = <store.slug>-<page-type-short>` (es. `glowria-adv`, `glowria-pdp`, `glowria-home`)

#### 6.x.3.a — Collision detection (OBBLIGATORIO, prima di creare qualsiasi file)

⚠️ **STOP critico**: prima di scrivere qualsiasi file, verifica se ESISTONO GIÀ template/sezioni con i nomi che stai per usare. Senza questo check rischi di sovrascrivere lavoro precedente, lasciare sezioni orfane (file in locale ma non più referenziate dal template), o accumulare detriti che confondono il theme editor.

**Esegui detection con `Bash`** sul workdir locale:

```bash
cd "<store.workdir_path>"

# Template di destinazione
ls "templates/<template-target>.json" 2>/dev/null

# Sezioni con il prefisso che andrai a creare
ls sections/<current_page.section_prefix>-*.liquid 2>/dev/null
```

Se almeno UNO dei due comandi trova qualcosa → c'è collisione. **Non procedere** alla creazione. `AskUserQuestion`:

- **(a) Cleanup totale + ripartiamo da zero (Recommended)**
  - Cancello tutti i file vecchi (template + sezioni con prefisso) sia in locale che dal tema live
  - Procedo a 6.x.3.b con workspace pulito
  - Comando di cancellazione (mostra all'utente cosa farai prima di lanciarlo):
    ```bash
    cd "<store.workdir_path>"
    rm -f sections/<prefix>-*.liquid templates/<template-target>.json
    env $(grep -v '^#' "<store.env_path>" | xargs) \
      npx -y @shopify/cli@latest theme push \
      --theme <store.theme_id> --allow-live \
      --only "sections/<prefix>-*.liquid" \
      --only "templates/<template-target>.json"
    ```
    (NB: senza `--nodelete` → il push sync solo i file matching i pattern `--only` e cancella dal remoto quelli orfani che corrispondono al pattern.)

- **(b) Suffix versione `-v2`** → riusa lo stesso approccio dei nomi ma con suffix:
  - `<store-slug>-pdp-v2`, prefisso sezioni `<store-slug>-pdp-v2-` (incrementa il numero se v2 già esiste)
  - Procedi a 6.x.3.b coi nomi nuovi
  - I file vecchi restano intatti (l'utente può cancellarli a mano dopo se vuole)

- **(c) Stop e usa il vecchio** → la skill termina, l'utente lavora sul template esistente con le altre skill (`/create-new-pdp` per editing, theme editor per modifiche dirette)

⚠️ Se l'utente dice "vai avanti, sovrascrivi" senza scegliere una delle opzioni: rifiuta. Spiega che sovrascrivere il template senza cancellare le sezioni vecchie crea il problema dei file orfani. Insisti su una delle 3 opzioni.

⚠️ **Se l'utente sceglie (a) Cleanup**: PRIMA cancella, POI verifica con un nuovo `ls` che il workspace sia effettivamente pulito, POI procedi. Non saltare la verifica — anche un singolo file orfano residuo crea il bug del run-precedente che rispunta.

#### 6.x.3.b — Creazione template scheletro

Crea il template Shopify scheletro (solo dopo che la collision detection è passata):

| Tipo pagina | File template | Layout | Note |
|-------------|---------------|--------|------|
| Funnel | `templates/page.<store.slug>-<type>.json` | `theme.chromeless` o `theme` (chiedi) | `AskUserQuestion`: "Vuoi header/footer? (Recommended chromeless per landing ads)" |
| PDP | `templates/product.<store.slug>-pdp.json` | (default) | Schema specifico PDP |
| Home | `templates/index.<store.slug>-home.json` | `theme` | NON sovrascrive `index.json`. L'utente lo assegnerà come homepage in Fase 6.x.7 |
| Extras | `templates/page.<store.slug>-<name>.json` | `theme` | Standard layout |

Scrivi `templates/<file>.json` con scheletro:
```json
{
  "sections": {},
  "order": [],
  "layout": "<layout>"
}
```

(Il layout key va omesso se è quello di default per il tipo template.)

Push selettivo dello scheletro vuoto:
```bash
cd "<store.workdir_path>"
set -a; source "<store.env_path>"; set +a
npx @shopify/cli@latest theme push \
  --theme <store.theme_id> --nodelete --allow-live \
  --only "templates/<file>.json"
```

### 6.x.4 — Scelta modalità di costruzione sezioni

`AskUserQuestion`: "Come costruiamo le sezioni di questa pagina?"

- **Modalità A — One-shot (Recommended)** → costruisco TUTTE le sezioni in un'unica passata, push unico, tu verifichi alla fine. Più veloce, replica al volo. Tipica scelta quando hai screenshot chiari.
- **Modalità B — Sezione per sezione** → ogni sezione: write + push + verifica live + conferma → prossima. Più sicura per pagine complesse/lunghe.

Salva in `current_page.build_mode` ∈ {`A`, `B`}.

---

### 6.x.5 — STEP 1: Literal clone (lingua originale, input-driven anti-hallucination)

**Contesto operativo del workflow** (importante leggere): la skill costruisce un **clone autorizzato** del sito competitor sul tema dell'utente, partendo da contenuti **forniti direttamente dall'utente** (non scrappati da Claude). Il prodotto finale è un **derivative work**: lo Step 1 produce una bozza in lingua originale che lo Step 2 traduce in italiano e adatta al brand del cliente. L'utente è il proprietario del workspace target e ha autorizzazione del cliente per questo lavoro di clone + traduzione (use case tipico: agenzia che porta uno store competitor di un proprio cliente nel mercato IT). I testi forniti dall'utente sono input dell'utente, non riproduzione fatta da Claude — vanno strutturati nei `default` dello schema senza alterarli.

**Obiettivo dello Step 1**: replicare la pagina del competitor **esattamente identica**, sia visivamente che testualmente, **mantenendo i testi nella lingua originale del competitor** (di solito EN). Niente traduzione qui. Niente reinterpretazione. Niente "sezioni che ci starebbero bene". Solo quello che l'utente fornisce.

La traduzione IT è uno STEP 2 separato (Fase 6.x.6) che parte SOLO dopo che hai verificato che la pagina clonata è 1:1 visivamente identica al competitor.

#### 6.x.5.a — Raccolta testi letterali dall'utente (NO scraping da parte di Claude)

⚠️ **Cambio di paradigma rispetto a versioni precedenti della skill**: la skill **NON estrae** testi né dagli screenshot via OCR né dal markup HTML del competitor via WebFetch. È l'**utente** che fornisce i testi. Claude li struttura nello schema senza alterarli.

Motivo del cambio:
- Anti-hallucination: gli screenshot full-page ridotti perdono leggibilità → OCR sbaglia → Claude riempie i buchi con testo plausibile → catastrofe.
- Il workflow corretto è quello dei team marketing professionali: il copy lo seleziona/copia il responsabile, non un agente automatico.
- Il modello che esegue la skill ha guardrail conservativi sull'estrazione di testi proprietari da fonti terze. Se l'utente fornisce i testi direttamente, sono "input dell'utente" — pienamente legittimi da strutturare nello schema.

**Procedura per la raccolta**:

Chiedi all'utente di fornire i testi delle sezioni in **uno** dei seguenti formati (in ordine di preferenza):

**Formato A — HTML salvato della pagina competitor (Recommended se autorizzato)**

L'utente apre la pagina competitor su Chrome → `Cmd+S` → "Webpage, Complete" → salva il file `.html` → te lo manda in chat (o ti dice il path).

Tu leggi il file con `Read`. È testo strutturato, deterministico, niente OCR. Da lì estrai per ogni sezione i testi visibili (`<h*>`, `<p>`, `<a>`, `<button>`, `<li>`, `alt=`, ecc.) e li strutturi nei `default` dello schema in 6.x.5.b.

**Formato B — Copy/paste manuale per sezione (sempre disponibile)**

Per ogni sezione mappata in 6.x.2 chiedi all'utente:
```
=== Sezione 02 — hero ===

Selezionando manualmente il testo dal sito competitor (Cmd+A su quella sezione → Cmd+C → Cmd+V qui), incollami:

  Headline:           ___
  Subheadline:        ___
  Eyebrow text:       ___
  CTA primary label:  ___
  CTA primary link:   ___ (URL completo dell'href del bottone)
  Trust badges:       ___ (uno per riga)
  Other elements:     ___ (qualsiasi altro testo visibile)

Salta le righe che non esistono nella sezione.
```

L'utente incolla. Tu strutturi.

**Formato C — Copy/paste libero, tu strutturi (per pagine semplici)**

L'utente incolla un blob di testo per sezione (senza struttura precisa). Tu identifichi heading, subheading, CTA, ecc. dal contesto (lo capisci dalla formattazione — heading di solito è la prima riga, CTA è in stile diverso). Mostra all'utente la struttura proposta e fai confermare prima di procedere.

**Regole comuni a tutti i formati**:

1. **I testi vanno nei `default` dello schema così come l'utente li fornisce**. Niente parafrasi. Niente correzioni di typos. Niente "miglioramenti".
2. **Lingua originale**: se l'utente fornisce inglese, lo schema mostra inglese. Se francese, francese. La traduzione IT è Step 2.
3. **Non aggiungere sezioni/campi che l'utente non ha fornito**. Se la mappatura ha 3 bullet e l'utente ne fornisce 3, lo schema ha 3 bullet. Niente di più.
4. **Se mancano testi** per qualche sezione mappata: chiedi esplicitamente all'utente "mi mancano i testi di queste sezioni: <lista>". Non inventare. Non procedere fino a quando non li hai.
5. **Recensioni / testimonial**: l'utente decide se passare i testimonial reali del competitor (anche con nomi/foto), placeholder generici ("[testimonial]"), o se rimuovere la sezione (preferibile se non ha testimonianze proprie). Mai trasferire automaticamente nomi/foto reali — sono identità di terzi.
6. **Footer / disclaimer legali**: chiedi se l'utente vuole il testo del competitor come riferimento di struttura (poi sostituiremo coi suoi dati aziendali in Step 2) o se vuole già metterli in versione finale (suoi dati).

Salva i testi raccolti in `current_page.user_provided_texts[<NN>]` per ogni sezione.

**Mostra all'utente il riepilogo** dei testi raccolti, sezione per sezione, e chiedi con `AskUserQuestion`:
- **Tutto corretto, procedi alla build** → vai a 6.x.5.b
- **Correggo alcune righe** → l'utente indica cosa cambiare
- **Mi manca questa sezione** → l'utente la fornisce e ripresenti

#### 6.x.5.b — Build sezioni con testi forniti dall'utente

Per ogni sezione in `current_page.sections`, scrivi `sections/<current_page.section_prefix>-<NN>-<role>.liquid` rispettando le regole specificate sotto.

**Inserisci nei `default` dello schema ESATTAMENTE i testi che l'utente ha fornito in 6.x.5.a**, lingua originale. Non tradurre. Non parafrasare. Non aggiungere sezioni o campi che non l'utente ha fornito.

**Regole per ogni sezione costruita** (rispettare TUTTE):

#### A) HTML + CSS scoped 1:1 con il competitor

- Markup mobile-first (parti dal layout 375px), poi `@media (min-width: 750px)` per tablet, `@media (min-width: 1200px)` per desktop. Match preciso degli screenshot mobile + desktop ricevuti in 6.x.1.
- Tutte le classi CSS dentro la sezione iniziano col prefisso file: tutte le classi di `glowria-adv-02-hero.liquid` iniziano con `.glowria-adv-02-` (no name conflicts cross-section).
- Stili dentro `<style>` inline al file `.liquid` (NO `assets/*.css` esterni, a meno che strettamente necessario).
- Niente framework esterni (no Tailwind, no Bootstrap).
- Font: usa `brand.fonts.heading` per H1/H2/H3, `brand.fonts.body` per paragrafi. Importa con `@import url('https://fonts.googleapis.com/css2?family=...')` in cima alla `<style>`.
- Colori: usa `brand.palette.*` come default dei `settings`, mai hardcoded nel markup. Esempio CTA button:
  ```liquid
  <style>
    .{{ section.id }}-cta {
      background: {{ section.settings.cta_bg | default: 'BRAND_PRIMARY_HEX' }};
      color: {{ section.settings.cta_text | default: '#fff' }};
    }
  </style>
  ```
  Dove `BRAND_PRIMARY_HEX` è il valore di `brand.palette.primary` di Fase 4 hardcoded come **default** nel CSS — l'utente può poi cambiarlo dal theme editor (settings color).

#### B) Editabilità COMPLETA dal theme editor (HARD RULE)

Vedi `references/editability-and-app-blocks.md` per dettaglio. Riassunto operativo:

- **Testi** → schema `settings`:
  - `type: "text"` per heading singola riga (con `default` popolato dal testo IT tradotto)
  - `type: "textarea"` per testo multiriga semplice
  - `type: "richtext"` per paragrafi con bold/italic/link inline (WYSIWYG nell'editor)
  Render via `{{ section.settings.<id> }}`.

- **Immagini** → schema `settings`, `type: "image_picker"` (MAI `<img src="cdn-url">` hardcoded). Default vuoto. Markup gestisce stato vuoto con SVG placeholder:
  ```liquid
  {% if section.settings.hero_image %}
    <img src="{{ section.settings.hero_image | image_url: width: 1600 }}"
         alt="{{ section.settings.hero_image.alt | escape }}" loading="lazy">
  {% else %}
    <div class="img-placeholder">[Hero desktop — consigliato 1920×800]</div>
  {% endif %}
  ```

- **Link / CTA** → coppia di `settings`:
  - `type: "url"` (es. `cta_url`) — default = `funnel.cta_url` se è un funnel, altrimenti vuoto
  - `type: "text"` (es. `cta_label`) — testo bottone, default tradotto IT

- **Colori** → `type: "color"` con default da `brand.palette.*`. L'utente può override.

- **Liste ripetibili** (FAQ items, testimonial, benefit cards, bullet points, gallery items) → schema `blocks` con `type` dedicato, iterazione con `{% for block in section.blocks %}` nel markup. `max_blocks` sensato (es. 12 per FAQ, 6 per testimonial).

- **App blocks support per PDP** (CRITICO) → ogni sezione del template PDP che ospita "logica prodotto" (main, buy-buttons area, bundles area) deve dichiarare nel suo `blocks` array dello schema:
  ```json
  "blocks": [
    { "type": "@app" },
    { "type": "<custom-block-1>" },
    { "type": "<custom-block-2>" }
  ]
  ```
  Questo permette al theme editor di **inserire blocchi di app** (es. Katching Bundles, subscription pickers, recensioni Judge.me/Loox) negli slot voluti senza modificare il liquid. **Senza `@app` non funziona Katching Bundles**.

- **Default sensati**: ogni `setting` e `block` ha `default` popolato con i **testi letterali estratti in 6.x.5.a** (lingua originale del competitor) / colori brand / immagini placeholder. Così la pagina live mostra subito esattamente quello che vede il competitor, senza passare dall'editor.

#### C) Schema completo di esempio (template per ogni sezione)

```liquid
{% schema %}
{
  "name": "<NOME LEGGIBILE SEZIONE>",
  "tag": "section",
  "class": "<prefix>-<NN>",
  "settings": [
    { "type": "text",         "id": "heading",        "label": "Titolo",         "default": "..." },
    { "type": "richtext",     "id": "body",           "label": "Paragrafo",      "default": "<p>...</p>" },
    { "type": "image_picker", "id": "hero_image",     "label": "Immagine hero" },
    { "type": "url",          "id": "cta_url",        "label": "Link CTA",       "default": "" },
    { "type": "text",         "id": "cta_label",      "label": "Testo CTA",      "default": "Scopri di più" },
    { "type": "color",        "id": "cta_bg",         "label": "Colore CTA",     "default": "#E36B6E" }
  ],
  "blocks": [
    { "type": "@app" },
    {
      "type": "faq_item",
      "name": "Domanda FAQ",
      "settings": [
        { "type": "text", "id": "question", "label": "Domanda", "default": "..." },
        { "type": "richtext", "id": "answer", "label": "Risposta", "default": "<p>...</p>" }
      ]
    }
  ],
  "max_blocks": 12,
  "presets": [
    { "name": "<NOME LEGGIBILE SEZIONE>" }
  ]
}
{% endschema %}
```

#### D) Aggiornamento template JSON

Per ogni sezione creata, aggiungi entry al template `templates/<file>.json`:

```json
"<NN>": {
  "type": "<prefix>-<NN>-<role>",
  "settings": {}
}
```

E aggiungi la chiave all'array `order` nella posizione corretta.

#### 6.x.5.c — Push literal

##### Modalità A (one-shot)

Scrivi tutti i file `.liquid` in parallelo (multiple `Write` in un singolo messaggio). Aggiorna il template JSON con tutte le sezioni in `sections{}` + `order[]`. Push selettivo unico:

```bash
cd "<store.workdir_path>"
set -a; source "<store.env_path>"; set +a
npx @shopify/cli@latest theme push \
  --theme <store.theme_id> --nodelete --allow-live \
  --only "templates/<file>.json" \
  --only "sections/<prefix>-01-<role>.liquid" \
  --only "sections/<prefix>-02-<role>.liquid" \
  # ... tutte le sezioni
```

##### Modalità B (sezione per sezione)

Loop: per ogni sezione, write file → aggiorna template JSON → push selettivo (solo questa sezione + template) → utente verifica live → conferma → prossima sezione.

A ogni conferma di sezione, prima di passare alla successiva, `AskUserQuestion`: "Confermata. Continua sezione-per-sezione, o passo al one-shot per le restanti N?"

#### 6.x.5.d — Verifica visuale 1:1 col competitor (literal phase)

Chiedi all'utente di aprire la pagina live (in modalità "draft / unpublished theme preview" se possibile) e fare confronto **side-by-side con la pagina competitor**:

```
Apri side-by-side:
  - Sinistra:  https://<competitor.url>
  - Destra:    https://<store.shopify_domain>/<live-url>?preview_theme_id=<store.theme_id>

Verifica per ogni sezione:
  [ ] Stesso ordine sezioni (top-to-bottom)
  [ ] Stessi testi (in lingua originale, parola per parola)
  [ ] Stessi colori e tipografia
  [ ] Stesso layout mobile (375px) e desktop (1440px)
  [ ] Stessi CTA labels e link
  [ ] Niente sezioni in più o in meno
```

`AskUserQuestion`:
- **Pagina identica al competitor** → procedi a 6.x.6 (Step 2 — Localizzazione IT)
- **Ho trovato discrepanze** → l'utente le elenca, applichi fix puntuali sezione-per-sezione, ripushi, ripeti la verifica
- **Voglio rivedere l'estrazione** → torna a 6.x.5.a per la sezione interessata

⚠️ **Non procedere alla localizzazione IT finché la pagina literal non è verificata identica al competitor.** Se traduci in IT prima della verifica, eventuali discrepanze testuali si moltiplicano per due (lingua originale + traduzione → entrambe sbagliate).

---

### 6.x.6 — STEP 2: Localizzazione IT

**Solo dopo che la pagina literal è confermata 1:1 con il competitor in 6.x.5.d.**

**Obiettivo dello Step 2**: tradurre i testi delle sezioni dall'originale (EN/altro) a italiano, mantenendo:
- **Struttura identica**: stesse sezioni, stesso ordine, stessi `settings`/`blocks`
- **Layout identico**: nessuna modifica al markup HTML/CSS
- **Tono di voce**: come definito in `brand.tone` di Fase 4

Niente nuovi contenuti. Niente aggiunte. Solo traduzione dei testi che sono GIÀ nei `default` dello schema.

#### 6.x.6.a — Traduzione default schema

Per ogni sezione, leggi il file `.liquid` esistente, identifica nel `{% schema %}` tutti i `default` testuali (settings tipo `text`/`textarea`/`richtext` + `block.settings.<id>` con `default`), e proponi la traduzione IT.

**Regole**:
- Modifica SOLO i `default` dei `settings` e `blocks` del `{% schema %}`. NON toccare il markup, le classi CSS, gli `<script>`, `<style>`, attributi `data-*`.
- Mantieni stessa lunghezza/peso del testo originale (importante per layout — un titolo molto più lungo può rompere il design).
- Applica le regole di `references/localization-it.md`:
  - Mantieni tono di voce competitor (`brand.tone` di Fase 4)
  - Modera claim cosmetici/integratori secondo normativa IT
  - Adatta valute/unità di misura/formati indirizzo se compaiono
  - I testimonial: NON tradurre, **rimuovi** (sono di clienti del competitor, non nostri). Sostituisci con placeholder o con testimonial nostri se l'utente li ha
  - Numeri specifici (es. "47% reduction in 28 days"): conferma con utente che li abbiamo davvero, altrimenti modera o rimuovi

**Mostra all'utente diff sezione-per-sezione** in formato compatto:

```
=== Sezione 02 — hero ===
heading
  EN: "The cream that finally works"
  IT: "La crema che finalmente funziona"

subheading
  EN: "Visible results in 28 days, 100% money-back guarantee"
  IT: "Risultati visibili in 28 giorni, garanzia soddisfatto o rimborsato"

cta_label
  EN: "Get yours now"
  IT: "Ordina la tua"
```

`AskUserQuestion` di switch:
- **Approvo tutto, applica e pusha** → batch translation + push selettivo
- **Sezione per sezione, voglio rivedere** → loop con conferma per ogni sezione
- **Cambio tono di voce** → l'utente specifica più diretto/pacato/scientifico, rigenera

#### 6.x.6.b — Push translation

Modalità batch:
```bash
cd "<store.workdir_path>"
set -a; source "<store.env_path>"; set +a
npx @shopify/cli@latest theme push \
  --theme <store.theme_id> --nodelete --allow-live \
  --only "sections/<prefix>-01-<role>.liquid" \
  --only "sections/<prefix>-02-<role>.liquid" \
  # ... tutte le sezioni tradotte
```

Niente push del template `.json` (lo schema dei file è cambiato dentro i `.liquid`, ma la struttura del template JSON resta identica).

#### 6.x.6.c — Verifica final IT

Chiedi all'utente di aprire la pagina live e fare uno smoke check:

```
Verifica IT (URL: <live-url>):
  - [ ] Tutti i testi sono in italiano (no residui inglesi tranne brand name se voluto)
  - [ ] Layout intatto (testi più lunghi/corti non rompono il design)
  - [ ] Mobile 375px e desktop 1440px coerenti con la versione literal
  - [ ] Tono di voce coerente sezione per sezione
  - [ ] Nessun claim cosmetico/integratore problematico per normativa IT
```

`AskUserQuestion`:
- **Pagina IT OK** → procedi a 6.x.7 (caso speciale per tipo pagina)
- **Aggiusto questi testi** → fix puntuali e ripush

### 6.x.7 — Caso speciale per ogni tipo di pagina

#### Se la pagina è una PDP

Dopo il push del template + sezioni:
```
=== Crea il prodotto in Shopify ===

1. Admin → Products → Add product.
2. Title: scegli il nome (può essere quello del competitor tradotto IT, o nuovo).
3. Scrivi descrizione base (anche placeholder, useremo il template).
4. Aggiungi varianti, prezzi, immagini prodotto come desideri.
5. Sidebar destra → Online store → Theme template → seleziona `<store.slug>-pdp`.
6. Save.
7. Tornami con:
   - Slug del prodotto (es. `crema-corpo-autoabbronzante`)
   - URL live (`https://<store.shopify_domain>/products/<slug>`)
```

Aspetta che l'utente confermi e mandi l'URL. Salva in:
- `pages.pdp.product_slug`
- `pages.pdp.cta_url` (URL pubblico del prodotto live, useralo come default in tutti i CTA dei funnel/home costruiti dopo)

#### Se la pagina è un Funnel

Dopo il push del template + sezioni:
```
=== Crea la Page in Shopify Admin ===

1. Admin → Online Store → Pages → Add page.
2. Title: <titolo visibile della pagina, tradotto IT>.
3. Content: lascialo vuoto (le sezioni vengono dal template).
4. Sidebar destra → Theme template → seleziona `<store.slug>-<type>`.
5. Save.
6. Tornami con lo slug della page (es. `scopri-perche`).
```

Aspetta conferma e slug. Salva in `pages.funnel.page_slug`.

⚠️ Se in Fase 3 era stata richiesta anche la PDP, il funnel l'ha già nel `default` dei `cta_url` perché è stato costruito DOPO la PDP. Verifica i CTA: aprendo il funnel live tutte le CTA devono già puntare alla PDP creata.

#### Se la pagina è la Home

Dopo il push del template + sezioni:
```
=== Assegna come homepage ===

Hai 2 opzioni:

OPZIONE A (Recommended) — assegna come homepage:
  1. Admin → Online Store → Themes → <theme name> → Customize.
  2. Top-left picker → Home page.
  3. Sidebar destra → Theme template → seleziona `<store.slug>-home`.
  4. Save.

OPZIONE B — sostituisci direttamente l'index.json di default:
  Te lo posso fare io con un push che sovrascrive `templates/index.json`.
  Più semplice ma non reversibile senza altro push.
  Conferma se vuoi questa strada.
```

`AskUserQuestion` per scegliere. Se OPZIONE B → push selettivo con override di `templates/index.json` previo backup locale.

#### Se la pagina è un Extra (contattaci, traccia ordine, FAQ, about, blog, collection)

Stessa procedura del Funnel (Page in Admin → assegna template). Vedi `references/page-types-extras.md` per template-specifici (es. `traccia-il-tuo-ordine` ha campo input order tracking).

### 6.x.8 — Loop alla pagina successiva

Quando l'utente conferma OK sia in 6.x.5.d (literal) sia in 6.x.6.c (IT), e ha completato il caso speciale di 6.x.7 (creazione product/page/assignment): marca `current_page.status = "done"`, passa alla prossima in `scope.pages_ordered`.

Quando tutte le pagine in `scope.pages_ordered` sono done → vai a Fase 7.

---

## Fase 7 — Pagine extra opzionali

`AskUserQuestion` multi-select: "Vuoi aggiungere altre pagine prima di chiudere?"

Opzioni (vedi `references/page-types-extras.md` per template specifici):
- **Contattaci** (`contact`) — modulo + email + telefono + indirizzo
- **Traccia il tuo ordine** (`traccia-il-tuo-ordine`) — input numero ordine + email
- **FAQ** — accordion editabile da blocks
- **Chi siamo** (`about`) — storia brand
- **Blog landing** — lista articoli
- **Collection page** — griglia prodotti

Per ogni pagina extra selezionata, ripete il sub-workflow 6.x.1 → 6.x.8 (con peso ridotto sull'analisi visiva — le extras sono più standardizzate). Vale sempre la regola: prima literal clone (lingua originale), poi traduzione IT.

Quando l'utente non vuole aggiungere altro → vai a Fase 8.

---

## Fase 8 — Verifica finale + handoff

Mostra checklist finale:

```
=== Riepilogo clone competitor ===

Pagine create:
  ✓ Funnel <type>:    https://<shop>/pages/<funnel.slug>
  ✓ PDP:              https://<shop>/products/<pdp.slug>
  ✓ Home:             assegnata come homepage (template <store.slug>-home)
  ✓ Contattaci:       https://<shop>/pages/contact
  ...

Brand applicato:
  - Logo:             <store.logo_url>
  - Email:            <store.support_email>
  - Telefono:         <store.phone>
  - Indirizzo:        <store.legal_address>

Cose da verificare manualmente:
  [ ] Tutte le pagine renderizzano mobile + desktop
  [ ] Tutti i CTA dei funnel puntano alla PDP creata
  [ ] Home → click prodotto → arriva alla PDP
  [ ] Theme editor: ogni sezione mostra settings editabili
  [ ] PDP: click "+ Add block" mostra app blocks (Katching Bundles, ecc.)
  [ ] Footer corretto su tutte le pagine
```

Suggerisci passi successivi (fuori scope skill):
- Configurare policy (Privacy, Terms, Refund, Shipping) — vedi memoria di sessione, le abbiamo già scritte per altri store
- Installare app: Klaviyo, GTM, Meta Pixel, Triple Whale
- Configurare spedizioni / pagamenti / tax dello store
- Pushare il dominio live, configurare DNS

`AskUserQuestion` finale: "Tutto OK? Posso chiudere la sessione | Ho un fix da fare prima."

Se chiudi: dichiara il clone pronto.

---

## Troubleshooting rapido

| Sintomo                                         | Causa                                              | Fix                                                                  |
|-------------------------------------------------|----------------------------------------------------|----------------------------------------------------------------------|
| `theme list` ritorna 401                        | Token Theme Access scaduto                         | Rigenera `shptka_*` da app Theme Access, aggiorna `.env`             |
| Push fallisce con "Liquid syntax error"         | Schema malformato                                  | Leggi error line, `Edit` puntuale del file, ri-push                  |
| WebFetch del competitor restituisce vuoto       | JS-rendered content / paywall                      | Chiedi all'utente screenshot full-page + URL, lavora dagli screenshot |
| Brand discovery estrae colori sbagliati         | CSS dinamico / dark mode default                   | Mostra colori estratti, lascia override utente in 4.3                 |
| PDP non mostra app blocks editabili             | Schema sezione manca `{"type": "@app"}` nei blocks | Edit schema, riaggiungi `@app`, ripusha sezione                      |
| Home non si vede sul dominio                    | Template non assegnato come homepage in Theme settings | Admin → Online Store → Themes → Customize → assegna template       |
| CTA del funnel puntano a `#`                    | PDP creata DOPO il funnel (ordine sbagliato)        | Edit setting `cta_url` di tutte le sezioni funnel, sostituisci URL PDP |
| Mobile text overflow                            | Font-size fissi in px                              | Sostituisci con `clamp(1rem, 4vw, 1.5rem)` o media query             |

---

## File correlati (references)

### Specifici di questa skill (in `skills/clone-competitor-store/references/`)

- `competitor-discovery.md` — come WebFetch + estrazione palette/font/logo dal markup competitor
- `visual-replication.md` — regole per ricreare design 1:1 da screenshot + URL (mobile-first, breakpoint, classi scoped)
- `editability-and-app-blocks.md` — hard rule editabilità schema, blocks pattern, `@app` block per PDP (Katching Bundles)
- `localization-it.md` — principi traduzione ENG → IT, tono di voce, cosa fare con claim regulatory IT
- `home-template.md` — gestione `index.<slug>.json` + assignment manuale come homepage
- `page-types-extras.md` — template per contattaci, traccia ordine, FAQ, about, blog, collection

### Riusati dalle skill esistenti (path relativo)

- `../create-new-pdp/references/auth-pattern.md` — `.env`, Theme Access token, verifica connessione
- `../create-new-pdp/references/section-naming.md` — convenzioni prefissi e naming sezioni
- `../create-new-pdp/references/selective-push.md` — comando push selettivo standard
- `../create-new-pdp/references/section-schema-patterns.md` — pattern schema editabile
- `../create-new-funnel/references/brand-identity-discovery.md` — base per `competitor-discovery.md` (estesa)
- `../create-new-funnel/references/funnel-types.md` — strutture base advertorial/listicle/quiz
- `../create-new-funnel/references/page-template-layout.md` — chromeless vs theme layout
- `../create-new-funnel/references/funnel-image-specs.md` — specs immagini per ruolo sezione funnel
