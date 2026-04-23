# Page template layout — chromeless vs theme

Un funnel è una pagina (`templates/page.<nome>.json`), e il primo grande bivio è: **header/footer Shopify on or off?**. Questo controlla se la pagina eredita navbar + footer del tema o si presenta full-screen come una landing dedicata.

## Schema minimo `templates/page.<nome>.json`

```json
{
  "sections": {
    "main": {
      "type": "main-page",
      "settings": {}
    }
  },
  "order": [
    "main"
  ],
  "layout": "theme"
}
```

- `sections` — oggetto che mappa `key → { type, settings, blocks }`. Ogni `key` identifica un blocco nella pagina.
- `order` — array ordinato di `key`. Definisce l'ordine di rendering dall'alto al basso.
- `layout` — **chiave decisiva**: dice quale file `layout/*.liquid` avvolge il contenuto.

## Opzioni di `layout`

### `"layout": "theme"` (default Shopify)

Avvolge la pagina con `layout/theme.liquid`, che di solito include:
- `<head>` col meta + font + CSS globale
- Header (navbar del tema)
- `{{ content_for_layout }}` ← dove vanno le sezioni della pagina
- Footer del tema

**Quando usarlo**: funnel informativo che fa parte della navigazione normale (es. articolo blog-like che l'utente può raggiungere dal menu).

### `"layout": "theme.chromeless"` (se il tema lo supporta)

Avvolge con `layout/theme.chromeless.liquid`, tipicamente:
- `<head>` identico a `theme.liquid`
- **NESSUN header**
- `{{ content_for_layout }}`
- **NESSUN footer** (o solo disclaimer minimo)

**Quando usarlo**: funnel "ads-destination" dove l'utente arriva da Meta/TikTok e deve essere intrappolato nella pagina, senza distrazioni del menu. **Default consigliato per funnel**.

### `"layout": false`

Nessun layout — il template si auto-wrappa (deve contenere anche `<html>`, `<head>`, `<body>`). Raramente necessario.

### Custom — es. `"layout": "page.funnel"`

Per creare un layout dedicato, si crea `layout/page.funnel.liquid` con solo il chrome che serve (es. mini-header con logo + footer legal minimo). Richiede:
1. Creare il file `layout/page.funnel.liquid`.
2. Referenziarlo in `templates/page.<nome>.json` → `"layout": "page.funnel"`.
3. Pushare sia il layout sia il template.

## Verifica se il tema supporta `theme.chromeless`

```bash
ls <workdir>/layout/
```

Se esiste `theme.chromeless.liquid` → supportato.
Se no:
- Opzione A: crea un file `layout/theme.chromeless.liquid` duplicando `theme.liquid` e rimuovendo header + footer.
- Opzione B: usa un nome custom (`layout/page.funnel.liquid`) e procedi.

**Quando crei un layout chromeless manualmente**, il contenuto minimo è:

```liquid
<!doctype html>
<html class="no-js" lang="{{ request.locale.iso_code }}">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=2.0">
    <link rel="canonical" href="{{ canonical_url }}">
    {{ content_for_header }}
    {{ 'base.css' | asset_url | stylesheet_tag }}
    <title>{{ page_title }}</title>
  </head>
  <body>
    {{ content_for_layout }}
  </body>
</html>
```

Niente header, niente footer, niente navigazione. Solo le sezioni definite nel template.

## Precedenti noti

### Glowria — autoabbronzante listicle
- File: `templates/page.glowria-autoabbronzante-listicle.json`
- Layout: `"layout": "theme.gempages.blank"` (layout custom creato da GemPages pre-detachment)
- Dopo detach: può essere migrato a `"theme.chromeless"` se supportato dal tema Dawn-based.

### Nimea — nessun funnel attuale
Tutti i prodotti hanno solo PDP. Il primo funnel Nimea userà di default `theme.chromeless` se disponibile, altrimenti layout custom `layout/funnel.liquid`.

## Checklist post-push

Dopo push del `templates/page.<nome>.json`:
1. Crea la Page in Admin (Online Store → Pages → Add page).
2. In "Theme template" seleziona il nuovo nome.
3. Apri l'URL `/pages/<handle>` in incognito.
4. Verifica visivamente:
   - Se layout `theme` → vedi header + footer del tema ✓
   - Se layout `theme.chromeless` o custom → **NESSUN** header/footer ✓
5. DevTools → rete → nessun 404 su CSS/JS.

Se vedi ancora header quando ti aspettavi chromeless:
- Il tema non ha `theme.chromeless.liquid` → creane uno custom e rifai push.
- Il `"layout"` nel JSON è errato → correggi e rifai push.

## Riepilogo decisione rapida

| Situazione                                  | Layout da scegliere          |
|---------------------------------------------|------------------------------|
| Funnel destinazione ads, full-screen        | `theme.chromeless` (o custom) |
| Articolo informativo nella navigation       | `theme`                       |
| Il tema non ha chromeless                   | Crea `layout/funnel.liquid`   |
| Quiz funnel immersivo                       | `theme.chromeless`            |
| Advertorial linkato da SEO                  | `theme` (serve l'header per brand recognition)  |
