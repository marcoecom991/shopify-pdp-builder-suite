---
name: create-new-pdp
description: Crea una nuova PDP Shopify da zero. Guida l'utente attraverso 7 fasi — scelta store, verifica auth, duplicazione template e sezioni con nuovo prefisso, raccolta materiali di ricerca sul prodotto, popolamento testi sezione-per-sezione mantenendo i layout esistenti, indicazioni sulle immagini da caricare, verifica finale. Da usare ogni volta che un membro del team deve pubblicare una PDP per un nuovo prodotto su uno degli store configurati (Nimea, Glowria, o altri aggiunti a config/stores.json).
---

# /create-new-pdp — Orchestrator

Guida passo-passo per creare una PDP Shopify da zero. Segui le 7 fasi **in sequenza**, mai saltare. `AskUserQuestion` come gate fra una fase e la successiva.

## Convenzioni (valide per TUTTE le fasi)

### Phase tag (priorità massima)

Ogni fase deve aprirsi con una riga isolata `<wsa-phase id="<id>" />` come **PRIMISSIMA cosa** della tua risposta — niente commenti, niente code-fence, niente prosa prima. Senza il tag, la roadmap nella sidebar Working Suite resta indietro e l'operatore non sa che sei avanzato.

| Fase                                        | Tag                                       |
| ------------------------------------------- | ----------------------------------------- |
| 1 — Scelta store                            | `<wsa-phase id="select-store" />`         |
| 2 — Auth + tema                             | `<wsa-phase id="auth-check" />`           |
| 3 — Duplicazione template                   | `<wsa-phase id="duplicate-template" />`   |
| 4 — Raccolta materiali                      | `<wsa-phase id="research-materials" />`   |
| 5 — Riscrittura testi                       | `<wsa-phase id="rewrite-content" />`      |
| 6 — Guida immagini                          | `<wsa-phase id="images-guide" />`         |
| 7 — Verifica finale                         | `<wsa-phase id="final-check" />`          |
| 8 — Link su Shopify (solo sotto WSA)        | `<wsa-phase id="link-shopify" />`         |

### AskUserQuestion

Per domande con **opzioni discrete** (Sì/No, scelte di lista, conferme): usa SEMPRE il tool `AskUserQuestion`. Working Suite lo renderizza come pannello cliccabile sotto la chat.

**Non duplicare la domanda nel testo prima del tool** — l'utente la vede già nel pannello. Limita la prosa a 1-2 righe di contesto reale se serve.

Per "una sola opzione disponibile e serve conferma": `AskUserQuestion` con `Procedi con <nome>` + `Indietro`. Per "una sola opzione e niente da confermare": vai avanti senza chiedere.

Domande aperte (token, URL, slug, testi liberi): chiedi in testo — Working Suite mostra l'input testuale in basso.

### Push selettivo (template di comando — riusalo ovunque)

```bash
cd "$STORE_WORKDIR"
set -a; source "$STORE_ENV"; set +a
npx @shopify/cli@latest theme push \
  --theme "$STORE_THEME_ID" --nodelete --allow-live \
  --only "<file1>" --only "<file2>" ...
```

Mai `theme push` senza `--only`. Il `--only` include SOLO i file del nuovo prefisso (template + sezioni duplicate).

### Regole di intoccabilità

1. **Mai modificare il template base o le sue sezioni.** Il template base (es. `product.berberina-pills.json`) e le sue sezioni (es. `cboe-pdp-05.liquid`) appartengono a un prodotto live diverso. Si duplicano in file nuovi con prefisso nuovo; ogni `Edit`/`Write` avviene sui duplicati.
2. **Mai riscrivere il markup esistente** durante la riscrittura testi. Layout, CSS, JS si preservano; modifichi solo i `default` dei `{% schema %}`. Fonte: `references/workflow-faithful-rebuild.md`.
3. **Ogni sezione duplicata deve essere editabile dal theme editor.** Se è hardcoded (markup legacy), liquidify automaticamente — vedi Fase 3.5. Pattern: `references/section-schema-patterns.md`.
4. **Mai scrivere dentro `~/.claude/`.** Il cwd corrente è il plugin skin: tutto quello che modifichi vive lì.
5. **Conferma con l'utente prima di ogni azione irreversibile** (creazione file, push live, rinomina).

## Setup ambient (Working Suite)

Sotto WSA — riconoscibile da `$WSA_INTERNAL_KEY` valorizzato — il wrapper ti consegna l'ambiente pronto:

- **cwd**: `~/Desktop/shopify-pdp-builder/` (plugin skin per-workspace).
- **`config/stores.json`**: SEMPRE in `./config/stores.json` (relativo al cwd). Read diretto, niente fallback.
- **`.env` di ogni store**: già scritto dal wrapper con il Theme Access token salvato dall'operatore in **Configurazioni → Store → Credenziali**. **Non chiedere mai il token in chat.**
- **Stores mancanti**: lo `stores.json` è in **sola lettura** sotto WSA (rigenerato a ogni init). Se serve uno store non in lista, rimanda l'operatore a Configurazioni → Store nella dashboard, poi ricaricare la chat.

In modalità manuale (claude lanciato direttamente, no `$WSA_INTERNAL_KEY`): puoi chiedere il token in chat e scrivere `.env` inline. Pattern + sezione "Come generare un nuovo Theme Access token" in `references/auth-pattern.md`.

---

## Fase 1 — Scelta store

🏷️ Prima riga: `<wsa-phase id="select-store" />`

`Read config/stores.json` (path relativo al cwd). Mostra la lista all'utente via `AskUserQuestion` — una opzione per store, niente "Altro" sotto WSA.

Salva i campi dello store scelto in memoria: `store.name`, `store.shopify_domain`, `store.theme_id`, `store.workdir_path`, `store.env_path`.

---

## Fase 2 — Verifica auth + scelta tema

🏷️ Prima riga: `<wsa-phase id="auth-check" />`

1. Verifica che `store.env_path` esista e contenga `SHOPIFY_CLI_THEME_TOKEN=` non vuoto. Se vuoto sotto WSA: di' all'utente di aggiungere il token in Configurazioni → Store e ricaricare. Stop.
2. Elenca i temi:
   ```bash
   cd "<store.workdir_path>"
   set -a; source "<store.env_path>"; set +a
   npx @shopify/cli@latest theme list --no-color
   ```
   Se l'output mostra `401` / `Invalid API key`: token scaduto. Rimanda l'utente a rigenerarlo.
3. Mostra i temi parsati con il flag `[live]` evidenziato:
   ```
   • [live] <Nome>  (id: <ID>)
   • [unpublished] <Nome>  (id: <ID>)
   ```
4. `AskUserQuestion`: tema su cui operare? Default proposto: `[live]` (o `store.theme_id` da config). Una opzione per tema.
5. Salva la scelta in `store.theme_id` + `store.theme_name`.
6. **Pull fresco del tema scelto** (chiedi conferma prima — il pull sovrascrive file locali non pushati):
   ```bash
   cd "<store.workdir_path>"
   set -a; source "<store.env_path>"; set +a
   npx @shopify/cli@latest theme pull --theme <store.theme_id> --nodelete
   ```

---

## Fase 3 — Duplicazione template

🏷️ Prima riga: `<wsa-phase id="duplicate-template" />`

### 3.1 Nome nuovo template

`AskUserQuestion`: "Che nome vuoi dare al nuovo template PDP?"

Vincoli: kebab-case, solo `[a-z0-9-]`, esempi `crema-borse-occhiaie-pdp`, `siero-vitamina-c-pdp`. Verifica che `templates/product.<nome>.json` non esista già.

### 3.2 Template base

```bash
ls "<store.workdir_path>/templates/" | grep '^product\..*\.json$'
```

`AskUserQuestion` con una opzione per template trovato. Se ne esiste uno solo oltre al default `product.json`, assumi quello senza chiedere.

### 3.3 Prefisso sezioni

Genera default dalle iniziali (vedi `references/section-naming.md`):
- `crema-borse-occhiaie-pdp` → `cboe`
- `siero-vitamina-c-pdp` → `svc`

`AskUserQuestion`: prefisso `<default>`? Conferma o alternativo.

### 3.4 Duplicazione

1. Leggi il template base JSON.
2. Estrai le sezioni referenziate (`"type": "<nome>"` che corrisponde a un file `sections/<nome>.liquid`).
3. Identifica il prefisso del base (inizio comune dei nomi sezione).
4. Per ogni sezione col prefisso base:
   - `Read sections/<old>-<suffix>.liquid` → `Write sections/<nuovo>-<suffix>.liquid`.
   - Aggiorna il nome schema (`"name"` in `{% schema %}` e `presets[0].name`) con uppercase del nuovo prefisso. Le classi CSS interne (`.cboe-...`) restano invariate.
5. Scrivi `templates/product.<nome>.json` come copia del base, sostituendo i `"type"` col nuovo prefisso.
6. Riepilogo all'utente + conferma prima del push.

**Mai modificare i file originali.**

### 3.5 Liquidify automatico (OBBLIGATORIO, prima del push)

Ogni sezione duplicata deve essere editabile dal theme editor. Pattern: `references/section-schema-patterns.md`.

Per ogni file duplicato:

1. **Detection** (deterministica, no domande):
   - Leggi schema → settings/blocks non vuoti?
   - Testi visibili nel markup referenziati come `{{ section.settings.<id> }}` / `{{ block.settings.<id> }}`?
   - Immagini via `| image_url`, no `<img src="https://cdn.shopify.com/...">` hardcoded?
   - Se sì a tutto → sezione già editabile, lascia invariata.
   - Altrimenti → marca `needs_liquidify`.

2. **Se needs_liquidify**: liquidifica la copia (mai l'originale):
   - Estrai testi visibili, URL immagine (`<img src>`, `srcset`, `background-image: url(...)` inline), link, `alt`.
   - Per ciascuno: setting con `id` semantico (`hero_heading`, `hero_image`, `cta_primary_url`...) e `default` = valore estratto.
   - Tipi: `richtext` per paragrafi formattati, `image_picker` per immagini, `url`+`text` per CTA, `blocks` per gruppi ripetuti.
   - Sostituisci nel markup: testi → `{{ section.settings.<id> }}`, immagini → `{{ section.settings.<id> | image_url: width: <W> }}`.
   - NON modificare: tag HTML, classi CSS, `data-*`, `<script>`, `<style>`, JSON-LD, Liquid logic.

3. Riepilogo all'utente: quali sono già editabili, quali liquidificate.

### 3.6 Push iniziale

Push selettivo con `--only` per template + tutte le sezioni duplicate (vedi convenzioni).

### 3.7 Creazione Product in Shopify Admin

Istruzioni all'utente:
```
1. Admin → Products → Add product
2. Title: "<nome del nuovo prodotto>"
3. Theme template (sidebar destra): seleziona <nome>
4. Save
5. Torna qui col lo slug del prodotto (es. crema-borse-occhiaie-ufficiale)
```

Salva lo slug in `product.slug`. URL live: `https://<store.shopify_domain>/products/<product.slug>`.

---

## Fase 4 — Raccolta materiali

🏷️ Prima riga: `<wsa-phase id="research-materials" />`

Chiedi all'utente (checklist):

1. **PDP competitor** (1-3 URL/screenshot) — linguaggio, claim, struttura.
2. **Transcript video** (1-3) — tono di voce, punti di dolore, frasi ad alta conversione.
3. **Research prodotto**: ingredienti chiave, claim principali, target audience, differenziazione, proof points.
4. **Angolo marketing attuale** (1-3 esempi di ad: copy + headline + immagine/video).
5. **Brand assets**: palette HEX, font, logo URL CDN.

### 4.1 Angle analysis (OBBLIGATORIO, non saltabile)

Dopo aver ricevuto i materiali, prima della riscrittura:

1. **Identifica il MAIN ANGLE** dei competitor. L'angle è il frame narrativo (NON il claim). Esempi:
   - Ingrediente scientifico esclusivo
   - Routine naturale di 5 minuti
   - Alternativa economica al trattamento da €300
   - Before/after reale di persone come te
   - Risolve X in N giorni — garantito

2. **Presenta il main angle** con evidenze:
   ```
   ➤ ANGLE: <una riga>
      Evidenze:
      - Competitor A (URL): <come lo usa>
      - Competitor B (URL): <come lo usa>
      - Ads attuali: <come lo rinforzano>
   ```

3. `AskUserQuestion`: "Usi questo angle, proponimi alternative, o hai un angle in mente?"

4. Se alternative: proponi 2-3 angle (a) usati in adiacenza, (b) sostenibili dai nostri proof points, (c) differenzianti rispetto al main. `AskUserQuestion` finale.

Salva in `pdp.angle` — bussola della Fase 5.

### 4.2 Riepilogo + conferma

Bullet sintetico: nome prodotto, angle, 3 claim, 3 proof, palette. `AskUserQuestion`: "Materiali completi, procedo con la scrittura testi?"

---

## Fase 5 — Riscrittura testi

🏷️ Prima riga: `<wsa-phase id="rewrite-content" />`

**Le sostituzioni avvengono nei `default` dei `{% schema %}`** (settings/blocks), non nel markup HTML. Markup/CSS/JS della sezione restano identici. Tutti i testi coerenti con `pdp.angle`.

### 5.0 Modalità (obbligatorio)

`AskUserQuestion`:
- **A — Sezione per sezione** (consigliata per prima PDP): riscrivi, mostra diff, push solo quella sezione, l'utente verifica live, passa alla successiva.
- **B — Batch**: tutte parallele, diff completo unico, approvazione globale, push selettivo unico.
- **C — Misto**: cambia modalità in corsa quando vuoi.

### 5A — Sotto-flusso sezione per sezione

Per ogni sezione del template nuovo, in ordine di apparizione:

1. **Read** del file duplicato → identifica ruolo (hero, FAQ, testimonial, ...) usando `references/image-specs-per-section.md`.
2. **Proponi testi nuovi** coerenti con `pdp.angle`, stesso numero di bullet/card/FAQ. Diff before/after chiaro:
   ```
   — Headline
      prima: "..."
      dopo:  "..."
   ```
   Se la research non basta per una struttura (es. 5 card, 3 claim): chiedi input mirato invece di inventare.
3. **Aspetta conferma/correzioni** prima di scrivere.
4. **Applica** modifiche SOLO ai `default` dello schema:
   - Poche sostituzioni → `Edit` con `old_string`/`new_string` esatti.
   - Molte → `Bash` con `python3 <<'PY' ... PY` heredoc (parsa schema JSON, modifica `default`, riscrivi blocco).
   - Non toccare: markup HTML, classi CSS, `<script>`, `<style>`, `data-*`, URL immagini in `image_picker` default (verranno sostituite in Fase 6), Liquid tags.
   - Se trovi testi ancora hardcoded nel markup: segnala e completa la liquidify su quella sezione (non patchare nel markup).
5. **Verifica no-regressioni**: prima del push, conferma di non aver toccato file `<vecchio-prefisso>-*` o `templates/product.<vecchio-nome>.json`.
6. **Push selettivo** della sola sezione modificata.
7. Utente verifica `https://<store.shopify_domain>/products/<product.slug>` mobile + desktop.
8. Aggiustamenti minimi (font-size, white-space, spelling) → incrementali, mai rifare la sezione.
9. Prima della prossima sezione: `AskUserQuestion` "Continua sezione-per-sezione, passa a batch, o misto?" Rispetta la scelta.

### 5B — Sotto-flusso batch

1. **Read tutte** le sezioni duplicate in parallelo (multiple Read in un messaggio).
2. **Pianifica riscrittura integrata** — testi tra sezioni coerenti, stesso linguaggio, claim ripetuti con stessa terminologia.
3. **Diff completo unico** raggruppato per sezione.
4. Approvazione globale o correzioni puntuali → applica.
5. Verifica no-regressioni → push selettivo unico con tutte le sezioni.
6. Utente check completo. Aggiustamenti successivi → puntuali (anche se il primo push era batch).

### 5C — Misto

L'utente cambia modalità in corsa. Rispetta la scelta, continua dalla prossima sezione non ancora lavorata.

---

## Fase 6 — Guida immagini

🏷️ Prima riga: `<wsa-phase id="images-guide" />`

Le immagini sono `image_picker` nello schema. L'utente le carica **dal theme editor**, nessun replace di URL CDN nei file. Specs: `references/image-specs-per-section.md`.

Per ogni sezione con `image_picker`:

1. `grep image_picker` nello schema → conta + ruoli.
2. Brief all'utente:
   ```
   Sezione: <nuovo-prefisso>-<suffix>.liquid
   - Ruolo immagine: <cosa deve mostrare>
   - Ratio: <es. 4:5>, dimensioni: <1200×1500>, peso max: <150 KB>
   - Campo theme editor: "<label image_picker>"
   ```
3. Istruzioni:
   ```
   1. Admin → Online Store → Themes → Customize (<store.theme_name>)
   2. Top-left picker → Products → <prodotto>
   3. Seleziona la sezione → click sul campo immagine → Upload → Save
   ```
4. Conferma visiva su URL live. Nessun push necessario.

---

## Fase 7 — Verifica finale

🏷️ Prima riga: `<wsa-phase id="final-check" />`

Checklist:
- [ ] Testi validati su tutte le sezioni
- [ ] Tutte le immagini caricate e renderizzate
- [ ] Mobile 375px ok
- [ ] Desktop 1440px ok
- [ ] DevTools console pulita (no errori Liquid, no 404 immagini)
- [ ] URL diretto ok: `https://<store.shopify_domain>/products/<product.slug>`
- [ ] Custom domain (se presente)

Per ogni problema → torna alla fase corrispondente. Quando tutto ✅: dichiara la PDP pronta. Suggerisci catalogo/collezione, pricing/inventory/variants, bundling (Katching: vedi `references/workflow-faithful-rebuild.md`).

---

## Fase 8 — Link su Shopify (solo sotto WSA)

🏷️ Prima riga: `<wsa-phase id="link-shopify" />`

Se `$WSA_INTERNAL_KEY` vuoto: salta la fase.

```bash
test -n "$WSA_INTERNAL_KEY" && echo "WSA_ACTIVE" || echo "WSA_INACTIVE"
```

Se attiva: chiedi l'URL PDP live (testo libero o `AskUserQuestion`), salva in `$PRODUCT_URL`, poi:

```bash
curl -sS -X POST "$WSA_INTERNAL_BASE/link-shopify" \
  -H "Content-Type: application/json" \
  -H "X-WSA-Key: $WSA_INTERNAL_KEY" \
  -d "$(printf '{"url":"%s"}' "$PRODUCT_URL")"
```

Risposta:
- `ok: true` + `productLinked: true` → sessione collegata al prodotto in `workspace_products`. Conferma all'utente, chiudi.
- `ok: true` + `productLinked: false` → URL salvato ma handle non corrisponde. Suggerisci: "Vai su Configurazioni → Prodotti e aggiungi `<handle>` per collegare anche le analytics, oppure proseguiamo senza."
- `ok: false` → mostra errore + suggerisci link manuale dalla pagina sessione Builder.

---

## References

- `workflow-faithful-rebuild.md` — regola d'oro + tecniche sostituzione + form Katching
- `auth-pattern.md` — Theme Access token, modalità manuale
- `selective-push.md` — comando push completo
- `section-naming.md` — convenzioni prefissi
- `image-specs-per-section.md` — dimensioni/ratio per ruolo sezione
- `section-schema-patterns.md` — liquidify, setting types, edge case

## Troubleshooting

| Sintomo                          | Causa                                  | Fix                                                  |
| -------------------------------- | -------------------------------------- | ---------------------------------------------------- |
| `theme list` ritorna 401         | Token scaduto/revocato                 | Operatore rigenera in Configurazioni → Store         |
| Push fallisce "Liquid syntax"    | Sostituzione ha rotto Liquid           | Read file, localizza errore, Edit per riparare       |
| Sezione non appare su PDP live   | Template JSON o sezione non pushata    | Verifica `templates/product.<nome>.json` + push      |
| Stile diverso dall'originale     | Hash legacy toccato per errore         | Ripristina file originale + rifai solo testi schema  |
| Product non usa il nuovo template| Template suffix non selezionato        | Admin → Product → Theme template → seleziona         |
