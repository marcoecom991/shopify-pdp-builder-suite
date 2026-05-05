# Page types — Extras (Contattaci, Traccia ordine, FAQ, About, Blog, Collection)

Riferimento per Fase 7 di `/clone-competitor-store`. Spiega come gestire le pagine extra opzionali che vengono chieste alla fine del flusso principale.

## Quando vengono chieste

Solo dopo che le pagine core (Funnel + PDP + Home) sono completate. La skill chiede:

> "Vuoi aggiungere altre pagine prima di chiudere?"

Opzioni multi-select:
- Contattaci
- Traccia il tuo ordine
- FAQ
- Chi siamo (about)
- Blog landing
- Collection page

Per ogni pagina selezionata, ripete il sub-workflow di Fase 6 (con peso più leggero — sono pagine più standardizzate, non serve replicare 1:1 dal competitor).

## Pattern comune

Ogni pagina extra:
- File template: `templates/page.<store.slug>-<name>.json`
- Sezioni: `sections/<store.slug>-<name>-<NN>-<role>.liquid`
- Layout: `theme` (header + footer standard, mai chromeless)
- L'utente crea Page in Admin assegnando il template

## Specifiche per tipo

### 1. Contattaci (`contact`)

**Slug consigliato**: `contact` (handle Shopify standard, va bene anche per i link automatici)

**Sezioni tipiche**:
1. `<store.slug>-contact-01-hero` — Heading + intro
2. `<store.slug>-contact-02-info` — Email, telefono, indirizzo, orari assistenza
3. `<store.slug>-contact-03-form` — Form `{% form 'contact' %}` Shopify nativo
4. `<store.slug>-contact-04-faq-quick` — opzionale, 3-5 FAQ rapide

**Form contact (Shopify nativo)**:
```liquid
{% form 'contact', class: 'glowria-contact-03__form' %}
  {% if form.posted_successfully? %}
    <div class="success">Grazie per il messaggio. Risponderemo entro 24 ore.</div>
  {% endif %}
  
  {% if form.errors %}
    <div class="errors">
      {{ form.errors | default_errors }}
    </div>
  {% endif %}
  
  <label>Nome *</label>
  <input type="text" name="contact[name]" required>
  
  <label>Email *</label>
  <input type="email" name="contact[email]" required>
  
  <label>Numero ordine (opzionale)</label>
  <input type="text" name="contact[order_number]">
  
  <label>Messaggio *</label>
  <textarea name="contact[body]" rows="6" required></textarea>
  
  <button type="submit">{{ section.settings.cta_label | default: 'Invia messaggio' }}</button>
{% endform %}
```

Il form invia automaticamente all'email impostata in **Settings → Notifications → Customer email**. L'utente deve impostare quella email per riceverle.

**Settings da esporre**: heading hero, intro, email, telefono, indirizzo, orari, label CTA.

### 2. Traccia il tuo ordine (`traccia-il-tuo-ordine`)

**Slug consigliato**: `traccia-il-tuo-ordine`

**Sezioni tipiche**:
1. `<store.slug>-tracking-01-hero` — Heading + intro
2. `<store.slug>-tracking-02-form` — Input numero ordine + email
3. `<store.slug>-tracking-03-faq` — FAQ tracking (tempi spedizione, dove vedere)

**Form tracking**:
Shopify offre la pagina nativa `/account/orders/<token>` ma è dietro login. Per pubblico va creato un form custom che redirige al servizio del corriere o a una pagina di status.

Approccio semplice: form che invia email al supporto:
```liquid
{% form 'contact' %}
  <h2>{{ section.settings.heading }}</h2>
  <p>{{ section.settings.intro }}</p>
  
  <label>Numero ordine</label>
  <input type="text" name="contact[order_number]" required>
  
  <label>Email dell'ordine</label>
  <input type="email" name="contact[email]" required>
  
  <input type="hidden" name="contact[body]" value="Richiesta tracking">
  
  <button type="submit">{{ section.settings.cta_label | default: 'Traccia ordine' }}</button>
{% endform %}
```

Approccio avanzato (se il corriere ha API tracking pubblica): JavaScript che chiama l'API e mostra lo status inline. Out of scope per la skill v1 — la skill fa la versione semplice form-based.

### 3. FAQ

**Slug consigliato**: `faq`

**Sezioni tipiche**:
1. `<store.slug>-faq-01-hero` — Heading
2. `<store.slug>-faq-02-accordion` — Accordion editabile da blocks
3. `<store.slug>-faq-03-cta-bottom` — "Hai altre domande?" + link contact

**Accordion FAQ** con `<details>` HTML nativo (no JS):
```liquid
{% for block in section.blocks %}
  {% if block.type == 'faq_item' %}
    <details class="{{ section.id }}-item" {{ block.shopify_attributes }}>
      <summary class="{{ section.id }}-question">
        {{ block.settings.question }}
        <span class="{{ section.id }}-icon">+</span>
      </summary>
      <div class="{{ section.id }}-answer">
        {{ block.settings.answer }}
      </div>
    </details>
  {% endif %}
{% endfor %}
```

CSS rotation icon su `[open]`:
```css
.{{ section.id }}-item[open] .{{ section.id }}-icon { transform: rotate(45deg); }
```

**Settings da esporre**:
- `blocks[]` di tipo `faq_item` con `question` (text) + `answer` (richtext)
- Hero heading + intro
- CTA bottom label + url (link verso `/pages/contact`)

### 4. Chi siamo (`about`)

**Slug consigliato**: `about` (in italiano: chi-siamo, ma about è handle universale)

**Sezioni tipiche**:
1. `<store.slug>-about-01-hero` — Heading + foto founder/team
2. `<store.slug>-about-02-story` — Story brand (paragrafo lungo + immagini)
3. `<store.slug>-about-03-mission` — Mission/values (3-4 pillar)
4. `<store.slug>-about-04-team` — Team grid (foto + nomi + ruoli)
5. `<store.slug>-about-05-cta` — CTA verso shop

**Tono**: editoriale, storytelling. La fonte di ispirazione è la pagina "Our Story" del competitor (chiedi all'utente l'URL specifico).

⚠️ **Attenzione**: i contenuti di "Our story" del competitor sono quasi sempre **biografia/storia personale del founder**. NON vanno copiati — è la storia di un'altra persona/azienda. Genera testi nuovi seguendo questa struttura ma chiedi all'utente di fornire la storia reale del proprio brand. Se non l'ha: lasciale come placeholder (`[Inserire storia personale del brand qui]`).

### 5. Blog landing

**Slug consigliato**: `blog` (handle Shopify standard per blog)

**Approccio**: Shopify ha una nativa `/blogs/<handle>` che lista articoli. Non serve template page custom per la landing del blog — usa il template `templates/blog.json` (che esiste già nel tema).

Se l'utente vuole una landing custom (es. blog magazine-style con featured article + categories), allora crea `templates/page.<store.slug>-blog-landing.json` con:
1. Hero featured article (dal blog Shopify)
2. Categories grid
3. Latest articles list
4. Newsletter signup

Usa Liquid `{% paginate blog.articles by 9 %}` per fetch articoli.

### 6. Collection page

**Slug consigliato**: dipende dal nome della collection (es. `collections/best-sellers`, `collections/skincare`)

**Approccio**: Shopify ha template nativo `templates/collection.json`. Se serve customizzare, crea `templates/collection.<store.slug>-<name>.json`.

Sezioni tipiche:
1. Hero collection (banner + heading + filter intro)
2. Filter sidebar / horizontal filter bar
3. Product grid (con paginazione)
4. CTA bottom

Usa Liquid:
```liquid
{% paginate collection.products by 12 %}
  {% for product in collection.products %}
    <!-- product card -->
  {% endfor %}
  {{ paginate | default_pagination }}
{% endpaginate %}
```

⚠️ Filtering avanzato (price range, category, color) richiede Shopify Search & Discovery app o storefront filtering API. La skill fa filter base (collection-level), per filter avanzato segnala all'utente e lascia spazio per app custom.

## Workflow per gli extras

Per ogni pagina extra selezionata in Fase 7:

1. **Skip 6.x.1 dell'analisi screenshot**: per gli extras spesso non serve un competitor specifico (sono pagine standard). Chiedi all'utente: "Hai uno screenshot/URL competitor specifico per questa pagina, o uso la struttura standard?"

2. **Genera struttura da default** (vedi sopra per tipo) o adatta a screenshot specifico se l'utente lo manda.

3. **Costruzione sezioni** identica a Fase 6.x.5 (regole editabilità + brand + IT).

4. **Caso speciale per Page** (contact, tracking, FAQ, about):
   ```
   Crea la Page in Shopify Admin:
   1. Admin → Online Store → Pages → Add page.
   2. Title: "<titolo>".
   3. Content: vuoto.
   4. Sidebar destra → Theme template → seleziona `<store.slug>-<name>`.
   5. Save.
   ```

5. **Caso speciale per Blog/Collection** (template-only, no Page entity):
   - Blog: Admin → Online Store → Blog posts → l'utente crea articoli reali, il template li mostra.
   - Collection: Admin → Products → Collections → l'utente crea la collection e assegna prodotti, poi nella sidebar destra → Theme template seleziona quello custom.

## Footer condiviso

Tutte le pagine extra (e le core) usano lo **stesso footer** del tema, che la skill ha popolato in Fase 5 con:
- Email assistenza
- Telefono
- Indirizzo legale
- Logo
- Link policy (Privacy, Terms, Refund, Shipping) — se l'utente li ha già configurati in Settings → Policies

Se il tema ha una sezione footer custom (es. `sections/footer.liquid`), la skill aggiorna i `default` dei `setting` del footer in Fase 5 per popolare `email`, `phone`, `address`, `logo`. Se il footer è standard del tema (Dawn/Horizon), Shopify lo gestisce via Theme Settings → Customize → Footer.

## Quando chiudere

Quando l'utente non vuole aggiungere altre pagine extra → vai a Fase 8 (verifica finale).

## Lista non esaustiva di altre pagine possibili

(Su richiesta utente, fuori dal default)

- **Trova negozio fisico** (per brand omnichannel)
- **Programma fedeltà** (descrizione + CTA iscrizione)
- **Affiliati** (form per partner)
- **Press kit** (download asset stampa)
- **Recensioni dedicate** (Yotpo wall standalone)

Se richieste, segui lo stesso pattern (template page + sezioni + Admin Page assignment).
