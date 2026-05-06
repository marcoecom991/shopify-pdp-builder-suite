# PDP main section — configurazione above-the-fold

Riferimento per Fase 6.x.5 di `/clone-competitor-store` quando la pagina corrente è una **PDP**. Spiega come configurare la sezione **`product-information`** (nativa Shopify Horizon/Dawn) per matchare esattamente l'above-the-fold del competitor.

## Perché è critico

La sezione `product-information` è la sezione "main" della PDP: contiene gallery prodotto, titolo, prezzo, varianti, bottoni acquisto, descrizione. È **nativa del tema Shopify**, NON una sezione che noi costruiamo da zero. Però i suoi blocchi sub-componenti possono essere:

- **Inclusi/esclusi** (`disabled: true/false` nel template JSON)
- **Riordinati** (`block_order` array)
- **Stilizzati** (CSS custom inline o block dedicato)
- **Estesi con app blocks** (Katching Bundles, subscription pickers, recensioni)

Replicare il competitor 1:1 above-the-fold richiede di capire ESATTAMENTE quali sub-blocchi mostra/nasconde, in che ordine, con che styling.

Errore tipico: la skill costruisce solo i blocchi custom (bullets, trust banner, FAQ) e lascia la sezione `main` con la configurazione di default Shopify (quindi: prezzo grande, variant picker default, quantity selector visibile, buttoni accelerated checkout PayPal/Apple Pay visibili, add-to-cart con stile tema). Il competitor magari ha: niente prezzo above-the-fold, niente quantity selector, solo add-to-cart custom-styled, e un bundle widget Katching al posto del classic price/qty. Risultato: pagina visivamente sbagliata anche con tutto il resto perfetto.

## Procedura — Analisi above-the-fold

Per ogni PDP, **prima di scrivere il JSON template**, analizza con l'utente lo screenshot above-the-fold del competitor (i primi 800px della pagina su desktop) e identifica:

### 1. Elementi above-the-fold del competitor (checklist)

Per ognuno: **presente** (✓) / **assente** (✗) sul competitor. L'utente conferma con `AskUserQuestion`.

```
=== Above-the-fold competitor — analisi ===

Above title (sopra al nome prodotto):
  [ ] Pills / badges (es. "BESTSELLER", "ZERO ALLERGENI")
  [ ] Eyebrow text (es. "EDIZIONE LIMITATA")
  [ ] Logo brand sopra title

Title:
  [✓] Sempre presente

Sotto title:
  [ ] Star rating (stelle gialle)
  [ ] Review count (es. "4.8 / 5 — 2,431 recensioni")
  [ ] Subtitle / claim breve (es. "Crema autoabbronzante naturale")
  [ ] Bullet list breve (3-4 benefici)

Price:
  [ ] Presente above-the-fold (con discount/strike-through)
  [ ] Nascosto above-the-fold (mostrato solo dentro bundle widget)

Variant picker:
  [ ] Color/size/scent picker (se varianti)
  [ ] Bundle / Pack picker (3 opzioni: 1, 2, 3 pezzi con sconto progressivo)
      [ ] Se sì, gestito da app: Katching Bundles / Bundler / FastBundle / altro

Quantity selector:
  [ ] Visibile (input numerico +/-)
  [ ] Nascosto (non c'è proprio, l'utente compra solo "1 unit")

Buy buttons:
  [ ] Solo "Add to cart" (one button, custom styled)
  [ ] "Add to cart" + "Buy now" (due button)
  [ ] Accelerated checkout visibili: Apple Pay / Google Pay / PayPal
  [ ] Express checkout NASCOSTI

Sotto buttons:
  [ ] Stock indicator (es. "✓ In stock — 3 left")
  [ ] Urgency timer (countdown)
  [ ] Trust badges (sicurezza, spedizione, garanzia)
  [ ] Subscription option (Klaviyo Subscribe & Save, ReCharge, ecc.)
  [ ] Free gift / bonus
```

### 2. Mapping checklist → template JSON

In base alla checklist, configura il template `product.<slug>.json` settando i blocchi della sezione `main` (`product-information`):

#### Blocco `_product-details` (la colonna destra above-the-fold)

```json
"product-details": {
  "type": "_product-details",
  "static": true,
  "settings": { ... },
  "blocks": {
    "group_header": { ... },        // Title + pills + rating
    "buy_buttons": { ... },         // Bottoni acquisto
    // ... altri blocchi custom (bullets, trust banner)
  },
  "block_order": [
    "group_header",
    "desc_bullets",
    "atc_button_style",   // ← se vuoi style custom CTA, vedi sotto
    "buy_buttons",
    "trust_banner",
    // ...
  ]
}
```

#### Blocco `buy_buttons` — configurazione native

Questo blocco ospita i sub-blocchi `quantity`, `add-to-cart`, `accelerated-checkout`. Per ognuno usa `disabled: true/false`:

```json
"buy_buttons": {
  "type": "buy-buttons",
  "settings": {
    "stacking": true,
    "show_pickup_availability": false,
    "gift_card_form": true
  },
  "blocks": {
    "quantity": {
      "type": "quantity",
      "disabled": true,       // ← se competitor non mostra quantity
      "static": true,
      "settings": {},
      "blocks": {}
    },
    "add-to-cart": {
      "type": "add-to-cart",
      "static": true,
      "settings": {
        "style_class": "button"   // ← classe CSS per styling custom
      },
      "blocks": {}
    },
    "accelerated-checkout": {
      "type": "accelerated-checkout",
      "disabled": true,       // ← se competitor non mostra Apple Pay/Google Pay/PayPal
      "static": true,
      "settings": {},
      "blocks": {}
    }
  },
  "block_order": []
}
```

#### Custom CSS per il button add-to-cart

Se il competitor ha un button add-to-cart con:
- Colore di sfondo specifico (es. `#00C853` verde)
- Font specifico (es. Poppins 800 uppercase)
- Border-radius specifico (es. pill 999px)
- Dimensioni specifiche (es. min-height 56px)

Aggiungi un blocco `custom-liquid` chiamato `atc_button_style` nel `_product-details` PRIMA del blocco `buy_buttons` nel `block_order`. Esempio per CTA coral Poppins (ispirato al template autoabbronzante esistente):

```json
"atc_button_style": {
  "type": "custom-liquid",
  "name": "ATC native button style",
  "settings": {
    "custom_liquid": "<style>\n  @import url('https://fonts.googleapis.com/css2?family=Poppins:wght@700;800&display=swap');\n  .product-information .add-to-cart-button,\n  product-form .add-to-cart-button {\n    background: #00C853 !important;\n    color: #ffffff !important;\n    border: none !important;\n    border-radius: 999px !important;\n    font-family: 'Poppins', sans-serif !important;\n    font-weight: 800 !important;\n    font-size: 18px !important;\n    letter-spacing: 0.05em !important;\n    text-transform: uppercase !important;\n    min-height: 56px !important;\n    padding: 16px 20px !important;\n    width: 100% !important;\n  }\n  .product-information .add-to-cart-button:hover {\n    opacity: 0.9 !important;\n  }\n  .product-information .add-to-cart-button .add-to-cart-icon {\n    display: none !important;\n  }\n</style>"
  },
  "blocks": {}
}
```

I selettori CSS sono quelli del tema Horizon — puoi adattarli per altri temi:
- `.product-information .add-to-cart-button` (Horizon)
- `.product-form__submit` (Dawn)
- `button[type="submit"][name="add"]` (universale)

#### Slot per Katching Bundle / subscription / recensioni app

Se il competitor usa Katching Bundles (o altra app simile per multi-pack pricing), il bundle widget va nel `_product-details.blocks` con un block `@app`:

```json
"_product-details": {
  ...
  "blocks": {
    "group_header": { ... },
    "katching_bundle_slot": {
      "type": "@app",       // ← slot vuoto per app block
      "settings": {}
    },
    "buy_buttons": { ... },
    ...
  },
  "block_order": [
    "group_header",
    "desc_bullets",
    "katching_bundle_slot",  // ← posiziona dove competitor lo ha (es. tra title e add-to-cart)
    "atc_button_style",
    "buy_buttons",
    ...
  ]
}
```

⚠️ **Nota tecnica**: i blocchi `@app` non si possono pre-popolare con un'app specifica via JSON (non c'è un identificatore stabile per Katching/Bold/altre). Lo slot `@app` apparirà nel theme editor come "Add block → Apps" e l'utente deve selezionare l'app block giusto manualmente. Quando finisci la PDP, **istruisci l'utente** a:

```
=== Per attivare Katching Bundles ===

1. Online Store → Themes → <tema> → Customize.
2. Naviga alla PDP del nuovo prodotto.
3. Clicca il blocco "Add block" (icona +) nella sezione "Product details".
4. Tab "Apps" → seleziona "Katching Bundle Widget" (o nome simile).
5. Trascinalo nella posizione che vedi nel competitor (sopra/sotto add-to-cart).
6. Save.
```

#### Hide del prezzo native

Se il competitor non mostra il prezzo above-the-fold (es. perché il prezzo è dentro il bundle widget), non c'è una `disabled: true` per il blocco price (è dentro group_header). Soluzione: aggiungi un block `custom-liquid` con CSS che nasconde il prezzo native:

```json
"hide_native_price": {
  "type": "custom-liquid",
  "name": "Hide native price",
  "settings": {
    "custom_liquid": "<style>\n  .product-information .price,\n  .product-information [class*='price-block'] {\n    display: none !important;\n  }\n</style>"
  },
  "blocks": {}
}
```

E lo metti nel `block_order` del `_product-details`.

## Esempio completo — config per "PDP con Katching, no quantity, no express checkout, CTA verde Poppins"

```json
{
  "sections": {
    "main": {
      "type": "product-information",
      "blocks": {
        "media-gallery": { "type": "_product-media-gallery", "static": true, "settings": {...}, "blocks": {} },
        "product-details": {
          "type": "_product-details",
          "static": true,
          "settings": {...},
          "blocks": {
            "group_header": {...},
            "desc_bullets": {...},
            "katching_slot": { "type": "@app", "settings": {} },
            "atc_button_style": { "type": "custom-liquid", "settings": { "custom_liquid": "<style>...</style>" }, "blocks": {} },
            "buy_buttons": {
              "type": "buy-buttons",
              "settings": { "stacking": true, "show_pickup_availability": false, "gift_card_form": true },
              "blocks": {
                "quantity": { "type": "quantity", "disabled": true, "static": true },
                "add-to-cart": { "type": "add-to-cart", "static": true, "settings": { "style_class": "button" } },
                "accelerated-checkout": { "type": "accelerated-checkout", "disabled": true, "static": true }
              },
              "block_order": []
            },
            "trust_banner": {...}
          },
          "block_order": [
            "group_header",
            "desc_bullets",
            "katching_slot",
            "atc_button_style",
            "buy_buttons",
            "trust_banner"
          ]
        }
      },
      "settings": {...}
    },
    "pdp-02": {...},
    "pdp-03": {...}
    // ... altre sezioni custom della PDP
  },
  "order": [
    "main",
    "pdp-02",
    ...
  ]
}
```

## Self-check prima del push

Per ogni PDP costruita, verifica:

- [ ] La checklist "Above-the-fold competitor" è stata fatta con l'utente
- [ ] `quantity.disabled` corretto (true se non visibile su competitor)
- [ ] `accelerated-checkout.disabled` corretto (true se nessun Apple Pay/Google Pay/PayPal su competitor)
- [ ] Block `atc_button_style` presente se CTA ha styling diverso dal default
- [ ] Block `@app` slot presente se competitor usa Katching/Bundler/altre app
- [ ] Block `hide_native_price` presente se competitor nasconde il prezzo above-the-fold
- [ ] `block_order` rispetta l'ordine visivo del competitor (es. bundle widget sopra/sotto add-to-cart come nel competitor)
- [ ] Istruzioni utente per attivare app blocks fornite a fine build

## Quando il competitor NON è Shopify

Se il competitor è WordPress/WooCommerce, Magento, altro:
- Non c'è un mapping diretto blocchi → templates JSON
- Però la logica è identica: identifica gli elementi above-the-fold e replica solo quelli, disabilita quello che non serve, applica style CSS custom al CTA
- L'utente farà l'integrazione di app Shopify (Katching) come setup post-clone, non c'è equivalente diretto su WP

## Sezioni reviews — pattern doppio (carousel + grid)

Le PDP DTC hanno tipicamente **DUE blocchi review** distinti, separati da altre sezioni intermedie. Sono visivamente diversi e servono a scopi diversi:

### Sezione A — Carousel testimonial (highlight, posizionato sopra)

Posizione tipica: subito dopo benefits / how-it-works, prima delle reviews complete. Funzione: prova sociale "narrative" che ricorda al lettore "altri come te lo amano già".

Caratteristiche visive:
- Carousel orizzontale (3-6 cards visibili su desktop, 1-2 su mobile)
- Header con highlight giallo o accent ("LOVED BY 50,000+ CUSTOMERS")
- Pills "Verified" / "Verified Buyer" su ogni card
- Card content: headline ("Game changer!" / "Best purchase ever") + paragrafo medio + nome cliente + foto cliente

File: `sections/<prefix>-pdp-NN-testimonial-carousel.liquid`

Schema:
- `settings`: section heading, subheading, badge text ("LOVED BY..."), CTA opzionale
- `blocks` array di tipo `testimonial`:
  - `image_picker` foto cliente
  - `text` headline ("Best ever!")
  - `richtext` body (paragrafo)
  - `text` author name
  - `text` author location/credential
  - `checkbox` verified

⚠️ **Le testimonianze sono L4** (placeholder). Inserisci 3-5 placeholder nel default `blocks` lasciando struttura ma con testo `[Testimonial — sostituire con recensione tua cliente]`.

### Sezione B — Reviews grid (con breakdown stelle)

Posizione tipica: bottom della pagina prima del footer, oppure penultima sezione prima di "Closing CTA". Funzione: prova sociale "data-driven" — più recensioni totali, breakdown 5★/4★/.../1★, sort/filter, "view more".

Caratteristiche visive:
- Header con avg rating big (es. "4.8 / 5") + "Based on 2,431 reviews"
- 5 bar orizzontali per rating breakdown (5★ ████████░ 1,832  /  4★ ████░ 421  /  etc.)
- Sort dropdown (Most recent / Most helpful / Highest / Lowest)
- Filter pills (Verified / With photos / Specific traits)
- Grid di review cards (2 columns desktop, 1 mobile) con: rating stars, date, author, title, body, photo opzionale, "X people found this helpful"
- "Load more" button al fondo

File: `sections/<prefix>-pdp-NN-reviews-grid.liquid`

Schema:
- `settings`: section heading ("Customer Reviews"), avg rating, total count, sort options labels, filter labels
- `blocks` array di tipo `review_breakdown` (5 blocks fissi per stelle 1-5):
  - `text` star count ("5 stars")
  - `text` percentage / count
- `blocks` array di tipo `review_card`:
  - `range` rating 1-5
  - `text` author + date
  - `text` title
  - `richtext` body
  - `image_picker` photo (opz.)
  - `checkbox` verified
  - `text` "X people found helpful"

⚠️ **App-driven alternative**: la maggior parte degli store usa app per reviews (Judge.me, Loox, Yotpo, Stamped). In quel caso la sezione `<prefix>-pdp-NN-reviews-grid.liquid` può essere semplicemente un wrapper con header custom + slot `@app` dove l'utente attiva l'app review block dal theme editor:

```json
"settings": [
  { "type": "text", "id": "section_heading", "label": "Heading sezione", "default": "Customer Reviews" }
],
"blocks": [
  { "type": "@app" }
]
```

Istruzione utente a fine build:
```
=== Per attivare le reviews ===

1. Theme editor → PDP → Sezione "Customer Reviews" → Add block (icona +)
2. Tab "Apps" → seleziona "Judge.me Reviews" / "Loox Reviews" / "Yotpo" 
   (in base all'app installata)
3. Save.

Il block app gestirà avg rating + breakdown + grid + sort/filter automaticamente.
```

### Differenze chiave (per distinguerle)

| Caratteristica | Carousel testimonial | Reviews grid |
|----------------|---------------------|--------------|
| Posizione PDP | Above (vicino a benefits) | Below (verso il footer) |
| Layout | Orizzontale slider | Vertical grid |
| Numero cards visibili | 3-6 (curated) | 10-50+ (con load more) |
| Source | Curated dal merchant | UGC reale via app |
| Ruolo conversione | Hook emotional | Validation razionale |
| Scope clone skill | Sezione custom liquid | Wrapper + slot @app |

**Errore tipico da NON fare**: codificare un'unica sezione "reviews" che mischia carousel + grid + breakdown. Le PDP DTC le tengono separate per ragioni di UX (la carousel high-impact va sopra come hook, la grid completa va sotto come deep dive).

---

## Note operative

- **Theme variations**: Horizon usa `disabled: true` per native blocks. Dawn più vecchio usa `_product-form-buy-buttons-show_dynamic_checkout: false`. Verifica nel tema target qual è il pattern corretto.
- **Mobile vs desktop above-the-fold**: lo stesso elemento può essere visibile mobile e nascosto desktop o viceversa. La sezione main su Horizon ha sub-settings per mobile-specific behavior — sfrutta media query CSS se serve.
- **Animazioni del bundle widget**: Katching ha animazioni di hover/select. Lascia che le gestisca lui — non sovrascriverle con CSS custom.
- **Subscription**: se competitor offre subscription (es. ReCharge, Bold, Skio), aggiungi un altro `@app` slot dedicato. Spesso è una checkbox "Subscribe & Save 10%" sopra/sotto add-to-cart.
