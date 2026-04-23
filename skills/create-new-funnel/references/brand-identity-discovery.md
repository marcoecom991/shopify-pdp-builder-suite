# Brand identity discovery — come estrarre colori e tipografia

Durante la Fase 3 della skill `/create-new-funnel`, Claude deve capire quali colori e font usa il brand dello store scelto, **partendo da una PDP esistente indicata dall'utente**. L'obiettivo è che il funnel nuovo sia visivamente coerente col resto del sito.

## Flusso di discovery

1. **Chiedi all'utente quale PDP usare come riferimento.** Elenca `templates/product.*.json` nel workdir, presentali leggibili (es. `berberina-pills`, `crema-borse-occhiaie-pdp`), lascia scegliere.
2. **Leggi in parallelo 4 fonti** del workdir:
   - `templates/product.<scelto>.json` — per la lista sezioni.
   - Le prime 2-3 sezioni referenziate (di solito hero + sezione con CTA primario).
   - `config/settings_data.json` — oggetto `current` con `colors`, `typography`, `color_schemes`.
   - `config/settings_schema.json` (opzionale) — per dare nomi umani ai settings.
3. **Estrai e normalizza** (vedi sotto).
4. **Mostra all'utente un riepilogo** e chiedi conferma o correzioni puntuali.

## Cosa estrarre e dove cercarlo

### Palette colori

**Fonte primaria**: `config/settings_data.json` → `current.color_schemes`

Shopify Online Store 2.0 salva schemi colore come:

```json
{
  "current": {
    "color_schemes": {
      "scheme-1": {
        "settings": {
          "background": "#FFFFFF",
          "text": "#1F1F1F",
          "button": "#1A3A5C",
          "button_label": "#FFFFFF",
          "secondary_button_label": "#1A3A5C"
        }
      },
      "scheme-2": { ... }
    }
  }
}
```

Normalizza così:

| Campo output       | Mappato da                                          |
|--------------------|-----------------------------------------------------|
| `primary`          | `button` dello scheme usato dall'hero della PDP     |
| `primary_text`     | `button_label`                                      |
| `text`             | `text`                                              |
| `background`       | `background`                                        |
| `accent`           | Colore più usato nei badge/highlight delle sezioni  |
| `secondary`        | Second color scheme dominante                       |

**Fonte secondaria**: HEX hardcoded nelle sezioni hero/CTA. Cercali con regex `#[0-9A-Fa-f]{3,6}` dentro il file `.liquid` e rankali per frequenza. Colori che compaiono >3 volte sono candidati primary/secondary.

**Euristica CTA primario**: il colore più usato in `class="*button*"`, `.btn-primary`, `.cta` o su elementi `<button>` / `<a class="button">` dentro la sezione hero è quasi certamente il primario.

### Tipografia

**Fonte primaria**: `config/settings_data.json` → `current` → campi font

```json
{
  "current": {
    "type_header_font": "playfair_display_n4",
    "type_body_font": "inter_n4"
  }
}
```

I valori sono **Shopify font handles** (`<family>_<weight><style>`). Per leggibilità convertili:
- `playfair_display_n4` → "Playfair Display, serif" (n=normal, 4=weight 400)
- `inter_n5` → "Inter, sans-serif" (weight 500)
- `inter_i4` → "Inter, sans-serif" italic

**Fonte secondaria**: `font-family:` hardcoded nelle sezioni. Cercalo con regex.

**Regola classificazione**:
- Il font applicato a `<h1>`, `<h2>`, `.title`, `.headline` → **heading font**.
- Il font applicato a `<p>`, `<body>`, `.text`, `.description` → **body font**.
- Se entrambi usano lo stesso font handle → nel riepilogo nota "Single font: X".

### Stili ricorrenti

Guarda i CSS inline delle sezioni hero/CTA e estrai:

| Stile             | Come trovarlo                                            | Esempio output                |
|-------------------|----------------------------------------------------------|-------------------------------|
| `border-radius`   | `border-radius:` nei bottoni e card                      | `12px` / `999px` (pill)       |
| `box-shadow`      | `box-shadow:` nei container elevated                     | `soft` / `sharp` / `none`     |
| `button style`    | Mix di radius, padding, letter-spacing                   | `pill`, `squared`, `outlined` |
| `heading weight`  | `font-weight` su h1/h2                                   | `400` / `700` / `900`         |
| `heading style`   | `text-transform`, `font-style`                           | `uppercase bold`, `italic`    |

Queste info servono al funnel per produrre bottoni/titoli che "sembrano dello stesso tema".

## Output atteso — oggetto `brand`

Riepilogo da mostrare all'utente e salvare in memoria:

```
Brand discovery — riferimento: product.berberina-pills

Palette:
  Primario:     #1A3A5C  (bottoni CTA, headline sfondo)
  Primary text: #FFFFFF  (testo su primario)
  Secondario:   #F4E4D4  (bg card, sezioni alternate)
  Accent:       #E63946  (badge, highlight, prezzi barrati)
  Testo:        #1F1F1F  (body text)
  Background:   #FFFFFF  (pagina)

Tipografia:
  Heading:  "Playfair Display", serif (weight 700)
  Body:     "Inter", sans-serif (weight 400)

Stili ricorrenti:
  Bottoni:    radius 8px, padding 14px 28px, letter-spacing 0.5px
  Card:      radius 12px, shadow soft (0 4px 12px rgba(0,0,0,0.06))
  Headline:  uppercase, bold, tracking tight
```

## Conferma utente — override puntuali

Dopo aver mostrato il riepilogo, usa `AskUserQuestion` oppure chiedi in chiaro:
- "Tutto OK così?" → procedi.
- Se l'utente corregge un singolo campo (es. "no, il primario è #0F2540 non #1A3A5C") → applica la correzione e ripresenta.
- Se la PDP scelta è atipica rispetto al resto del sito → offri di cambiare PDP di riferimento.

## Esempi pratici

### Nimea (berberina-pills)
```
Primario: #1E3A5F (blu navy)
Secondario: #F5E6D3 (crema)
Heading: "Playfair Display", Body: "Inter"
```

### Glowria (crema-corpo-autoabbronzante)
```
Primario: #C9764C (terracotta)
Accent: #F7D9C4 (pesca chiaro)
Heading: "Fraunces" (serif), Body: "DM Sans" (sans)
```

## Fallback: settings_data mancante o atipico

Se `config/settings_data.json` non contiene `color_schemes` (temi legacy pre-2.0) o la PDP scelta usa GemPages (CSS inline pesante), l'estrazione automatica potrebbe essere incompleta.

In quel caso:
1. Estrai quello che riesci dalle sezioni.
2. Mostra all'utente quello che hai trovato + lista dei **gap** (es. "font heading non chiaro").
3. Chiedi all'utente di **colmare i gap** manualmente (incolla HEX / nome font).
4. Salva l'oggetto `brand` completo con i valori utente.
