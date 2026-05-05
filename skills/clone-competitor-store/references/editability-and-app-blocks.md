# Editability + App blocks — regola d'oro

Riferimento per Fase 6.x.5 di `/clone-competitor-store`. Questa è la **hard rule più importante** della skill. Ogni sezione costruita deve essere **completamente editabile dal Shopify theme editor** senza che l'utente debba toccare il liquid.

In più, le sezioni PDP devono **supportare app blocks** per ospitare Katching Bundles, subscription pickers, app review (Judge.me, Loox, ecc.).

## Perché è critico

L'utente userà il theme editor per:
1. Cambiare i testi delle sezioni (correggere errori di traduzione, A/B test claim, aggiornare offerte stagionali)
2. Caricare le immagini reali dopo che la skill ha pushato il template (la skill mette placeholder)
3. Cambiare colori dei CTA quando vuole testare diverse varianti
4. Aggiungere blocchi di app (Katching, subscription, recensioni) negli slot che la skill ha predisposto
5. Riordinare elementi ripetibili (FAQ, testimonial, benefit cards) trascinandoli nella sidebar

Se la skill produce sezioni con testo/immagini/link hardcoded nel markup, l'utente è costretto a chiamarci ogni volta che vuole cambiare una virgola → fallisce lo scopo della skill.

## Regole operative

### 1. Testi → settings o blocks

Ogni testo visibile nel markup va dichiarato come `setting` o `block` nello schema, mai inserito direttamente nel HTML.

**ROTTO ❌**
```liquid
<h1>La crema autoabbronzante che cambia tutto</h1>
<p>Risultati in 30 minuti, senza arancio.</p>
```

**CORRETTO ✓**
```liquid
<h1>{{ section.settings.headline }}</h1>
<p>{{ section.settings.subheadline }}</p>
```

```json
{
  "settings": [
    { "type": "text",     "id": "headline",    "label": "Titolo",       "default": "La crema autoabbronzante che cambia tutto" },
    { "type": "richtext", "id": "subheadline", "label": "Sottotitolo",  "default": "<p>Risultati in 30 minuti, senza arancio.</p>" }
  ]
}
```

**Mapping tipi → setting**:
- Heading singola riga → `text`
- Sottotitolo / paragrafo breve → `textarea`
- Paragrafo con bold/italic/link → `richtext` (l'editor mostra WYSIWYG)
- Singola label corta → `text`

### 2. Immagini → image_picker

MAI `<img src="https://cdn.shopify.com/...">` hardcoded. Sempre `image_picker` con default vuoto.

**ROTTO ❌**
```liquid
<img src="https://cdn.shopify.com/s/files/1/0123/4567/files/hero.jpg" alt="Hero">
```

**CORRETTO ✓**
```liquid
{% if section.settings.hero_image %}
  <img src="{{ section.settings.hero_image | image_url: width: 1600 }}"
       alt="{{ section.settings.hero_image.alt | escape }}"
       width="{{ section.settings.hero_image.width }}"
       height="{{ section.settings.hero_image.height }}"
       loading="lazy">
{% else %}
  <div class="img-placeholder">[Hero desktop — consigliato 1920×800]</div>
{% endif %}
```

```json
{
  "settings": [
    { "type": "image_picker", "id": "hero_image", "label": "Immagine hero" }
  ]
}
```

L'utente carica l'immagine dal theme editor cliccando il campo "Immagine hero" → "Select image" → upload o select da Files.

**Note tecniche**:
- Usa `image_url: width: <W>` per generare URL responsive — Shopify CDN ottimizza automaticamente.
- Aggiungi `srcset` per responsive serving (esempio in `visual-replication.md`).
- Sempre `alt` (anche vuoto se decorativo) e `loading="lazy"` salvo per immagini above-the-fold.

### 3. Link / CTA → coppia url + text

Ogni bottone CTA è una coppia di `settings`:
- `type: "url"` per il link di destinazione
- `type: "text"` per il label visibile

**ROTTO ❌**
```liquid
<a href="https://glowria-beauty.com/products/crema" class="cta">Aggiungi al carrello</a>
```

**CORRETTO ✓**
```liquid
<a href="{{ section.settings.cta_url | default: '/' }}" class="{{ section.id }}-cta">
  {{ section.settings.cta_label }}
</a>
```

```json
{
  "settings": [
    { "type": "url",  "id": "cta_url",   "label": "Link CTA",   "default": "" },
    { "type": "text", "id": "cta_label", "label": "Testo CTA",  "default": "Aggiungi al carrello" }
  ]
}
```

**Default per i CTA**: se la sezione è costruita per un funnel con PDP target già nota (Fase 6 PDP costruita prima del funnel), il default `cta_url` è la URL della PDP. Altrimenti vuoto e l'utente la imposta dall'editor.

### 4. Colori variabili → setting color

Per colori che potrebbero cambiare (CTA background, accent, ecc.) usa `type: "color"`.

```liquid
<style>
  .{{ section.id }}-cta {
    background: {{ section.settings.cta_bg }};
    color: {{ section.settings.cta_text }};
  }
</style>
```

```json
{
  "settings": [
    { "type": "color", "id": "cta_bg",   "label": "Sfondo CTA", "default": "#E36B6E" },
    { "type": "color", "id": "cta_text", "label": "Testo CTA",  "default": "#FFFFFF" }
  ]
}
```

Default = valori di `brand.palette.*` di Fase 4.

I colori HARD del layout (es. background sezione che è sempre `#fff`) possono restare hardcoded nel CSS — non serve esporli all'editor se sono parte del design fisso.

### 5. Liste ripetibili → blocks

Per elementi che si ripetono (FAQ items, testimonial, benefit cards, bullet list, slide carousel), usa il pattern `blocks`.

**ROTTO ❌**
```liquid
<div class="faq">
  <div class="faq-item">
    <h3>Funziona davvero?</h3>
    <p>Sì, certificato da X test clinici.</p>
  </div>
  <div class="faq-item">
    <h3>In quanto tempo vedo risultati?</h3>
    <p>4-6 settimane di uso costante.</p>
  </div>
  <!-- ... -->
</div>
```

**CORRETTO ✓**
```liquid
<div class="faq">
  {% for block in section.blocks %}
    {% if block.type == 'faq_item' %}
      <div class="faq-item" {{ block.shopify_attributes }}>
        <h3>{{ block.settings.question }}</h3>
        <div>{{ block.settings.answer }}</div>
      </div>
    {% endif %}
  {% endfor %}
</div>
```

```json
{
  "blocks": [
    {
      "type": "faq_item",
      "name": "Domanda FAQ",
      "settings": [
        { "type": "text",     "id": "question", "label": "Domanda",  "default": "..." },
        { "type": "richtext", "id": "answer",   "label": "Risposta", "default": "<p>...</p>" }
      ]
    }
  ],
  "max_blocks": 12,
  "presets": [
    {
      "name": "FAQ",
      "blocks": [
        { "type": "faq_item" },
        { "type": "faq_item" },
        { "type": "faq_item" }
      ]
    }
  ]
}
```

**Vantaggi**:
- L'utente trascina/ordina le FAQ dalla sidebar
- L'utente aggiunge nuove FAQ senza toccare il file
- Cancellare una FAQ è un click

**Quando NON usare blocks**: per elementi UNICI/strutturali (un solo hero, un solo CTA principale, un solo paragrafo intro). Lì usa `settings`.

### 6. App blocks → critico per PDP (Katching Bundles, subscription, reviews)

Le sezioni di tipo PDP che ospitano "logica prodotto" (main, buy-buttons area, post-add-to-cart) DEVONO dichiarare `{"type": "@app"}` nei blocks per permettere all'utente di **inserire blocchi di app** dal theme editor senza modificare il liquid.

```json
{
  "blocks": [
    { "type": "@app" },
    {
      "type": "custom_bullet",
      "name": "Bullet point",
      "settings": [...]
    }
  ]
}
```

L'`@app` è un wildcard: il theme editor mostra all'utente l'opzione "+ Add block" → tab "Apps" → lista di tutti i blocchi delle app installate sullo store (Katching, Bold Subscriptions, Judge.me, Loox, ecc.).

**Senza `@app` non funziona Katching Bundles**: Katching pubblica i suoi block via App Embed Block API standard, ma se la sezione PDP non li accetta (`@app` mancante), l'utente non può inserirli e deve modificare manualmente il liquid.

**Sezioni dove SERVE `@app`**:
- `<store-slug>-pdp-02-main` (la sezione principale del prodotto)
- `<store-slug>-pdp-04-bundle` (se hai una sezione dedicata bundle)
- `<store-slug>-pdp-09-reviews` (per ospitare app review)
- `<store-slug>-pdp-10-closing-cta` (per ospitare upsell apps)

**Sezioni dove NON serve `@app`** (ma puoi metterlo lo stesso, non fa danno):
- Sezioni puramente di contenuto (hero, ingredients, how-it-works, faq, footer)
- Sezioni del funnel/home (di solito le app PDP non si attaccano qui)

### 7. Render dei blocks con `block.shopify_attributes`

Quando renderizzi i `blocks` nel markup, AGGIUNGI SEMPRE `{{ block.shopify_attributes }}` sull'elemento root del block:

```liquid
<div class="{{ section.id }}-faq-item" {{ block.shopify_attributes }}>
  ...
</div>
```

Questo expose hooks per il theme editor (drag & drop, click-to-edit). Senza `shopify_attributes` il block è renderizzato ma non interagibile dall'editor.

### 8. Default sensati su tutto

Ogni `setting` e `block` deve avere un `default` popolato:
- Testi → testo IT tradotto dal competitor
- Colori → da `brand.palette.*`
- URL → vuoto o (se applicabile) URL della PDP creata
- Image_picker → vuoto (l'utente carica)

**Motivo**: la pagina live deve mostrare contenuto reale subito dopo il push, non placeholder vuoti `[empty]`. Se l'utente apre la pagina prima di aver toccato il theme editor, deve vedere comunque qualcosa di sensato.

## Esempio completo — sezione Hero advertorial

```liquid
{%- liquid
  assign cta_url = section.settings.cta_url
  if cta_url == blank
    assign cta_url = '/'
  endif
-%}

<style>
  @import url('https://fonts.googleapis.com/css2?family=Playfair+Display:wght@900&family=Poppins:wght@400;600&display=swap');
  
  .{{ section.id }}-hero {
    padding: 48px 16px;
    background: {{ section.settings.bg_color }};
    text-align: center;
  }
  .{{ section.id }}-hero__headline {
    font-family: 'Playfair Display', serif;
    font-size: clamp(28px, 6vw, 48px);
    font-weight: 900;
    color: {{ section.settings.heading_color }};
    line-height: 1.1;
    margin: 0 0 16px;
  }
  .{{ section.id }}-hero__sub {
    font-family: 'Poppins', sans-serif;
    font-size: clamp(15px, 2.5vw, 18px);
    color: {{ section.settings.body_color }};
    line-height: 1.5;
    margin: 0 0 32px;
  }
  .{{ section.id }}-hero__cta {
    display: inline-block;
    background: {{ section.settings.cta_bg }};
    color: {{ section.settings.cta_text }};
    padding: 16px 40px;
    border-radius: 999px;
    font-weight: 700;
    font-size: 16px;
    text-decoration: none;
    transition: opacity 0.2s;
  }
  .{{ section.id }}-hero__cta:hover { opacity: 0.9; }
  .{{ section.id }}-hero__img {
    margin-top: 32px;
    max-width: 100%;
    height: auto;
  }
  
  @media (min-width: 750px) {
    .{{ section.id }}-hero { padding: 96px 32px; }
  }
</style>

<section class="{{ section.id }}-hero">
  <h1 class="{{ section.id }}-hero__headline">{{ section.settings.headline }}</h1>
  <div class="{{ section.id }}-hero__sub">{{ section.settings.subheadline }}</div>
  <a class="{{ section.id }}-hero__cta" href="{{ cta_url }}">{{ section.settings.cta_label }}</a>
  
  {% if section.settings.hero_image %}
    <img class="{{ section.id }}-hero__img"
         src="{{ section.settings.hero_image | image_url: width: 1600 }}"
         srcset="{{ section.settings.hero_image | image_url: width: 750 }} 750w,
                 {{ section.settings.hero_image | image_url: width: 1600 }} 1600w"
         sizes="100vw"
         alt="{{ section.settings.hero_image.alt | escape }}"
         loading="eager"
         width="{{ section.settings.hero_image.width }}"
         height="{{ section.settings.hero_image.height }}">
  {% endif %}
</section>

{% schema %}
{
  "name": "Advertorial Hero",
  "tag": "section",
  "class": "glowria-adv-02",
  "settings": [
    { "type": "text",         "id": "headline",       "label": "Titolo principale", "default": "La crema autoabbronzante che cambia tutto" },
    { "type": "richtext",     "id": "subheadline",    "label": "Sottotitolo",       "default": "<p>Risultati in 30 minuti, senza arancio. Adatta a tutte le tonalità.</p>" },
    { "type": "image_picker", "id": "hero_image",     "label": "Immagine hero" },
    { "type": "url",          "id": "cta_url",        "label": "Link CTA",          "default": "" },
    { "type": "text",         "id": "cta_label",      "label": "Testo CTA",         "default": "Scopri di più" },
    { "type": "header",       "content": "Colori" },
    { "type": "color",        "id": "bg_color",       "label": "Sfondo sezione",    "default": "#FAF6F2" },
    { "type": "color",        "id": "heading_color",  "label": "Colore titolo",     "default": "#1A1A1A" },
    { "type": "color",        "id": "body_color",     "label": "Colore testo",      "default": "#4A4A4A" },
    { "type": "color",        "id": "cta_bg",         "label": "Sfondo CTA",        "default": "#E36B6E" },
    { "type": "color",        "id": "cta_text",       "label": "Testo CTA",         "default": "#FFFFFF" }
  ],
  "presets": [
    { "name": "Advertorial Hero" }
  ]
}
{% endschema %}
```

## Self-check prima del push

Per ogni sezione, prima di pushare:

- [ ] Tutti i testi visibili sono in `settings` o `blocks`, mai hardcoded
- [ ] Tutte le `<img>` usano `image_picker` (no `cdn-url` hardcoded)
- [ ] Tutti i `<a href="...">` di contenuto usano `settings.url`
- [ ] Tutti i colori "variabili" sono in `setting type:"color"`, brand.palette default
- [ ] Liste ripetibili usano `blocks`, non duplicazione hardcoded
- [ ] Sezioni PDP main/bundle/reviews/closing hanno `{"type": "@app"}` nei blocks
- [ ] Ogni block render ha `{{ block.shopify_attributes }}` sull'elemento root
- [ ] Ogni `setting` e `block` ha un `default` sensato
- [ ] `presets[]` definito per permettere all'utente di "Add section" dal theme editor

## Quando deviare

L'unica eccezione: elementi puramente strutturali e immutabili (es. wrapper `<div class="container">`, padding del max-width, regole CSS scoped del file). Quelli possono restare hardcoded perché non hanno significato di "contenuto modificabile".

Per il resto: zero deviazioni. La hard rule è hard.
