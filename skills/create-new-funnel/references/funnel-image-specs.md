# Funnel image specs — dimensioni per tipo di sezione

Durante la Fase 8 di `/create-new-funnel`, per ogni sezione con immagini Claude genera un brief: cosa mostrare, dimensioni, placeholder. Questa è la fonte di verità.

## Principi generali

- **Tutte le immagini** caricate manualmente dall'utente in Shopify Admin → **Settings → Files** → Upload. L'utente copia l'URL CDN (`https://cdn.shopify.com/s/files/1/...`) e lo incolla in chat.
- **Formato**: `.webp` preferibile per foto, `.png` per grafica/trasparenza, `.svg` per icone/loghi.
- **Peso max**: 300 KB, ideale < 150 KB. Usa `squoosh.app` per comprimere.
- **Alt text**: sempre descrittivo, incluso.
- **Mobile-first**: quasi tutte le sezioni funnel sono lette da mobile (traffico ads paid). Ottimizza per viewport 375-414px.

## Specs per ruolo sezione

Mapping basato sul **ruolo nel suffix** (es. `aa-02-hero` → hero). Se una sezione ha un ruolo atipico, scegli la spec più vicina.

### `hero` / `01-hero` / `02-hero`
**Advertorial o generica**:
- **1 immagine** full-width lifestyle (mood emotivo, protagonista umano se advertorial).
- Ratio: 16:9 desktop, 4:5 o 1:1 per mobile-first chromeless.
- Dimensioni: 1600×900 (desktop) o 1200×1500 (mobile-first).
- Contenuto: foto lifestyle, protagonista che guarda verso la headline, no packshot.

**Listicle**:
- **1 immagine** editoriale o packshot su background colorato.
- Ratio: 16:9 o 4:3.
- Dimensioni: 1600×900 o 1200×900.

**Quiz**:
- **1 immagine** o illustrazione che rappresenta il "risultato ideale" del prodotto (pelle perfetta, benessere, ecc.).
- Ratio: 4:5 o 1:1.
- Dimensioni: 1000×1250 o 1000×1000.

### `problem` / `hook`
- **1 immagine** rappresentativa del dolore/problema.
- Ratio: 4:5 o 1:1.
- Dimensioni: 800×1000.
- Contenuto: foto close-up che amplifica il problema (es. zona borse occhiaie, pelle secca), tono naturale non drammatizzato.

### `story`
- **1-2 immagini** personali (before/after, foto candid del "protagonista").
- Ratio: varia, 4:5 o split 1:1.
- Dimensioni: 800×1000 o 600×600.
- Contenuto: se before/after, due foto con stesso framing (solo la condizione cambia).

### `solution` / `solution-intro`
- **1 immagine** packshot prodotto con mood pulito.
- Ratio: 1:1 o 4:5.
- Dimensioni: 1000×1000 o 1000×1250.
- Contenuto: prodotto centrato, background coerente con palette brand, no clutter.

### `how-it-works` / `benefits`
- **1 immagine principale** + **3-4 icon/illustrazioni** per i step/benefit.
- Principale: 1200×900, ratio 4:3.
- Icons: 200×200 SVG o PNG trasparente.
- Contenuto: principale = prodotto in uso o ingredienti; icons = pittogrammi coerenti (outline o filled, mai misti).

### `reason-XX` (listicle)
- **1 immagine per reason card**.
- Ratio: 4:5 o 4:3.
- Dimensioni: 600×750 o 800×600.
- Contenuto: visual che rappresenta il benefit specifico del punto. Non ripetere la stessa foto su più card.

### `social-proof` / `testimonials`
- **2-5 immagini** testimonianza (foto cliente o screenshot chat/recensione).
- Ratio: vario (1:1 per avatar, 4:5 per foto intera, 9:16 per screenshot chat).
- Dimensioni: 400×400 (avatar), 800×1000 (foto intera), 400×800 (screenshot).
- Contenuto mix: 50% facce reali / 50% screenshot recensioni autentiche.

### `comparison`
- **0-2 immagini** (spesso solo icone check/X) + eventualmente 2 packshot prodotti a confronto.
- Ratio packshot: 1:1.
- Dimensioni: 400×400.
- Contenuto: mini packshot del tuo prodotto vs competitor blurred/generico.

### `quiz-question`
- **1 illustrazione** per domanda (opzionale) + **1 icon per opzione** (consigliata).
- Illustrazione domanda: 800×400 orizzontale.
- Icon opzione: 120×120 SVG.
- Contenuto: stile illustrativo coerente per tutte le domande (non mischiare foto con illustrazioni).

### `result`
- **1 immagine** packshot del prodotto consigliato.
- Ratio: 1:1 o 4:5.
- Dimensioni: 1000×1000 o 1000×1250.
- Contenuto: prodotto con background coerente al risultato (es. per "pelle secca" → background idratante, acquoso).

### `cta-offer` / `offer`
- **1 immagine hero offerta**: packshot + badge "OFFERTA" o "-30%".
- Ratio: 16:9 o 4:3.
- Dimensioni: 1600×900 o 1200×900.
- Contenuto: prodotto centrale + elementi grafici offerta sovrapposti.

### `trust-badges`
- **3-6 icone** (garanzia, spedizione, pagamenti sicuri, made in Italy, ecc.).
- Dimensioni: 128×128 SVG o PNG trasparente.
- Contenuto: stile uniforme (tutti outline o tutti filled, stesso peso).

### `faq`
- **Nessuna immagine** tipicamente (solo accordion testo). Eventualmente 1 svg decorativo in header.

### `footer`
- **Logo brand** (SVG o PNG trasparente, max 200×60).
- Eventualmente icone social 32×32.

## Placeholder nei file `.liquid`

Quando la skill crea una sezione, lascia placeholder espliciti per le immagini:

```liquid
<img
  src="data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 800 1000'%3E%3Crect width='100%25' height='100%25' fill='%23F0E8DC'/%3E%3Ctext x='50%25' y='50%25' dominant-baseline='middle' text-anchor='middle' fill='%23888' font-family='sans-serif'%3EIMG 02-problem (800×1000)%3C/text%3E%3C/svg%3E"
  alt="IMG_PLACEHOLDER_02_problem"
  class="fnl-02__img"
  width="800" height="1000"
  loading="lazy">
```

Così nella pagina live appare un riquadro grigio con dimensioni, e l'utente capisce dove va quale immagine.

In alternativa, commento HTML:

```liquid
<!-- IMG_PLACEHOLDER_02_problem — 800×1000, 4:5, foto close-up dolore -->
```

## Brief template per l'utente (Fase 8)

Per ogni sezione:

```
Sezione: aa-02-hero (hero advertorial)
Immagini richieste: 1

  1. Ruolo: foto hero lifestyle — protagonista + emozione
     Ratio: 4:5 (mobile-first) o 16:9 (desktop)
     Dimensioni: 1200×1500 oppure 1600×900
     Peso max: 150 KB (.webp)
     Placeholder attuale: IMG_PLACEHOLDER_01_hero

Passi:
  1. Prepara l'immagine con le specifiche sopra.
  2. Caricala su Shopify Admin → Settings → Files.
  3. Copia l'URL CDN.
  4. Incollalo qui, così la skill fa il replace e push.
```
