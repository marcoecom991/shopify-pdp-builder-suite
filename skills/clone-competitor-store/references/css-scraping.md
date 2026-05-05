# CSS scraping del competitor

Riferimento per Fase 6.x.4.bis di `/clone-competitor-store`. Step **opzionale** che porta la fedeltà visuale del clone da ~70% (ricostruzione a vista) a ~90% (CSS sorgente riapplicato).

## Perché è legittimo

I file CSS di un sito web sono **asset pubblici**: vengono inviati al browser di chiunque visiti il sito (è il modo in cui il browser sa come renderizzare il sito). Sono raggiungibili con un GET HTTP semplice, senza autenticazione. Scaricarli con `curl` è esattamente quello che fa qualsiasi browser.

L'uso che ne facciamo è **transformative**: estraiamo valori CSS (padding, font-size, colori, ecc.) e li riapplichiamo nel nostro tema con classi nostre, su markup nostro, per un cliente nostro. Non stiamo né ridistribuendo né riprodurre il file CSS originale — stiamo usando i suoi valori come riferimento per ricostruire un design simile, esattamente come un dev userebbe Inspect Element + DevTools per studiare un sito.

Questo workflow è standard nel mondo agency / front-end dev professionale.

## Limiti onesti del CSS scraping

CSS scraping funziona bene con:
- Siti Shopify con tema "classico" (Dawn, Sense, Studio, Pipeline, ecc.) → CSS leggibile, classi semantiche
- Siti WordPress con WooCommerce → CSS organizzato, breakpoint standard
- Siti custom con classi BEM o utility-first leggibili

CSS scraping funziona MENO BENE con:
- React/Next.js con CSS-in-JS (styled-components, Emotion) → classi auto-generate tipo `.css-1abc2de` linkate a JSX specifico — non riusabili 1:1 sul nostro markup HTML
- Tailwind CSS purge-mode → classi atomiche generate al build time, presenti solo dove usate
- Webflow con CSS custom heavy → spesso pesante e con regole molto specifiche

In quei casi: skip CSS scraping (Opzione 3 della Fase 6.x.4.bis) e ricostruisci a vista.

## Procedura dettagliata

### Step 1 — Download HTML della pagina competitor

```bash
mkdir -p /tmp/cb-css-<store-slug>
curl -sL "<current_page.url>" \
  -A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36" \
  -H "Accept: text/html,application/xhtml+xml" \
  -o "/tmp/cb-css-<store-slug>/page-<NN>.html"
```

User-Agent realistico per evitare bot blocks. Se il server restituisce 403/429/Cloudflare challenge → non insistere, segnala all'utente "il competitor blocca i bot, suggerisco di salvare la pagina manualmente con `Cmd+S` da Chrome e mandarmi il file HTML".

### Step 2 — Estrazione `<link rel="stylesheet">` e `<style>` inline

Usa `python3` heredoc dentro `Bash` per parsing robusto:

```bash
python3 <<'PY'
import re, os
from urllib.parse import urljoin

with open("/tmp/cb-css-<slug>/page-<NN>.html") as f:
    html = f.read()

base_url = "<current_page.url>"

# Estrai <link rel="stylesheet" href="...">
links = re.findall(r'<link[^>]*rel=["\']stylesheet["\'][^>]*href=["\']([^"\']+)["\']', html, re.IGNORECASE)
links += re.findall(r'<link[^>]*href=["\']([^"\']+)["\'][^>]*rel=["\']stylesheet["\']', html, re.IGNORECASE)

# Estrai <style>...</style> inline
inline_styles = re.findall(r'<style[^>]*>(.*?)</style>', html, re.DOTALL | re.IGNORECASE)

# Risolvi URL relativi
absolute_links = [urljoin(base_url, link) for link in links]

# Salva la lista per il prossimo step
os.makedirs("/tmp/cb-css-<slug>", exist_ok=True)
with open("/tmp/cb-css-<slug>/css-urls.txt", "w") as f:
    f.write("\n".join(absolute_links))

with open("/tmp/cb-css-<slug>/inline-styles.css", "w") as f:
    f.write("\n\n".join(inline_styles))

print(f"Found {len(absolute_links)} external CSS files")
print(f"Found {len(inline_styles)} <style> blocks ({sum(len(s) for s in inline_styles)} chars total)")
PY
```

### Step 3 — Download dei file CSS esterni

```bash
counter=1
while IFS= read -r url; do
  curl -sL "$url" \
    -A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36" \
    -o "/tmp/cb-css-<slug>/external-${counter}.css"
  counter=$((counter+1))
done < "/tmp/cb-css-<slug>/css-urls.txt"
```

### Step 4 — Aggregazione

```bash
cat /tmp/cb-css-<slug>/inline-styles.css \
    /tmp/cb-css-<slug>/external-*.css \
    > "/tmp/cb-css-<slug>/aggregated-<NN>.css"

# Conta righe e KB
wc -l "/tmp/cb-css-<slug>/aggregated-<NN>.css"
du -h "/tmp/cb-css-<slug>/aggregated-<NN>.css"
```

Salva in memoria: `current_page.competitor_css_path = "/tmp/cb-css-<slug>/aggregated-<NN>.css"`.

### Step 5 — Estrazione regole rilevanti per ogni sezione

Per ogni sezione mappata in 6.x.2 con `competitor_selector` definito (Fase 6.x.4.bis.2), leggi il file CSS aggregato e estrai:

1. **Regole che colpiscono direttamente il selettore**:
   ```
   .hero-banner { ... }
   .hero-banner.above-fold { ... }
   ```
2. **Regole sui figli/discendenti**:
   ```
   .hero-banner h1 { ... }
   .hero-banner .cta-primary { ... }
   .hero-banner img { ... }
   ```
3. **Regole sui pseudo-elementi e media queries**:
   ```
   .hero-banner::before { ... }
   @media (min-width: 750px) { .hero-banner { ... } }
   ```

Per fare l'estrazione in modo affidabile, usa `python3` con regex sui blocchi CSS:

```python
import re

with open("/tmp/cb-css-<slug>/aggregated-<NN>.css") as f:
    css = f.read()

selector_root = ".hero-banner"  # da current_page.sections[<NN>].competitor_selector

# Pattern: cattura blocchi `selector { ... }` che contengono il root
# Anche dentro @media query
pattern = re.compile(
    r'(@media[^{]+\{[^{}]*'  # Inizio media query (opzionale)
    r'|^|\})\s*'              # O inizio file/fine blocco precedente
    r'([^{}]*' + re.escape(selector_root) + r'[^{}]*)\s*'  # Selettore che contiene il root
    r'\{([^{}]*)\}',          # Body del blocco
    re.MULTILINE
)

matches = pattern.findall(css)
relevant_rules = "\n".join(f"{sel.strip()} {{{body.strip()}}}" for _, sel, body in matches)

print(relevant_rules[:5000])  # Anteprima
```

### Step 6 — Riapplicazione sulle classi nostre

Una volta estratte le regole rilevanti per la sezione, **traduci i selettori del competitor → nostri prefissi scoped** prima di scriverle nel `<style>` della sezione liquid.

Mapping:
- `.hero-banner` → `.{{ section.id }}-hero` (la classe wrapper della nostra sezione)
- `.hero-banner h1` → `.{{ section.id }}-hero h1`
- `.hero-banner .cta-primary` → `.{{ section.id }}-hero .{{ section.id }}-cta`
- ...

Il replacement va fatto con `python3` o `sed` su tutto il blocco delle regole estratte:

```python
selector_competitor = ".hero-banner"
selector_ours = ".{{ section.id }}-hero"

translated = relevant_rules.replace(selector_competitor, selector_ours)
# Replica per i selettori di figli/CTA:
translated = translated.replace(".cta-primary", f"{selector_ours}-cta")
# ...

# Inserisci translated dentro <style> del .liquid
```

### Step 7 — Override con i nostri valori brand-specifici

Dopo aver applicato il CSS del competitor, **sovrascrivi** i valori che vogliamo siano controllati dal nostro brand identity (Fase 4):
- Colori CTA → da `brand.palette.cta` (anche se il competitor ne ha uno diverso, vogliamo il nostro brand)
- Logo / brand-mark colors → `brand.palette.primary`
- Font headings/body → da `brand.fonts.*` (importati via Google Fonts)

Questo perché lo scopo della clone è "stesso design, brand nostro". Il rosso del CTA del competitor non deve restare rosso se il nostro brand è coral.

## Quando NON usare CSS scraping

- Competitor con framework JS heavy (React + CSS-in-JS) → `Opzione 3` della Fase 6.x.4.bis (fallback a vista)
- Pagine dietro login/paywall → `curl` non riesce ad accedere
- Cloudflare bot detection blocca i nostri request → fallback a HTML salvato manualmente dall'utente (`Cmd+S` su Chrome)
- File CSS gigante (>2MB) di sito enterprise → estrazione regole troppo lenta, valuta caso per caso

## File temporanei e cleanup

I file scaricati restano in `/tmp/cb-css-<slug>/` per la durata della sessione. Sono temp, possono essere cancellati a fine clone:

```bash
rm -rf /tmp/cb-css-<slug>
```

Da fare in Fase 8 (verifica finale) come pulizia.

## Esempio end-to-end

Pagina advertorial competitor: `https://primedrum.com/pages/why-it-works`

```bash
# Step 1
mkdir -p /tmp/cb-css-capitan-bucato
curl -sL "https://primedrum.com/pages/why-it-works" \
  -A "Mozilla/5.0..." \
  -o /tmp/cb-css-capitan-bucato/page-01.html

# Step 2 (estrazione URL CSS via python)
python3 <<'PY'
# ... vedi sopra
PY
# → scopre 4 file CSS esterni + 2 <style> inline

# Step 3 (download)
curl -sL "https://primedrum.com/cdn/shop/t/2/assets/theme.css" -o /tmp/cb-css-capitan-bucato/external-1.css
curl -sL "https://primedrum.com/cdn/shop/t/2/assets/sections.css" -o /tmp/cb-css-capitan-bucato/external-2.css
# ...

# Step 4 (aggregato 380KB di CSS)
cat /tmp/cb-css-capitan-bucato/inline-styles.css /tmp/cb-css-capitan-bucato/external-*.css \
    > /tmp/cb-css-capitan-bucato/aggregated-01.css

# Step 5 + 6 + 7 → fatti per ogni sezione durante 6.x.5.b
```

Risultato: il file `sections/capitan-bucato-adv-02-hero.liquid` contiene padding, font-size, colors, spacing **estratti dal CSS reale** del competitor + override del brand del cliente. Fedeltà visiva ~90%.
