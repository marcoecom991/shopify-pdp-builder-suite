---
name: create-new-funnel
description: Crea una nuova pagina funnel Shopify da zero (advertorial, listicle, quiz, o custom). Guida l'utente attraverso 8 fasi — scelta store, auth + scelta tema, brand identity discovery da una PDP esistente, creazione template page (con o senza header/footer), scelta tipologia funnel, raccolta materiali (competitor + research prodotto + URL PDP per le CTA), costruzione sezioni con modalità sezione-per-sezione/batch/misto, guida immagini. Da usare ogni volta che un membro del team deve pubblicare un funnel pre-PDP su uno degli store configurati.
---

# /create-new-funnel — Orchestrator

Sei una guida passo-passo per creare una pagina funnel Shopify da zero. Il funnel è una **pagina (`templates/page.*`) non un prodotto**: ospita contenuti storytelling / listicle / quiz che portano l'utente a cliccare una CTA verso una PDP esistente.

Segui le 8 fasi **in sequenza**. Non saltare fasi. Usa `AskUserQuestion` come gate tra una fase e la successiva.

## Principi generali

- **🔒 Mai toccare template o sezioni esistenti.** Le PDP e altre page template dello store sono intoccabili. Il funnel vive in file nuovi con prefisso nuovo (mai uguale a uno già presente).
- **Brand-consistent**: i colori e font del funnel vengono **ereditati** da una PDP di riferimento (Fase 3). Non inventare palette/tipografia.
- **CTA → PDP**: tutti i bottoni CTA del funnel puntano all'URL PDP del prodotto fornito in Fase 6. Mai `href="#"` o URL placeholder.
- **Mobile-first**: il traffico funnel è >80% mobile (ads paid). Ogni sezione deve leggersi benissimo a 375px.
- **No framework esterni**: niente Tailwind, Bootstrap, ecc. Solo CSS scoped dentro la sezione, con class-prefix che match del prefisso file.
- **Un push per volta, selettivo**. Mai `theme push` senza `--only`. Il `--only` include SOLO i file del nuovo funnel.
- **Conferma prima di ogni azione irreversibile** (creazione file, push live, rinomina).

## Lettura dello stato iniziale

All'avvio:
1. Leggi `config/stores.json` dal `<plugin-root>/config/stores.json`. Se NON esiste, copialo da `config/stores.example.json` e chiedi all'utente di compilarlo con path locali + theme IDs. Stop.
2. Se esiste, estrai la lista degli store.

---

## Fase 1 — Scelta store

Mostra all'utente la lista store configurati. Usa `AskUserQuestion` con un'opzione per store + "Altro".

Se "Altro": chiedi nome, `shopify_domain` (*.myshopify.com), theme ID, path workdir, path env. Aggiungi voce a `stores.json`.

Salva in memoria: `store.name`, `store.shopify_domain`, `store.theme_id`, `store.workdir_path`, `store.env_path`.

---

## Fase 2 — Verifica auth + scelta tema

1. Se `store.env_path` non esiste: guida alla creazione del `.env` (Theme Access token → `shptka_*`). Fonte: `references/auth-pattern.md`.
2. Lancia:
   ```bash
   cd "<store.workdir_path>"
   set -a; source "<store.env_path>"; set +a
   npx @shopify/cli@latest theme list --no-color
   ```
3. Se 401 / Invalid API key: token scaduto, utente rigenera e riprova.
4. **Mostra i temi** parsati: `[live]`, `[unpublished]`, `[development]` con nome + ID.
5. `AskUserQuestion`: "Su quale tema operare?" Default: `[live]` (o `store.theme_id` del config). Opzioni: una per tema.
6. Salva scelta in `store.theme_id` (sovrascrive config per sessione) e `store.theme_name`.
7. **Pull fresco**:
   ```bash
   cd "<store.workdir_path>"
   set -a; source "<store.env_path>"; set +a
   npx @shopify/cli@latest theme pull --theme <store.theme_id> --nodelete
   ```
   Chiedi conferma prima (il pull sovrascrive file locali non pushati).

---

## Fase 3 — Brand identity discovery

Obiettivo: capire palette + tipografia del brand per mantenere il funnel visivamente coerente col resto del sito. Fonte di verità: `references/brand-identity-discovery.md`.

### 3.1 Scelta PDP di riferimento

1. Elenca i `product.*.json` in `<store.workdir_path>/templates/`.
2. `AskUserQuestion`: "Quale PDP usare come riferimento per colori e tipografia?"
   - Mostra i nomi leggibili (es. `berberina-pills`, `crema-borse-occhiaie-pdp`).
   - Se ce n'è uno solo (oltre al default `product.json`) assumilo automaticamente.
3. Salva in `brand.reference_pdp`.

### 3.2 Estrazione automatica

Leggi in parallelo:
- `templates/product.<reference>.json`
- 2-3 sezioni principali referenziate (hero, prima sezione con CTA button)
- `config/settings_data.json` (color_schemes + type_header_font + type_body_font)
- `config/settings_schema.json` (opzionale, per nomi umani dei settings)

Estrai secondo le regole di `brand-identity-discovery.md`:
- **Palette**: `primary`, `primary_text`, `secondary`, `accent`, `text`, `background` (HEX).
- **Tipografia**: `heading_font`, `body_font` (normalizzati da Shopify handle a "Nome, fallback").
- **Stili ricorrenti**: border-radius, shadow, heading weight/style.

### 3.3 Mostra riepilogo e chiedi conferma

Presenta all'utente un blocco leggibile (esempio in `brand-identity-discovery.md`, sezione "Output atteso"). Usa `AskUserQuestion` o chiedi in chiaro:
- "Confermi questo brand profile?" → procedi.
- "Correggo un valore" → l'utente dice cosa cambiare, applica, ripresenta.
- "Cambio PDP di riferimento" → torna a 3.1.

Salva l'oggetto `brand.*` in memoria. Verrà iniettato nelle sezioni in Fase 7.

---

## Fase 4 — Creazione template funnel

### 4.1 Scelta partenza

`AskUserQuestion`: "Partire da zero o duplicare un template page esistente?"
- **Da zero** (default, caso tipico).
- **Duplica** esistente (se il team ha già un funnel "buono" da cui partire).

Se duplica: riusa il flusso della Fase 3 di `/create-new-pdp` (lista `page.*.json`, prefisso, duplicazione sezioni). Fonte: `references/section-naming.md`.

### 4.2 Nome nuovo template

`AskUserQuestion`: "Nome del nuovo template funnel?"
- Kebab-case, solo `[a-z0-9-]`.
- Esempi: `autoabbronzante-advertorial`, `berberina-listicle`, `crema-rassodante-quiz`.
- Valida che `<workdir>/templates/page.<nome>.json` NON esista.

Salva in `funnel.template_name`.

### 4.3 Layout — header/footer on/off

`AskUserQuestion`: "Mantenere header e footer Shopify?"

Opzioni:
- **No, chromeless (Recommended)** → pagina full-screen senza navbar/footer. Layout: `theme.chromeless` (se supportato) o custom.
- **Sì, tema standard** → header + footer del tema sempre visibili. Layout: `theme`.

Fonte: `references/page-template-layout.md`.

Se l'utente sceglie chromeless:
- Verifica con `Glob` o `ls` se esiste `<workdir>/layout/theme.chromeless.liquid`.
- Se NO: chiedi all'utente se vuole (a) creare un layout custom chromeless ora, (b) procedere con `theme` e poi decidere, (c) usare nome layout diverso.
- Se SÌ: usa direttamente `"layout": "theme.chromeless"`.

Salva in `funnel.layout`.

### 4.4 Prefisso sezioni

Proponi un prefisso di default dalle iniziali del nome (vedi `references/section-naming.md`, sezione "Naming per FUNNEL"):
- `autoabbronzante-advertorial` → `aa-` (breve) o `aa-adv-` (esplicita).
- `berberina-listicle` → `bl-` o `bb-lst-`.

`AskUserQuestion`: "Prefisso sezioni: `<default>`? (conferma o alternativo)"

Salva in `funnel.prefix`.

### 4.5 Generazione template scheletro

Scrivi `templates/page.<funnel.template_name>.json` con scheletro minimo:

```json
{
  "sections": {},
  "order": [],
  "layout": "<funnel.layout>"
}
```

(Le sezioni verranno aggiunte progressivamente in Fase 7.)

Se si è scelta la duplicazione da template esistente (4.1): applica la logica di duplicazione della Fase 3 PDP e salta i punti seguenti.

### 4.6 Push selettivo template

```bash
cd "<store.workdir_path>"
set -a; source "<store.env_path>"; set +a
npx @shopify/cli@latest theme push \
  --theme <store.theme_id> --nodelete --allow-live \
  --only "templates/page.<funnel.template_name>.json"
```

### 4.7 Istruzioni manuali Page in Admin

Mostra all'utente:

```
Ora crea la Page in Shopify Admin:
1. Admin → Online Store → Pages → Add page
2. Title: "<titolo visibile della pagina>" (es. "Scopri Autoabbronzante Glowria")
3. Content: lascialo vuoto (le sezioni vengono dal template)
4. In "Theme template" (sidebar destra): seleziona `<funnel.template_name>`
5. Save
6. Torna qui e dimmi lo slug della page (es. `scopri-autoabbronzante`), così posso riferirmi alla URL live
```

Salva lo slug in `funnel.page_slug`. URL live: `https://<store.shopify_domain>/pages/<funnel.page_slug>`.

---

## Fase 5 — Scelta tipo di funnel

`AskUserQuestion`: "Che tipo di funnel stiamo creando?"

Opzioni:
- **Advertorial** — articolo narrativo, storytelling personale che porta a PDP. Target freddo.
- **Listicle** — articolo "N motivi per", benefici ordinati, confronto. Target semi-caldo.
- **Quiz** — domande + risultato personalizzato, engagement-driven. Target freddo.
- **Altro** — descrivi liberamente la struttura, poi forniamo esempi.

Salva in `funnel.type` ∈ {`advertorial`, `listicle`, `quiz`, `other`}.

Carica la struttura base di riferimento da `references/funnel-types.md` per il tipo scelto. Sarà usata come **ipotesi di partenza** in Fase 7; verrà raffinata con gli esempi competitor di Fase 6.

Se `other`: chiedi all'utente una descrizione della struttura che ha in mente (numero sezioni, ordine, ruoli) — serve come punto di partenza.

---

## Fase 6 — Raccolta materiali

Presenta una checklist all'utente. Aspetta che abbia fornito tutto prima di procedere.

### 6.1 Esempi competitor del tipo scelto

Chiedi:
- 2-5 **screenshot / URL** di funnel competitor dello stesso tipo.
- Per ogni esempio, un breve contesto: perché ti piace, cosa copieresti, cosa cambieresti.

Se l'utente non ha esempi pronti, suggerisci:
- Typology hunt: swipefile.com, advertorial.com, altri aggregator.
- Screenshot da Facebook Ads Library di funnel visti su ads recenti.

Claude analizza gli esempi per:
- Confermare/raffinare la struttura base del tipo scelto.
- Identificare pattern di hook, claim, obiezioni, CTA.
- Estrarre lunghezza/density media.

### 6.2 Research prodotto

Identica a `/create-new-pdp` Fase 4:
- Ingredienti chiave (nome + effetto + dosaggio).
- Claim principali (3-5 effetti del prodotto).
- Target audience (chi, età, problema).
- Differenziazione vs competitor.
- Proof points (certificazioni, test clinici, numeri, recensioni reali).

### 6.3 URL PDP del prodotto (OBBLIGATORIO)

`AskUserQuestion` o chiedi in chiaro: "Qual è l'URL PDP del prodotto su questo store?"

Esempi:
- `https://glowria.com/products/crema-corpo-autoabbronzante-ufficiale`
- `https://nimea.com/products/berberina-pills`

Valida che:
- Inizia con `https://`.
- Contiene `/products/<slug>`.

Salva in `funnel.cta_url`. Sarà il `href` di **TUTTI** i bottoni CTA del funnel.

### 6.4 Angolo marketing corrente

Chiedi:
- 1-3 copy ads attuali (headline + descrizione).
- Concept video se esistente (link o transcript).

Serve per allineare il tono del funnel all'ads che porterà traffico.

### 6.5 Brand assets extra

Chiedi:
- Logo URL CDN (se diverso da quello già nel tema).
- Eventuali asset custom (pattern, mockup, loghi certificazioni).

### 6.6 Riepilogo e conferma

Fai un riepilogo sintetico:
- Tipo funnel + struttura pianificata (N sezioni).
- 3 claim principali prodotto.
- 3 proof points.
- Palette + tipografia (richiama `brand.*` da Fase 3).
- URL CTA.

Chiedi: "Tutto chiaro, procedo con la costruzione delle sezioni?"

---

## Fase 7 — Costruzione sezioni

### 7.0 Scelta modalità di lavoro (obbligatoria)

`AskUserQuestion`: "Come vuoi lavorare sulle sezioni?"

**Opzione A — Sezione per sezione (Recommended per prima PDP)**
Per ogni sezione: crea file → push → verifica live → conferma → prossima sezione.
Più lento ma sicuro.

**Opzione B — Batch**
Crea tutte le sezioni in parallelo → mostra preview completa → l'utente approva una volta → push finale unico.
Più veloce ma meno granulare.

**Opzione C — Misto leggero**
Checkpoint intermedi solo sulle sezioni complesse (hero, CTA offer, quiz-wrapper). Sezioni semplici (FAQ, footer, closing) passano in batch senza conferma individuale.
Compromesso consigliato per utenti già rodati.

Salva scelta in `funnel.build_mode` ∈ {`A`, `B`, `C`}.

### 7.1 Pianifica la lista sezioni

In base a:
- Struttura base del tipo funnel (Fase 5).
- Aggiustamenti dagli esempi competitor (Fase 6.1).

Produci una lista esplicita con nome file + ruolo + note:

```
Struttura pianificata (es. advertorial, prefisso `aa-`):

01. sections/aa-01-announcement.liquid  → announcement bar
02. sections/aa-02-hero.liquid          → hero advertorial
03. sections/aa-03-problem.liquid       → problem hook
04. sections/aa-04-story.liquid         → storia personale
05. sections/aa-05-solution.liquid      → intro prodotto
06. sections/aa-06-how-it-works.liquid  → come funziona + ingredienti
07. sections/aa-07-proof.liquid         → social proof
08. sections/aa-08-faq.liquid           → obiezioni
09. sections/aa-09-cta-offer.liquid     → CTA + offer
10. sections/aa-10-trust-badges.liquid  → badge garanzia
11. sections/aa-11-footer.liquid        → footer disclaimer
```

`AskUserQuestion` di conferma/modifica. L'utente può aggiungere, rimuovere, riordinare, rinominare sezioni.

### 7.2 Regole per ogni sezione costruita

Per ogni `sections/<funnel.prefix><NN>-<ruolo>.liquid`:

1. **Markup mobile-first**, poi `@media (min-width: 768px)` per desktop. Classi scoped con prefisso che matcha il nome file (es. tutte le classi dentro `aa-02-hero.liquid` iniziano con `.aa-02`).
2. **Stili inline** nel file `.liquid` dentro `<style>` scoped (no `assets/*.css` esterni a meno che sia strettamente necessario). Usa valori di `brand.*`:
   - Bottoni CTA: `background: <brand.primary>`, `color: <brand.primary_text>`, `border-radius: <brand.button_radius>`, hover più scuro.
   - Headline: `font-family: <brand.heading_font>`, `font-weight: <brand.heading_weight>`.
   - Body: `font-family: <brand.body_font>`, `color: <brand.text>`.
3. **Testi** dal research di Fase 6.2 + ispirazione competitor (Fase 6.1), con density/lunghezza coerente al ruolo (vedi `references/funnel-types.md`).
4. **Bottoni CTA** puntano a `<funnel.cta_url>` con `<a class="<prefix>__cta" href="<funnel.cta_url>">`. Ripeti la CTA almeno 2 volte nel funnel (hero + closing minimo).
5. **Placeholder immagini** espliciti (vedi `references/funnel-image-specs.md`): SVG data-uri con dimensioni + alt-text `IMG_PLACEHOLDER_<NN>_<ruolo>`. Non usare URL fittizi via/placeholder.com.
6. **Schema Liquid minimo**:
   ```liquid
   {% schema %}
   {
     "name": "<NOME LEGGIBILE SEZIONE>",
     "tag": "section",
     "class": "<prefix>-<NN>",
     "presets": [
       { "name": "<NOME LEGGIBILE SEZIONE>" }
     ]
   }
   {% endschema %}
   ```
7. **Aggiorna il template JSON** `templates/page.<funnel.template_name>.json`:
   - Aggiungi la sezione al blocco `sections`:
     ```json
     "<NN>": { "type": "<funnel.prefix><NN>-<ruolo>", "settings": {} }
     ```
   - Aggiungi la chiave all'array `order` nella posizione giusta.

### 7.3 Workflow per Opzione A (sezione per sezione)

Loop sulle sezioni in ordine 01→N:

1. Crea il file `.liquid` con `Write`.
2. Aggiorna `templates/page.<funnel.template_name>.json` aggiungendo la nuova sezione (usa `Edit` o `Read`+`Write`).
3. Push selettivo:
   ```bash
   cd "<store.workdir_path>"
   set -a; source "<store.env_path>"; set +a
   npx @shopify/cli@latest theme push \
     --theme <store.theme_id> --nodelete --allow-live \
     --only "sections/<funnel.prefix><NN>-<ruolo>.liquid" \
     --only "templates/page.<funnel.template_name>.json"
   ```
4. Chiedi all'utente di aprire `https://<store.shopify_domain>/pages/<funnel.page_slug>` e confermare mobile+desktop.
5. Gestisci aggiustamenti minimi (typo, font-size, spacing) in loop finché confermato.
6. Passa alla sezione successiva.

### 7.4 Workflow per Opzione B (batch)

1. Crea TUTTI i file `.liquid` in parallelo (multiple `Write` nel singolo messaggio).
2. Scrivi il `templates/page.<funnel.template_name>.json` definitivo con tutte le sezioni ordinate.
3. Mostra all'utente un **diff-preview** testuale della struttura e i punti chiave di ogni sezione (headline + primo paragrafo). Non dump di tutto il markup.
4. Aspetta approvazione globale o correzioni puntuali.
5. Push selettivo unico con `--only` di tutti i file (template + sezioni in un comando).
6. Utente verifica live.
7. Aggiustamenti successivi: passa automaticamente a modalità sezione-per-sezione per i fix puntuali.

### 7.5 Workflow per Opzione C (misto leggero)

1. Identifica sezioni "complesse" (hero, problem, solution, proof, cta-offer, quiz-wrapper, result) vs "semplici" (announcement, trust-badges, faq, footer, disclaimer).
2. Per le complesse: workflow A (checkpoint individuale).
3. Per le semplici: crea in batch, push unico intermedio senza conferma per ciascuna.
4. A fine fase: pull completo live, utente verifica tutto insieme.

### 7.6 Verifica no-regressioni

Prima di ogni push: conferma a te stesso di non aver aperto/modificato per sbaglio file esistenti fuori dal prefisso `<funnel.prefix>`. Se è successo: stop, ripristina, riprova.

---

## Fase 8 — Guida immagini

Una volta che tutti i testi/layout sono validati, passa alle immagini. Fonte: `references/funnel-image-specs.md`.

Per ogni sezione che ha placeholder immagini (cerca `IMG_PLACEHOLDER_` nei file `<funnel.prefix>*.liquid`):

1. Identifica il numero di immagini e il ruolo di ciascuna.
2. Genera il brief:
   ```
   Sezione: <prefix>-<NN>-<ruolo>.liquid
   Immagini richieste: <N>
   
   Per ogni immagine:
     - Ruolo: <es. "hero lifestyle protagonista + emozione">
     - Ratio: <4:5>
     - Dimensioni: <1200×1500>
     - Peso max: 150 KB (.webp)
     - Placeholder da sostituire: IMG_PLACEHOLDER_<NN>_<ruolo>
   
   Passi utente:
     1. Prepara l'immagine.
     2. Upload in Shopify Admin → Settings → Files.
     3. Copia l'URL CDN.
     4. Incollalo qui.
   ```
3. L'utente incolla l'URL CDN (`https://cdn.shopify.com/s/files/1/...`).
4. Fai replace nel file `.liquid` (Edit mirato sul placeholder specifico).
5. Push selettivo della sezione.
6. Chiedi conferma visiva.
7. Prossima sezione.

Modalità anche qui: A (per singola sezione), B (batch tutte le immagini insieme), C (solo su sezioni con immagini critiche). Usa la stessa `funnel.build_mode` di Fase 7 se l'utente non dice altro.

---

## Verifica finale

Checklist chiusura:
- [ ] Template `templates/page.<funnel.template_name>.json` pushato e sincronizzato.
- [ ] Tutte le sezioni pushate, schema Liquid senza errori.
- [ ] URL live funzionante: `https://<store.shopify_domain>/pages/<funnel.page_slug>`.
- [ ] Mobile 375px: tutte le sezioni leggibili, bottoni cliccabili, testi non troncati.
- [ ] Desktop 1440px: layout si espande correttamente, max-width container coerente.
- [ ] Palette + tipografia coerenti con `brand.*` (spot check visivo vs PDP di riferimento).
- [ ] Tutte le CTA cliccate → portano a `<funnel.cta_url>` (PDP prodotto).
- [ ] Nessuna immagine placeholder rimasta (grep `IMG_PLACEHOLDER_` deve ritornare 0 match nei file del funnel).
- [ ] DevTools console: no errori Liquid, no 404 su immagini.
- [ ] Header/footer come da scelta (chromeless = assenti, theme = presenti).

Chiedi conferma su ciascun check. Se qualcosa fallisce: torna alla fase corrispondente, fix, re-push.

Quando tutto ✓: dichiara il funnel pronto. Suggerisci:
- Impostare UTM tracking sugli URL degli ads che porteranno qui.
- Configurare Meta Pixel / Google Analytics sulla pagina (via theme settings o code injection).
- Testare un click CTA end-to-end (dalla page al PDP al checkout).

## Troubleshooting rapido

| Sintomo                                         | Causa                                          | Fix                                                                |
|-------------------------------------------------|------------------------------------------------|--------------------------------------------------------------------|
| La page live mostra ancora header/footer        | Layout non chromeless o tema senza theme.chromeless | Verifica `"layout"` nel template JSON; crea layout custom se manca |
| Bottoni non portano al prodotto                 | `href` vuoto o `#` in qualche sezione           | Grep `href="` nelle sezioni, verifica tutti = `<funnel.cta_url>`    |
| Font non coerente col resto del sito            | `brand.*` non applicato a tutte le sezioni      | Ri-scansiona i `.liquid` e inserisci `font-family: <brand.heading_font>` negli h1/h2 |
| Sezione invisibile live                         | Non aggiunta all'`order` del template JSON      | Apri `templates/page.<nome>.json`, verifica array `order`           |
| Push fallisce con Liquid syntax error           | Schema malformato, brace unmatched              | Leggi error line, `Edit` puntuale, ri-push                         |
| Mobile text overflow                            | Font-size fissi in px anziché clamp             | Sostituisci con `clamp(1rem, 4vw, 1.5rem)` o media query           |

## File correlati (references)

- `references/auth-pattern.md` — `.env`, Theme Access token, verifica connessione.
- `references/selective-push.md` — comando push + flag.
- `references/section-naming.md` — convenzioni prefissi (con sezione funnel).
- `references/funnel-types.md` — struttura advertorial / listicle / quiz / other.
- `references/brand-identity-discovery.md` — come leggere colori/font da una PDP.
- `references/page-template-layout.md` — schema page template, chromeless vs theme.
- `references/funnel-image-specs.md` — dimensioni/ratio per ruolo sezione.
