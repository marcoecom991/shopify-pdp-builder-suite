---
name: create-new-pdp
description: Crea una nuova PDP Shopify da zero. Guida l'utente attraverso 7 fasi — scelta store, verifica auth, duplicazione template e sezioni con nuovo prefisso, raccolta materiali di ricerca sul prodotto, popolamento testi sezione-per-sezione mantenendo i layout esistenti, indicazioni sulle immagini da caricare, verifica finale. Da usare ogni volta che un membro del team deve pubblicare una PDP per un nuovo prodotto su uno degli store configurati (Nimea, Glowria, o altri aggiunti a config/stores.json).
---

# /create-new-pdp — Orchestrator

Sei una guida passo-passo per creare una PDP Shopify da zero. Segui le 7 fasi **in sequenza**. Non saltare fasi. Usa `AskUserQuestion` (o in mancanza chiedi in chiaro) come gate tra una fase e la successiva per confermare progressi con l'utente.

## Principio generale

- **🔒 Il template base e le sue sezioni NON si toccano MAI.** Il template base (es. `product.berberina-pills.json`) e le sue sezioni (es. `cboe-pdp-05.liquid`) appartengono a un prodotto live diverso. Ogni modifica a quei file rompe un prodotto esistente. **Si duplicano sempre in file nuovi con prefisso nuovo**, e TUTTE le modifiche avvengono solo sui duplicati. Mai `Edit` o `Write` sui file base.
- **Non riscrivere il markup esistente.** Quando popoli le sezioni duplicate con i contenuti del nuovo prodotto, preservi layout, CSS, JS della sezione di partenza. Modifichi solo testi e URL immagini. Fonte di verità: `references/workflow-faithful-rebuild.md`.
- **Un push per volta, sempre selettivo.** Mai `theme push` senza `--only`. Il `--only` include SOLO i file duplicati (nuovo prefisso). Fonte di verità: `references/selective-push.md`.
- **Conferma con l'utente prima di ogni azione distruttiva o irreversibile** (creazione file, push live, rinomina).

## Lettura dello stato iniziale

All'avvio:
1. Leggi `config/stores.json` (path: `<plugin-root>/config/stores.json`). Se NON esiste, copialo da `config/stores.example.json` e dì all'utente di compilarlo con i suoi path locali e theme IDs prima di ricominciare. Stop.
2. Se esiste, estrai la lista degli store.

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

### 3.5 Push iniziale

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

### 3.6 Istruzioni per il Product Shopify

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

Dopo che l'utente ha fornito tutto: fai un riepilogo sintetico (nome prodotto, 3 claim principali, 3 proof points, palette) e chiedi conferma "Materiali completi, procedo con la scrittura testi sezione-per-sezione?"

---

## Fase 5 — Riscrittura testi per il nuovo prodotto

**Contesto importante**: le sezioni duplicate contengono ATTUALMENTE i testi del prodotto base (es. sezioni copiate da `berberina-pills` parlano ancora di berberina). Il tuo compito in questa fase è **riscrivere i testi del nuovo prodotto** partendo dalla research raccolta in Fase 4, mantenendo IDENTICI markup/CSS/JS/struttura della sezione.

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

4. **Applica le sostituzioni** SOLO sul file duplicato `sections/<nuovo-prefisso>-<suffix>.liquid`:
   - Per poche sostituzioni: `Edit` con `old_string` / `new_string` esatti, uno per testo.
   - Per molte sostituzioni (es. sezione FAQ con 7 Q&A, o reviews wall con 10+ recensioni): `Bash` con `python3 <<'PY' ... PY` heredoc, usando il pattern `s, n = src.replace(old, new, 1); assert n == 1, "…"` per ogni replace. Vedi `references/workflow-faithful-rebuild.md`.
   - **Non toccare**: markup HTML, classi CSS, blocchi `<script>`/`<style>` non-testuali, attributi `data-*`, URL immagine (verranno in Fase 6), `{% schema %}`, Liquid tags.

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

9. Quando l'utente conferma, passa alla sezione successiva.

Continua fino all'ultima sezione del template. A fine fase, tutti i testi del nuovo prodotto sono live, le immagini sono ancora quelle vecchie (placeholder del prodotto base): vengono sostituite in Fase 6.

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

Una volta che tutti i testi sono validati, passa alla fase immagini. Fonte: `references/image-specs-per-section.md`.

Per ogni sezione che contiene immagini:
1. Identifica il numero e ruolo delle immagini richieste.
2. Mostra all'utente il brief (vedi template a fine `image-specs-per-section.md`):
   - Ruolo dell'immagine (cosa deve mostrare).
   - Ratio consigliato.
   - Dimensioni target.
   - Peso max.
   - URL placeholder attuale da sostituire.
3. L'utente prepara l'immagine offline, la carica su Shopify Admin → Settings → Files, copia l'URL CDN.
4. L'utente incolla nella chat l'URL o gli URL (uno per immagine richiesta).
5. Fai il replace nel file `.liquid` (Edit mirato o Python heredoc), poi push selettivo.
6. Chiedi conferma visiva.

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
