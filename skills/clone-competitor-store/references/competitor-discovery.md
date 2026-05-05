# Competitor discovery — estrazione brand identity da URL

Riferimento per Fase 4 di `/clone-competitor-store`. Spiega come, dato un set di URL competitor, estrarre **palette colori, font, logo, tono di voce** in autonomia, senza chiedere all'utente di indicarli a parole.

## Strumenti a disposizione

- `WebFetch` — scarica HTML completo + segue eventuali redirect. Restituisce HTML + content delle prime 2-3 risorse statiche referenziate (CSS principale, immagini header).
- `Read` su screenshot inviati dall'utente (Claude legge le immagini).
- `Bash` con `curl` se WebFetch non basta (es. server bloccano user-agent generici).

## Estrazione step-by-step

### 1. WebFetch sull'URL competitor

```
WebFetch:
  url: <competitor.urls.funnel>  (o pdp / home)
  prompt: "Estrai tutti i colori HEX visibili, le font-family dichiarate inline o nel <link>, l'URL del logo nel header, e il tono dei testi heading/CTA principali. Restituisci come JSON strutturato."
```

Il prompt è ottimizzato per ridurre noise: il LLM interno di WebFetch fa già pre-processing.

Salva la risposta in `competitor.raw[<page>]`.

### 2. Estrazione palette

Tre fonti (in ordine di affidabilità):

#### a) CSS variabili custom (`--color-primary`, ecc.)

Cerca pattern `--color-*: #HEX` o `--c-*: #HEX` nel CSS. I temi moderni Shopify (Dawn, Horizon, Sense) li espongono in modo standard. Sono il segnale più pulito.

#### b) Inline style sui CTA

Identifica i `<a>` o `<button>` con `class*="cta"`, `class*="button-primary"`, `class*="add-to-cart"`. Estrai `background-color` o `background` dal `style=""` inline o dal CSS computato.

#### c) Frequenza colori sull'HTML completo

Conta tutte le occorrenze HEX (`#[0-9a-fA-F]{6}`) nel CSS della pagina. Esclude grayscale (`#000`, `#fff`, `#ccc`, `#333`, `#666`, `#999`, ecc.) e palette di librerie comuni (`#0d6efd` Bootstrap blue). Prendi i top 5-7 colori non-neutri.

#### d) Categorizzazione

Mappa i colori estratti in:
- **primary** → il colore CTA + heading dominante (di solito coincidono o sono contrastanti)
- **cta** → background CTA principale (button "Add to cart", "Get started")
- **heading** → colore degli H1/H2 (di solito #1A1A1A, #000, o brand-coloured)
- **body** → colore del paragrafo `<p>` (di solito #4A4A4A, #555, ecc.)
- **background** → page background (`<body>`, di solito `#fff`, cream `#FAF6F2`, ecc.)
- **accent** → colore ricorrente NON-CTA (gold detail, secondary buttons, badges)

### 3. Estrazione font

Tre fonti (in ordine di affidabilità):

#### a) Google Fonts `<link>`

```html
<link href="https://fonts.googleapis.com/css2?family=Playfair+Display:wght@700;900&family=Poppins:wght@300;400;500;700&display=swap" rel="stylesheet">
```

Estrai con regex i nomi font: `family=([A-Za-z+]+)`. Decodifica `+` → spazio. Risultato: `["Playfair Display", "Poppins"]`.

Per identificare quale è heading e quale body, cerca nel CSS quale è applicato a `h1, h2, h3` vs `body, p`.

#### b) Inline `font-family:`

`font-family: 'Playfair Display', serif;` — pattern identico, ma cerca nel CSS dei componenti.

#### c) `@font-face` custom

Se il brand usa un font custom (es. `MyBrandFont.woff2`), estrai il nome ma segnala all'utente: "Font custom rilevato (MyBrandFont). Useremo un fallback Google Fonts simile (es. Inter o Manrope) — confermami o suggerisci tu."

### 4. Estrazione logo

Cerca nell'HTML del header (primi 500 nodi del DOM, di solito):

```html
<header>
  <a href="/">
    <img src="..." class="header__logo" alt="Brand Name">
  </a>
</header>
```

Pattern di matching:
- `<img>` con `class*="logo"` (case-insensitive)
- `<img>` con `alt*="logo"` o `alt=<nome brand>`
- `<svg>` inline con `<title>Brand Name</title>` (logo SVG)

Salva URL assoluto (risolvi se relativo: prepend `https://<competitor-domain>`).

Se SVG inline: salva l'XML completo, l'utente potrà ricaricarlo come asset Shopify.

### 5. Estrazione tono di voce

Analizza i testi più grandi della pagina (H1, H2, CTA labels, hero subheadlines). Identifica:

- **Lunghezza media headline** (corte e dirette = punchy / lunghe e narrative = editoriale)
- **Persona** (you/your = diretto / we/our = brand-voice / he/she = storytelling terza persona)
- **Tono** (clinico/scientifico, casual/amichevole, lussuoso/elegante, urgente/scarcity)
- **Keyword ricorrenti** (3-5 parole-chiave dominanti — es. "natural", "fast", "proven", "guaranteed")

Salva come testo libero in `brand.tone` di 1-2 righe.

## Output finale (Fase 4.3)

Combina tutto in oggetto strutturato:

```json
{
  "palette": {
    "primary":    "#E36B6E",
    "cta":        "#1A1A1A",
    "heading":    "#1A1A1A",
    "body":       "#4A4A4A",
    "background": "#FAF6F2",
    "accent":     "#D4A574"
  },
  "fonts": {
    "heading": "Playfair Display, serif",
    "body":    "Poppins, sans-serif"
  },
  "logo_url_competitor": "https://competitor.com/cdn/shop/files/logo.png",
  "tone": "Editoriale, confidenziale, focus su risultati e ingredienti naturali. Headline lunghe (8-12 parole), persona diretta (you), keyword ricorrenti: natural, proven, results."
}
```

Mostra all'utente in formato leggibile (vedi SKILL.md Fase 4.3) e chiedi conferma con `AskUserQuestion`.

## Edge cases

### Sito JS-rendered (React, Next.js, Shopify Hydrogen)

WebFetch restituisce HTML scheletro senza contenuti. Sintomi:
- HTML <500KB con poche regole CSS
- `<div id="__next">` o `<div id="root">` vuoto
- `<noscript>` con messaggio "Please enable JavaScript"

Soluzione: chiedi all'utente di mandarti screenshot full-page + 2-3 inspect element del header (per vedere CSS custom). Lavora dagli screenshot per i colori, dal Computed CSS per i font.

### CDN dietro Cloudflare bot detection

WebFetch fallisce con 403 o "Just a moment". Soluzione: chiedi all'utente di salvare la pagina in HTML (browser → File → Save Page) e mandartelo come allegato.

### Multi-locale / regionalizzato

L'utente potrebbe mandarti URL della versione US, UK, IT del competitor. Confermache la versione che stai analizzando sia quella RIGHT del competitor (loro market di origine, di solito US per gli US drop). Le versioni regionali a volte hanno traduzioni meno curate o palette ridotte.

### Logo non in `<header>` ma come SVG nel footer

Cerca anche nel footer come fallback. Se trovi SVG inline e lo store ha branding minimal, il logo potrebbe essere SOLO testo (font + colore). Salva quello come "logo testo" e proponi all'utente di usare il nome brand come logo H1 in CSS, oppure caricare un PNG che faccia il match.

## Quando NON usare estrazione automatica

Se l'utente in Fase 4.3 dice che i colori estratti sono sbagliati 2 volte di seguito:
- Smetti di insistere sull'estrazione automatica
- Chiedi all'utente di mandarti una palette esplicita (lui può prenderla con un color picker dal suo browser)
- Rispetta i suoi valori senza override

## Note operative

- L'estrazione palette è **euristica**, non deterministica: due esecuzioni dello stesso URL possono dare top-7 leggermente diversi se il CSS è dinamico.
- Una pagina competitor sola non basta: usa più URL (funnel + PDP + home se disponibili) e fai un MERGE delle palette estratte. Colori che compaiono in 2+ pagine = high confidence.
- Tono di voce: lascia spazio per override utente. Spesso il tono di voce del competitor è da AGGIUSTARE per il mercato IT (più pacato, meno "clickbait" che funziona in US).
