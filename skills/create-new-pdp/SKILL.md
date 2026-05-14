---
name: create-new-pdp
description: Crea una nuova PDP Shopify da zero. Guida l'utente attraverso 7 fasi — scelta store, verifica auth, duplicazione template e sezioni con nuovo prefisso, raccolta materiali di ricerca sul prodotto, popolamento testi sezione-per-sezione mantenendo i layout esistenti, indicazioni sulle immagini da caricare, verifica finale. Da usare ogni volta che un membro del team deve pubblicare una PDP per un nuovo prodotto su uno degli store configurati (Nimea, Glowria, o altri aggiunti a config/stores.json).
---

# /create-new-pdp — Orchestrator

Sei una guida passo-passo per creare una PDP Shopify da zero. Segui le 7 fasi **in sequenza**. Non saltare fasi. Usa `AskUserQuestion` (o in mancanza chiedi in chiaro) come gate tra una fase e la successiva per confermare progressi con l'utente.

## Phase signaling (Working Suite)

Working Suite mostra una mini-roadmap nella sidebar del Builder che si aggiorna in base ai segnali che mandi qui. **Appena entri in una fase, scrivi UNA RIGA isolata** con il tag della fase, poi vai avanti come al solito. Niente commenti, niente codice fence, niente prefisso: la linea deve essere esattamente quella e nient'altro.

| Fase | Tag da scrivere |
|------|-----------------|
| Fase 1 — Scelta store | `<wsa-phase id="select-store" />` |
| Fase 2 — Auth + tema | `<wsa-phase id="auth-check" />` |
| Fase 3 — Duplicazione template | `<wsa-phase id="duplicate-template" />` |
| Fase 4 — Raccolta materiali | `<wsa-phase id="research-materials" />` |
| Fase 5 — Riscrittura testi | `<wsa-phase id="rewrite-content" />` |
| Fase 6 — Guida immagini | `<wsa-phase id="images-guide" />` |
| Fase 7 — Verifica finale | `<wsa-phase id="final-check" />` |
| Fase 8 — Link su Shopify (post-creazione) | `<wsa-phase id="link-shopify" />` |

Esempio inizio Fase 1:
```
<wsa-phase id="select-store" />

Ciao 👋 vediamo su quale store lavoriamo oggi...
```

Il tag viene processato e nascosto dall'UI; non lo vede l'utente, vede solo il testo dopo.

## Principio generale

- **🔒 Il template base e le sue sezioni NON si toccano MAI.** Il template base (es. `product.berberina-pills.json`) e le sue sezioni (es. `cboe-pdp-05.liquid`) appartengono a un prodotto live diverso. Ogni modifica a quei file rompe un prodotto esistente. **Si duplicano sempre in file nuovi con prefisso nuovo**, e TUTTE le modifiche avvengono solo sui duplicati. Mai `Edit` o `Write` sui file base.
- **Non riscrivere il markup esistente.** Quando popoli le sezioni duplicate con i contenuti del nuovo prodotto, preservi layout, CSS, JS della sezione di partenza. Modifichi solo testi e URL immagini. Fonte di verità: `references/workflow-faithful-rebuild.md`.
- **Ogni sezione duplicata deve finire editabile dal theme editor.** Durante la duplicazione (Fase 3), ogni file copiato viene ispezionato: se è già editabile (schema con `settings`/`blocks` che coprono testi e immagini) resta com'è; se è hardcoded (markup legacy, tipo GemPages detached) la skill lo **liquidifica automaticamente** — estrae testi/immagini/CTA nel markup del duplicato e li sposta nello schema come `settings`/`blocks` con `default`, sostituendo nel Liquid con `{{ section.settings.* }}`. Fonte: `references/section-schema-patterns.md`. **La liquidify tocca solo il duplicato, mai l'originale.**
- **Un push per volta, sempre selettivo.** Mai `theme push` senza `--only`. Il `--only` include SOLO i file duplicati (nuovo prefisso). Fonte di verità: `references/selective-push.md`.
- **Conferma con l'utente prima di ogni azione distruttiva o irreversibile** (creazione file, push live, rinomina).

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
PLUGIN_ROOT=$(dirname "$(dirname "$(readlink "$HOME/.claude/skills/create-new-pdp")")")
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

Mostra all'utente la lista degli store configurati. Usa `AskUserQuestion` con un'opzione per store + "Altro" (per aggiungerne uno nuovo on-the-fly).

Se l'utente sceglie "Altro":
- Chiedi nome store, `shopify_domain` (*.myshopify.com), theme ID, path workdir e path env.
- Aggiungi la voce a `stores.json` e procedi.

Salva in memoria i campi dello store scelto:
- `store.name`
- `store.shopify_domain`
- `store.theme_id`
- `store.workdir_path`
- `store.env_path`

---

## Fase 2 — Verifica auth + scelta tema

1. Controlla che `store.env_path` esista. Se NO:
   - Spiega all'utente cosa serve (token Theme Access). Fonte: `references/auth-pattern.md`, sezione "Come generare un nuovo Theme Access token".
   - Guidalo a creare il file: chiedigli di incollare il token, poi scrivi `store.env_path` con:
     ```
     SHOPIFY_CLI_THEME_TOKEN=<token-incollato>
     SHOPIFY_FLAG_STORE=<store.shopify_domain>
     ```
2. Verifica la connessione elencando i temi dello store:
   ```bash
   cd "<store.workdir_path>"
   set -a; source "<store.env_path>"; set +a
   npx @shopify/cli@latest theme list --no-color
   ```
3. Se l'output mostra errori di auth (`401`, `Invalid API key`, `Unauthorized`): il token è scaduto. Chiedi all'utente di rigenerarlo e aggiornare il `.env`. Rilancia il test.

4. **Mostra all'utente la lista temi parsata** in formato leggibile, evidenziando il live pubblicato:
   ```
   Temi attivi su <store.name>:
     • [live]     <Nome tema>  (id: <ID>)
     • [unpublished] <Nome tema>  (id: <ID>)
     • [development] <Nome tema>  (id: <ID>)
   ```

5. Usa `AskUserQuestion` per chiedere **su quale tema operare**:
   - Default proposto: il tema `[live]` (o quello in `store.theme_id` se presente in `config/stores.json`).
   - Opzioni: una per ogni tema elencato.
   - L'utente può scegliere anche un tema unpublished/development per testare in sicurezza.
6. Salva la scelta in `store.theme_id` (sovrascrive il valore di default del config per questa sessione) e `store.theme_name`.

7. **Pull fresco del tema scelto**, così le duplicazioni partono da file allineati al remoto (e il template base che modificheremmo per sbaglio non esiste solo in locale vecchio):
   ```bash
   cd "<store.workdir_path>"
   set -a; source "<store.env_path>"; set +a
   npx @shopify/cli@latest theme pull --theme <store.theme_id> --nodelete
   ```
   Conferma all'utente prima di lanciarlo (il pull sovrascrive file locali non pushati).

---

## Fase 3 — Duplicazione template

### 3.1 Nome nuovo template

Con `AskUserQuestion`: "Che nome vuoi dare al nuovo template PDP?"
- Valida: kebab-case, solo `[a-z0-9-]`, niente spazi.
- Esempi: `crema-borse-occhiaie-pdp`, `siero-vitamina-c-pdp`, `nuovo-prodotto-pdp`.
- Verifica che `<store.workdir_path>/templates/product.<nome>.json` NON esista già. Se esiste: chiedi conferma sovrascrittura o nome diverso.

### 3.2 Scelta template base

Elenca i `product.*.json` presenti in `<store.workdir_path>/templates/`:
```bash
ls "<store.workdir_path>/templates/" | grep '^product\..*\.json$'
```

Con `AskUserQuestion`: "Da quale template vuoi partire?"
- Mostra i nomi leggibili (es. `berberina-pills`, `crema-borse-occhiaie-pdp`).
- Fallback: se c'è un solo `product.*.json` oltre al default `product.json`, assumi quello.

### 3.3 Prefisso nuove sezioni

Genera un prefisso di default dalle iniziali del nome template (vedi `references/section-naming.md`):
- `crema-borse-occhiaie-pdp` → `cboe`
- `siero-vitamina-c-pdp` → `svc`

Con `AskUserQuestion`: "Prefisso sezioni: `<default>`? (conferma o proponi alternativo)"

### 3.4 Duplicazione effettiva

1. Leggi il template base JSON con `Read`.
2. Estrai la lista delle sezioni referenziate: ogni blocco `"type": "<nome-sezione>"` che corrisponde a un file `sections/<nome-sezione>.liquid`.
3. Identifica il prefisso corrente del template base (tipicamente l'inizio del nome di molte sezioni, es. `cboe-` se vengono da `cboe-pdp-05`, `cboe-pdp-06a-video`, ecc.).
4. Per ogni sezione referenziata che inizia col prefisso base:
   - Leggi il file originale `sections/<old-prefisso>-<suffix>.liquid` con `Read`.
   - Calcola nuovo path: `sections/<nuovo-prefisso>-<suffix>.liquid`.
   - Scrivi il nuovo file con `Write`, applicando queste sostituzioni:
     - In `{% schema %}`: `"name": "<OLD NAME>"` → `"name": "<NEW NAME>"` (uppercase del prefisso).
     - In `presets[0].name`: stessa cosa.
     - Nelle `class=` del markup: se c'è una classe tipo `cboe-` specifica nello scoping CSS, lasciala (è classe CSS, non identifica la sezione). Tocca SOLO il nome schema.
5. Scrivi il nuovo template JSON:
   - Copia il contenuto del base.
   - Sostituisci ogni `"type": "<old-prefisso>-<suffix>"` con `"type": "<nuovo-prefisso>-<suffix>"`.
   - Salva in `templates/product.<nome-nuovo>.json`.
6. Mostra all'utente un riepilogo:
   ```
   Creati:
   - templates/product.<nome>.json
   - sections/<nuovo-prefisso>-01.liquid (da <old>-01)
   - sections/<nuovo-prefisso>-05.liquid (da <old>-05)
   ...
   ```
7. Chiedi conferma prima del push.

### 3.5 Detection + liquidify automatico (OBBLIGATORIO, prima del push)

**Prima di pushare**, ogni file `sections/<nuovo-prefisso>-*.liquid` appena creato deve essere **editabile dal theme editor**. Fonte di verità: `references/section-schema-patterns.md`.

Per ogni sezione duplicata:

1. **Detection automatica** (nessuna domanda all'utente, comportamento deterministico):
   - Leggi il file con `Read`.
   - Estrai lo `{% schema %}` JSON.
   - Considera la sezione **già editabile** se:
     - Lo schema ha `settings` o `blocks` non vuoti, **E**
     - I testi visibili nel markup (tra tag `<h*>`, `<p>`, `<li>`, `<span>`, `<a>`, `<button>`, attributi `alt=`) sono referenziati come `{{ section.settings.<id> }}` / `{{ block.settings.<id> }}` / `{% for block in section.blocks %}…`, **E**
     - Le immagini sono referenziate via `{{ section.settings.<id> | image_url: … }}` o `{{ block.settings.<id> | image_url: … }}` (no `<img src="https://cdn.shopify.com/…">` hardcoded).
   - Altrimenti → marca `needs_liquidify = true`.

2. **Se `needs_liquidify = true`** → liquidifica la copia (mai il sorgente):
   - Estrai dal markup i testi visibili, gli URL immagine (`<img src>`, `srcset`, `background-image: url(...)` inline), i link (`<a href>` di contenuto) e gli `alt`.
   - Per ogni elemento estratto genera un `setting` con `id` semantico (snake_case: `hero_heading`, `hero_subheading`, `hero_image`, `cta_primary_url`, `cta_primary_label`, `benefit_1_title`, …) e `default` = valore estratto.
   - Usa `richtext` per paragrafi con formattazione inline (`<strong>`, `<em>`, `<a>`), `image_picker` per immagini, `url` + `text` per coppie CTA, `blocks` per gruppi ripetuti di elementi fratelli con stessa classe (FAQ, card benefici, testimonianze).
   - Sostituisci nel markup: testi → `{{ section.settings.<id> }}`, immagini → `{{ section.settings.<id> | image_url: width: <W> }}` (ricostruendo eventuale srcset via filter Shopify).
   - **NON modificare** tag HTML, classi CSS, attributi `data-*`, `<script>`, `<style>`, JSON-LD, Liquid logic esistente. Tocca solo contenuto.
   - **NON toccare MAI** il file sorgente `sections/<vecchio-prefisso>-*.liquid`.

3. **Se già editabile** → lascia il file invariato.

4. Mostra all'utente il riepilogo:
   ```
   Editabilità sezioni duplicate:
     ✓ <nuovo-prefisso>-hero.liquid       → già editabile, invariato
     ✓ <nuovo-prefisso>-benefits.liquid   → liquidified (14 settings, 1 blocks FAQ)
     ✓ <nuovo-prefisso>-reviews.liquid    → liquidified (3 blocks review)
   Template sorgente NON modificato.
   ```

Pattern e esempi before/after: `references/section-schema-patterns.md`.

### 3.6 Push iniziale

Push selettivo di TUTTI i file appena creati (template + sezioni):
```bash
cd "<store.workdir_path>"
set -a; source "<store.env_path>"; set +a
npx @shopify/cli@latest theme push \
  --theme <store.theme_id> \
  --nodelete \
  --allow-live \
  --only "templates/product.<nome>.json" \
  --only "sections/<nuovo-prefisso>-01.liquid" \
  --only "sections/<nuovo-prefisso>-05.liquid" \
  # ... tutte le sezioni duplicate
```

### 3.7 Istruzioni per il Product Shopify

Mostra all'utente istruzioni per creare il Product in Shopify Admin:
```
Ora crea il prodotto in Shopify:
1. Admin → Products → Add product.
2. Title: "<nome del nuovo prodotto>".
3. Scrivi una descrizione base (può essere placeholder, la personalizzeremo dopo via template).
4. In "Theme template" (sidebar destra, sezione Online store), seleziona: <nome> (il nostro nuovo template).
5. Save.
6. Torna qui e dimmi lo slug del prodotto (es. `crema-borse-occhiaie-ufficiale`), così posso riferirmi alla URL live per i check successivi.
```

Salva lo slug in memoria: `product.slug`. URL live: `https://<store.shopify_domain>/products/<product.slug>`.

---

## Fase 4 — Raccolta materiali

Chiedi all'utente di fornire (puoi presentare come checklist):

1. **PDP competitor**: 1-3 URL o screenshot di PDP di competitor dello stesso angolo/categoria. Servono per capire il linguaggio, i claim, la struttura.
2. **Transcript video**: 1-3 video (competitor o interni) trascritti in testo. Servono per capire tono di voce, punti di dolore menzionati, frasi ad alta conversione.
3. **Research prodotto**:
   - Ingredienti chiave (nome + cosa fa + dosaggio se rilevante).
   - Claim principali (effetto #1, #2, #3 del prodotto).
   - Target audience (chi compra, cosa risolve).
   - Differenziazione (perché noi vs competitor).
   - Proof points (certificazioni, test clinici, numeri di vendita, recensioni).
4. **Angolo marketing corrente**: 1-3 esempi di ad che l'utente sta girando su questo prodotto (copy + headline + immagine/video). Serve per allineare il tono della PDP.
5. **Brand assets**: palette colori (HEX), font (famiglie web), logo URL CDN.

### 4.1 Analisi angle competitor + proposta (OBBLIGATORIO)

**Step fisso, non saltabile.** Dopo aver ricevuto le PDP competitor e il resto dei materiali, PRIMA di passare alla riscrittura testi:

1. **Identifica il MAIN ANGLE** usato dai competitor per questo prodotto. L'angle è l'angolazione narrativa principale — NON è il claim, è il frame da cui il prodotto viene raccontato. Esempi:
   - "Ingrediente scientifico esclusivo" (angle scientifico)
   - "Routine naturale di 5 minuti al giorno" (angle lifestyle/semplicità)
   - "Alternativa economica al trattamento estetico da €300" (angle risparmio)
   - "Before/after reale di persone come te" (angle social proof/identificazione)
   - "Risolve X in N giorni — garantito" (angle result-driven/urgency)

2. **Presenta l'angle dominante** all'utente con evidenze:
   ```
   Dall'analisi dei competitor PDP + ads emerge questo angle dominante:

   ➤ ANGLE: <frase di 1 riga>
      Evidenze:
      - Competitor A (URL): <come lo usa>
      - Competitor B (URL): <come lo usa>
      - Ads attuali: <come lo rinforzano>
   ```

3. **`AskUserQuestion`**: "Vuoi usare questo stesso angle per la nostra PDP?"
   - **Sì, uso questo angle** → salva in `pdp.angle` e procedi.
   - **No, proponimi alternative** → vai al punto 4.
   - **Ho un angle mio in mente** → l'utente lo descrive, salva e procedi.

4. Se l'utente chiede alternative: **proponi 2-3 angle alternativi** che sono:
   - Usati in adiacenza (da competitor minori o in prodotti simili della stessa categoria).
   - Coerenti con i proof points / ingredienti del nostro prodotto (niente angle che non possiamo sostenere).
   - Differenzianti rispetto al main angle (per non essere "l'ennesima copia di X").

   Presentali con la stessa struttura (angle + evidenze + perché potrebbe funzionare meglio). Poi richiedi conferma via `AskUserQuestion`.

Salva l'angle scelto in `pdp.angle` — sarà la bussola della riscrittura di TUTTE le sezioni in Fase 5.

### 4.2 Riepilogo e conferma

Fai un riepilogo sintetico:
- Nome prodotto
- **Angle scelto** (una riga)
- 3 claim principali
- 3 proof points
- Palette

Chiedi: "Materiali completi, procedo con la scrittura testi sezione-per-sezione?"

---

## Fase 5 — Riscrittura testi per il nuovo prodotto

**Contesto importante**: le sezioni duplicate sono a questo punto **tutte editabili** (Fase 3.5 ha liquidificato quelle legacy). I testi del prodotto base (es. berberina) vivono nei `default` dello schema di ogni sezione. Il tuo compito in questa fase è **riscrivere i testi del nuovo prodotto** partendo dalla research raccolta in Fase 4, **modificando i `default` nello schema** — NON il markup HTML. Markup/CSS/JS/struttura restano identici.

**Regola operativa**: le sostituzioni di testo avvengono dentro il blocco `{% schema %}` (campi `default` di `settings` e `blocks`), non nel markup Liquid sopra. Markup e classi CSS non si toccano mai in questa fase.

Nessun HTML da incollare in questa fase — tutti i contenuti del nuovo prodotto li hai già dal blocco di ricerca della Fase 4.

### 5.0 Scelta modalità (obbligatoria, all'inizio della fase)

Usa `AskUserQuestion` per proporre le due modalità di lavoro:

**Opzione A — Sezione per sezione (consigliata per prima PDP)**
Per ogni sezione: riscrivi, mostra diff, push solo di quella sezione, l'utente verifica live, passa alla successiva.
→ Più lenta ma più sicura: se una modifica è sbagliata te ne accorgi subito e sai esattamente dove.

**Opzione B — Modalità batch**
Riscrivi tutte le sezioni in parallelo, mostra diff completo di tutte, l'utente approva una volta, push finale in un comando solo.
→ Più veloce ma meno granulare: un eventuale errore si nota solo alla fine.

**Opzione C — Misto**
L'utente può partire sezione-per-sezione e a metà passare a batch (o viceversa). Rispetta la scelta corrente finché non chiede diversamente.

Salva la scelta e procedi con il sotto-flusso corrispondente qui sotto.

### 5A — Sotto-flusso sezione per sezione

Per ogni sezione nel template nuovo, in ordine di apparizione nel JSON (top → bottom):

1. **Apri il file duplicato** `sections/<nuovo-prefisso>-<suffix>.liquid` con `Read` e identifica:
   - Il ruolo della sezione (hero, carousel testimonial, video, dottori, "perché funziona", hotspot, comparison, FAQ, reviews, closing CTA, …). Mappa il ruolo dal nome del file usando `references/image-specs-per-section.md`.
   - Tutti i testi attualmente presenti (del prodotto base): headline, sub-headline, bullet/paragrafi, CTA, label card, domande FAQ, testimonianze, etc.
   - Gli URL immagine attualmente presenti (verranno sostituiti in Fase 6, non qui).

2. **Proponi all'utente i testi riscritti** per il nuovo prodotto, derivati dalla research:
   - Per ogni testo base identificato, produci la versione equivalente per il nuovo prodotto con lo stesso peso/lunghezza/tono.
   - Usa claim, proof points, ingredienti, tono dalla research (Fase 4).
   - **Tutti i testi devono essere coerenti con `pdp.angle`** scelto in Fase 4.1. Se una sezione chiede di virare su un altro angolo, segnala e chiedi prima di procedere.
   - Mantieni la stessa struttura (stesso numero di bullet, stesse label di card, stesso numero di FAQ, ecc.). Se la research non basta per riempire una struttura (es. 5 card ma solo 3 claim), **chiedi input mirato** all'utente invece di inventare.
   - Mostra la proposta in formato diff chiaro:
     ```
     Sezione: <nuovo-prefisso>-<suffix>.liquid (<ruolo>)
     — Headline
        prima:  "Il segreto della berberina per abbassare il colesterolo"
        dopo:   "La crema che rassoda la pelle in 4 settimane"
     — Sub-headline
        prima:  "..."
        dopo:   "..."
     — Bullet 1/3
        prima:  "..."
        dopo:   "..."
     ...
     ```

3. **Aspetta conferma o correzioni** dall'utente prima di scrivere nel file. L'utente può:
   - Confermare tutto → vai al punto 4.
   - Correggere singoli testi → applica le correzioni e riconferma.
   - Chiedere un tono diverso (più diretto, più emotivo, più tecnico) → rigenera.

4. **Applica le sostituzioni** SOLO sul file duplicato `sections/<nuovo-prefisso>-<suffix>.liquid`, **dentro il blocco `{% schema %}`** (campi `default` di `settings` e `blocks`):
   - Per poche sostituzioni: `Edit` con `old_string` / `new_string` esatti sui `default` nello schema JSON.
   - Per molte sostituzioni (es. FAQ con 7 Q&A → 7 blocks da aggiornare, reviews wall con 10+ blocks): `Bash` con `python3 <<'PY' ... PY` heredoc. Pattern: parsa lo schema come JSON, modifica i `default`, riscrivi il blocco schema nel file.
   - **Non toccare**: markup HTML sopra lo schema, classi CSS, blocchi `<script>`/`<style>`, attributi `data-*`, URL immagine in `default` di `image_picker` (verranno sostituiti dall'utente dal theme editor in Fase 6), Liquid tags.
   - Se durante la modifica noti che un testo è ancora hardcoded nel markup (cioè la liquidify di Fase 3.5 non l'ha coperto), segnala e proponi di completare la liquidify su quella sezione prima di continuare — non patchare nel markup.

5. **Verifica no-regressioni sul template base**: prima del push, conferma a te stesso di non aver aperto/modificato per sbaglio file `sections/<vecchio-prefisso>-*.liquid` o `templates/product.<vecchio-nome>.json`. Se l'hai fatto, fermati e ripristina.

6. **Push selettivo** della sola sezione modificata:
   ```bash
   cd "<store.workdir_path>"
   set -a; source "<store.env_path>"; set +a
   npx @shopify/cli@latest theme push \
     --theme <store.theme_id> --nodelete --allow-live \
     --only "sections/<nuovo-prefisso>-<suffix>.liquid"
   ```

7. Chiedi all'utente di aprire l'URL live (`https://<store.shopify_domain>/products/<product.slug>`) e confermare testi/layout su mobile e desktop.

8. Gestisci aggiustamenti (font-size, white-space, nowrap, spelling) come modifiche minime incrementali — mai rifare la sezione.

9. Quando l'utente conferma, prima di passare alla sezione successiva **chiedi esplicitamente** con `AskUserQuestion`:

   "Sezione confermata. Come vuoi procedere con le restanti N sezioni?"
   - **Continua sezione-per-sezione** → ripeti il ciclo dal punto 1 sulla prossima sezione.
   - **Passa a batch (completa tutte le restanti insieme)** → switcha al sotto-flusso 5B (batch) per le sezioni ancora da fare. Il progresso sulle sezioni già confermate resta intatto.

   Rispetta la scelta. Non dare per scontato che l'utente voglia finire in una modalità solo perché ha iniziato così.

Continua (nella modalità corrente) fino all'ultima sezione del template. A fine fase, tutti i testi del nuovo prodotto sono live, le immagini sono ancora quelle vecchie (placeholder del prodotto base): vengono sostituite in Fase 6.

### 5B — Sotto-flusso batch

1. **Leggi tutte le sezioni duplicate in parallelo** con più chiamate `Read` in un singolo messaggio. Per ciascuna identifica ruolo + testi da riscrivere.
2. **Pianifica la riscrittura integrata di tutta la PDP**: i testi tra sezioni devono essere coerenti (hero e FAQ devono parlare lo stesso linguaggio, i claim principali vanno ripetuti con stessa terminologia, i numeri dei proof points tornano).
3. **Mostra all'utente un diff completo di tutte le sezioni in un unico messaggio**, raggruppato per sezione:
   ```
   === svc-hero-badge.liquid (hero) ===
   — Headline
      prima: "..."
      dopo:  "..."
   ...

   === svc-pdp-05.liquid (carousel testimonial) ===
   ...
   ```
4. **Aspetta approvazione globale o correzioni puntuali**:
   - Se l'utente approva tutto → applica le modifiche a tutti i file in parallelo (multiple `Edit` o `Bash` heredoc con `python3` che scrive più file).
   - Se chiede correzioni su sezioni specifiche → applica solo quelle correzioni e ripresenta il diff aggiornato per riconferma.
   - Se chiede un cambio di tono globale → rigenera tutto.
5. **Verifica no-regressioni sul template base**: prima del push, conferma a te stesso di non aver aperto/modificato per sbaglio file del prefisso base.
6. **Push selettivo unico** con tutte le sezioni in un solo comando:
   ```bash
   cd "<store.workdir_path>"
   set -a; source "<store.env_path>"; set +a
   npx @shopify/cli@latest theme push \
     --theme <store.theme_id> --nodelete --allow-live \
     --only "sections/<nuovo-prefisso>-<suffix1>.liquid" \
     --only "sections/<nuovo-prefisso>-<suffix2>.liquid" \
     # ...una per ogni sezione duplicata
   ```
7. Chiedi all'utente di aprire l'URL live e fare un check completo della PDP. Raccogli eventuali aggiustamenti e applicali puntualmente (in questa fase i fix successivi sono sezione-per-sezione, anche se il primo push era batch).

### 5C — Misto

L'utente può cambiare modalità in corsa dicendo "passa a batch" o "torna a sezione per sezione". Rispetta la scelta corrente senza resettare il progresso: continua dalla prossima sezione non ancora lavorata.

---

## Fase 6 — Guida immagini

Le sezioni sono editabili: le immagini sono `image_picker` nello schema. L'utente le carica **direttamente dal theme editor**, nessun replace di URL CDN nei file. Fonte: `references/image-specs-per-section.md` (specs) + `references/section-schema-patterns.md` (pattern editor).

Per ogni sezione che contiene immagini:
1. Identifica il numero e ruolo delle immagini richieste (grep degli `image_picker` nello schema).
2. Mostra all'utente il brief:
   - Ruolo dell'immagine (cosa deve mostrare).
   - Ratio consigliato + dimensioni target + peso max.
   - Nome del campo `image_picker` nel theme editor (es. `Hero image`, `Benefit 2 icon`).
3. Istruzioni utente:
   ```
   1. Apri il theme editor: Admin → Online Store → Themes → Customize (sul tema <store.theme_name>).
   2. Vai alla PDP del nuovo prodotto (top-left picker → Products → <nome prodotto>).
   3. Seleziona la sezione <nuovo-prefisso>-<suffix>.
   4. Click sul campo "<label image_picker>" → Select image → Upload → scegli il file → Save.
   ```
4. L'utente conferma in chat che ha caricato.
5. Nessun push necessario: le immagini sono salvate nei settings del template lato Shopify, non nei file Liquid del tema.
6. Chiedi conferma visiva sull'URL live.

---

## Fase 7 — Verifica finale

1. Mostra la checklist finale:
   - [ ] Tutte le sezioni popolate con testi validati
   - [ ] Tutte le immagini caricate e renderizzate correttamente
   - [ ] Mobile (viewport 375px) ok
   - [ ] Desktop (viewport 1440px) ok
   - [ ] DevTools console pulita (no errori Liquid, no 404 su immagini)
   - [ ] URL diretta funzionante: `https://<store.shopify_domain>/products/<product.slug>`
   - [ ] Custom domain (se presente): `https://<store.custom_domain>/products/<product.slug>`
2. Chiedi all'utente di confermare ogni check o segnalare problemi.
3. Per ogni problema segnalato: torna alla fase corrispondente (5 per testi, 6 per immagini) e fai le modifiche.
4. Quando tutto è ✅: dichiara la PDP pronta. Suggerisci:
   - Aggiungere il prodotto al catalogo/collezione giusta.
   - Impostare pricing, inventory, variants.
   - Collegare il sistema bundling (se usato) — se il prodotto usa Katching o simili, consulta `references/workflow-faithful-rebuild.md` per pattern form nativo.

## Troubleshooting rapido

| Sintomo                                         | Possibile causa                                       | Fix                                                                 |
| ----------------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------- |
| `theme list` ritorna 401                        | Token scaduto/revocato                                | Rigenera Theme Access token, aggiorna `.env`                        |
| Push fallisce con "Liquid syntax error"         | Sostituzione ha rotto Liquid                          | Leggi il file, localizza errore, `Edit` per riparare                |
| Sezione non appare su PDP live                  | Template JSON non aggiornato o sezione non pushata   | Verifica `templates/product.<nome>.json` + push sezione             |
| Stile CSS diverso dall'originale                | Hash GemPages toccato per errore                      | Ripristina il file originale e rifai solo la sostituzione testi     |
| Product non usa il nuovo template                | Template suffix non selezionato in Admin              | Admin → Product → sidebar → Theme template → seleziona nuovo nome   |

## File correlati (references)

- `references/workflow-faithful-rebuild.md` — regola d'oro + tecniche di sostituzione.
- `references/auth-pattern.md` — `.env`, Theme Access token, verifica connessione.
- `references/selective-push.md` — comando push standard + flag.
- `references/section-naming.md` — convenzioni prefissi + aggiornamento schema.
- `references/image-specs-per-section.md` — dimensioni/ratio per tipo sezione.
- `references/section-schema-patterns.md` — regole di liquidify: schema editabile, pattern before/after, tipi di setting, edge case.
