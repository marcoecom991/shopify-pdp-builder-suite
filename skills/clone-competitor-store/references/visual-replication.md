# Visual replication — ricostruire il design competitor 1:1

Riferimento per Fase 6.x.5 di `/clone-competitor-store`. Spiega come, dato uno screenshot full-page mobile + desktop + URL competitor, scrivere HTML + CSS che lo replicano fedelmente.

## Obiettivo

Output finale: una sezione liquid del nostro tema che, quando renderizzata sul nostro store, è **visivamente indistinguibile** dalla sezione corrispondente del competitor. Stessi colori, font, spacing, layout, micro-interazioni. Stessi testi (in lingua originale prima, IT dopo).

Non è "ispirato a" — è "uguale a". L'utente avrà investito tempo nello scegliere quel competitor perché funziona; il nostro lavoro è non perdere niente del valore di quella scelta.

## 🚫 Regola anti-hallucination (la più importante)

**Replica SOLO quello che l'utente fornisce.** Niente sezioni "che ci starebbero bene", niente bullet "che potrebbero esserci", niente claim "tipici di un advertorial". Se l'utente fornisce 7 sezioni di testi, lo schema ha 7 sezioni. Se per una sezione fornisce 3 bullet, lo schema ha 3 bullet. Se il titolo che fornisce è "The cream that finally works", lo schema mostra "The cream that finally works".

**Divisione delle responsabilità**:
- **L'utente**: fornisce i testi di ogni sezione (HTML salvato o copy/paste manuale dal sito competitor) in Fase 6.x.5.a.
- **Claude**: costruisce HTML+CSS della sezione che riproduce il layout dello screenshot, e mette i testi forniti dall'utente nei `default` dello schema **senza alterarli**.

Conseguenze pratiche:

- **Mai compilare i `default` con testi inventati.** I `default` sono ESATTAMENTE quello che l'utente ha fornito in 6.x.5.a, parola per parola.
- **Mai usare gli screenshot come fonte testuale.** Gli screenshot servono per il layout (capire posizioni, ordine, dimensioni). Per i testi → chiedi all'utente.
- **Mai aggiungere sezioni "perché aiuterebbe la conversione".** Stiamo clonando, non ottimizzando. L'ottimizzazione viene dopo, in iterazioni separate.
- **Mai parafrasare** i testi forniti. Se l'utente vuole un wording diverso, lo cambia lui in chat o in step 2 (localizzazione IT).
- **Se ti manca un testo** per una sezione: FERMATI e chiedilo all'utente. Non interpretare. Non immaginare.

Sintomo che la regola non sta funzionando: i `default` dello schema contengono frasi tipo "Discover the secret" / "Unlock your potential" / "Transform your skin today" che sono chiaramente generiche e non sono nei testi che l'utente ha fornito. Se vedi queste frasi nei tuoi `default`, **stop e torna a chiedere i testi all'utente**.

## Materiali in input

Per ogni sezione, dovresti avere a disposizione:

1. **Screenshot full-page DESKTOP** del competitor (1440px wide tipico). Lo userai come reference per layout principale.
2. **Screenshot full-page MOBILE** del competitor (375px wide tipico). Lo userai per breakpoint mobile.
3. **URL della pagina competitor**. Lo userai per inspect del CSS via WebFetch.
4. **Brand identity** (`brand.*` da Fase 4): palette, font, tono.

## Strategia di replica per ogni sezione

### 1. Identifica il layout di base

Guarda lo screenshot desktop. Cerca pattern noti:

- **Hero centered**: testo + CTA centrato verticalmente, immagine sfondo o vuoto
- **Hero split (50/50)**: testo a sinistra, immagine a destra (o invertito)
- **Hero asymmetric**: testo dominante + immagine piccola laterale
- **Bento grid**: griglia 2×2 / 3×2 di card
- **Stripe**: full-width banner orizzontale (announcement bar, USP strip)
- **Carousel**: slider orizzontale con cards/recensioni
- **Stack list**: items verticali con icona/numero a sinistra (FAQ, benefits, steps)
- **Comparison table**: due colonne fianco a fianco con tick/cross
- **Z-pattern**: alternanza testo-sx/img-dx → testo-dx/img-sx (long-form sections)

Mappa il layout in una struttura HTML minimale.

### 2. Misura spacing e proporzioni dallo screenshot

Apri lo screenshot in modalità lettura visiva e stima:

- **Container max-width** (di solito 1200px o 1440px su desktop)
- **Padding interno verticale** (32px / 64px / 96px tipici)
- **Padding orizzontale** (16px mobile, 24px tablet, 48px desktop tipici)
- **Gap tra elementi** (8px / 16px / 24px / 48px)
- **Border-radius** (0px squared / 4px subtle / 8px medium / 16px rounded / 999px pill)
- **Font-size headlines** (rapporto rispetto al body — di solito H1 ≈ 2.5-4× body)

Se non sei sicuro: assumi valori di base (24px padding, 16px gap, 8px border-radius) e aggiusta a vista in iterazione successive.

### 3. Implementa mobile-first

Parti SEMPRE dallo screenshot mobile:

```css
/* Default: mobile (375px - 749px) */
.{prefix}-{nn}-{role} {
  padding: 32px 16px;
  /* ... */
}

@media (min-width: 750px) {
  /* Tablet */
  .{prefix}-{nn}-{role} {
    padding: 48px 32px;
  }
}

@media (min-width: 1200px) {
  /* Desktop */
  .{prefix}-{nn}-{role} {
    padding: 96px 64px;
  }
}
```

I breakpoint 750/1200 sono lo standard Shopify (Dawn, Horizon). Niente `768`/`1024`/`1440` se non strettamente necessari.

### 4. Usa CSS scoped al prefisso file

Tutte le classi dentro `sections/<prefix>-<NN>-<role>.liquid` iniziano con `.<prefix>-<NN>-<role>__*` o `.<prefix>-<NN>-*`. Così non ci sono mai conflitti tra sezioni diverse.

Esempio per `glowria-adv-02-hero.liquid`:
```css
.glowria-adv-02-hero { /* container sezione */ }
.glowria-adv-02-hero__headline { /* H1 */ }
.glowria-adv-02-hero__sub { /* H2 */ }
.glowria-adv-02-hero__cta { /* button */ }
```

(Convenzione BEM-style con suffix `__element`. Modificatori `--variant` se servono.)

### 5. Iniezione brand

Tutti i valori di colore nei CSS vengono dal `brand.palette.*` di Fase 4, **come default** dei `setting` schema:

```liquid
<style>
  .{{ section.id }}-cta {
    background: {{ section.settings.cta_bg }};
    color: {{ section.settings.cta_text }};
  }
</style>

{% schema %}
{
  "settings": [
    { "type": "color", "id": "cta_bg",   "label": "Colore CTA",       "default": "#E36B6E" },
    { "type": "color", "id": "cta_text", "label": "Colore testo CTA", "default": "#FFFFFF" }
  ]
}
{% endschema %}
```

Dove `#E36B6E` è il valore estratto dal competitor in Fase 4 (`brand.palette.primary`).

Stesso pattern per `font-family`:
```liquid
<style>
  .{{ section.id }}-headline {
    font-family: {{ section.settings.heading_font | default: 'Playfair Display, serif' }};
    font-size: clamp(28px, 6vw, 48px);
    font-weight: 900;
  }
</style>
```

### 6. Tipografia: clamp + line-height costante

Per font-size responsive senza media query esplosive:
```css
font-size: clamp(MIN, vw-based, MAX);
```

Esempi:
- H1 hero: `clamp(28px, 6vw, 56px)`
- H2 section: `clamp(22px, 4vw, 36px)`
- Body: `clamp(15px, 2.5vw, 17px)`

Line-height:
- Heading: `1.1` o `1.2`
- Body: `1.5` o `1.6`
- Small text: `1.4`

### 7. Immagini con `image_picker` e SVG placeholder

Le immagini del competitor non vanno copiate sul nostro store (issue copyright + CDN invalida). Lascia placeholder che l'utente popola dal theme editor:

```liquid
{% if section.settings.hero_image %}
  <img class="{{ section.id }}-img"
       src="{{ section.settings.hero_image | image_url: width: 1600 }}"
       srcset="{{ section.settings.hero_image | image_url: width: 750 }} 750w,
               {{ section.settings.hero_image | image_url: width: 1100 }} 1100w,
               {{ section.settings.hero_image | image_url: width: 1600 }} 1600w"
       sizes="(min-width: 1200px) 50vw, 100vw"
       alt="{{ section.settings.hero_image.alt | escape }}"
       loading="lazy"
       width="{{ section.settings.hero_image.width }}"
       height="{{ section.settings.hero_image.height }}">
{% else %}
  <div class="{{ section.id }}-img-placeholder">
    <span>Hero desktop — consigliato 1920×800 / mobile 750×900</span>
  </div>
{% endif %}
```

CSS placeholder:
```css
.{{ section.id }}-img-placeholder {
  aspect-ratio: 16/9;
  background: #f0f0f0;
  border: 2px dashed #999;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 13px;
  color: #666;
  text-align: center;
  padding: 20px;
}
```

### 8. Animation/transition leggere

Se il competitor ha animazioni (fade-in, slide, parallax), replicale in modo conservativo:
- `transition: all 0.3s ease` standard
- `@keyframes` solo per effetti che impattano funzionalmente (shimmer su offer badge, pulse su CTA limited time)
- No GSAP, no AOS library — solo CSS

Per scroll-triggered animations evita per default (impatto LCP). Se l'utente le chiede esplicitamente, usa `IntersectionObserver` con classe `.is-visible` aggiunta da JS minimal inline.

### 9. Accessibilità basics

Non sacrificare per fedeltà visuale:
- Sempre `alt` su `<img>` (vuoto se decorativo)
- `aria-label` su CTA che hanno solo icona
- Contrasto minimo AA (`#000` su sfondo coral chiaro è ok, gray claro su gray scuro NO)
- `<button>` per click reali (non `<div onclick>`)

## Pattern ricorrenti — snippet riusabili

### Container max-width responsive

```css
.{{ section.id }}-container {
  width: 100%;
  max-width: 1200px;
  margin-inline: auto;
  padding-inline: 16px;
}
@media (min-width: 750px) {
  .{{ section.id }}-container { padding-inline: 32px; }
}
@media (min-width: 1200px) {
  .{{ section.id }}-container { padding-inline: 48px; }
}
```

### CTA pill button

```css
.{{ section.id }}-cta {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  background: {{ section.settings.cta_bg }};
  color: {{ section.settings.cta_text }};
  padding: 16px 32px;
  border-radius: 999px;
  font-weight: 700;
  font-size: 16px;
  text-decoration: none;
  border: none;
  cursor: pointer;
  transition: opacity 0.2s ease;
}
.{{ section.id }}-cta:hover { opacity: 0.9; }
```

### Two-column responsive (50/50 desktop, stack mobile)

```css
.{{ section.id }}-cols {
  display: grid;
  grid-template-columns: 1fr;
  gap: 32px;
}
@media (min-width: 750px) {
  .{{ section.id }}-cols {
    grid-template-columns: 1fr 1fr;
    gap: 48px;
    align-items: center;
  }
}
```

### Bento grid 2×2 → 1col mobile

```css
.{{ section.id }}-bento {
  display: grid;
  grid-template-columns: 1fr;
  gap: 16px;
}
@media (min-width: 750px) {
  .{{ section.id }}-bento {
    grid-template-columns: 1fr 1fr;
    gap: 24px;
  }
}
```

## Verifica visiva post-build

Prima di marcare la sezione come done:

1. Apri lo screenshot competitor + URL live nostra side-by-side (split screen sul browser).
2. Compara: spacing top/bottom, font-size, font-weight, colore CTA, posizione elementi.
3. Differenze < 5% del rendering → ok.
4. Differenze evidenti → torna allo step 2 (misura) e correggi.

## Cosa NON replicare

- Watermark / loghi di terzi (copyright)
- Foto specifiche del competitor (lascia placeholder)
- Recensioni Trustpilot/Yotpo specifiche (sono di altri clienti, non dei nostri)
- Numeri di fatturato / "X clienti soddisfatti" — usa numeri nostri o rimuovi se non li abbiamo
- Tracking pixel terzi (FB, GTM dei loro account)

## Note operative

- Replica a 100% non significa identità byte-per-byte: il nostro tema ha un layout container (`<main>`, header, footer) che impatta su padding e width. Adatta i valori dello competitor ai vincoli del nostro tema.
- Se una sezione del competitor è troppo "fragile" (basata su immagini hero molto specifiche, lottie complesse, video player custom), avvisa l'utente: "Questa sezione richiede asset specifici (es. video lottie). Posso replicare il layout ma serve un'immagine equivalente da te."
- Mai farsi fregare da uno screenshot stretto o ritagliato: chiedi sempre full-page.
