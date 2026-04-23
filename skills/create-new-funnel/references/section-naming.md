# Section naming — convenzioni per prefissi e nomi

Quando si crea un nuovo template PDP, **ogni sezione referenziata va duplicata con un prefisso nuovo** per evitare di modificare le sezioni del template di partenza (che è già live su un altro prodotto).

## Regola del prefisso

Derivazione automatica dal nome template:

| Nome template               | Prefisso sezioni | Esempio file sezione              |
| --------------------------- | ---------------- | --------------------------------- |
| `crema-borse-occhiaie-pdp`  | `cboe-`          | `sections/cboe-pdp-05.liquid`     |
| `autoabbronzante-listicle`  | `alist-` o `listicle-` | `sections/listicle-02-hero.liquid` |
| `nuovo-prodotto-pdp`        | `npp-`           | `sections/npp-pdp-05.liquid`      |
| `siero-vitamina-c-pdp`      | `svc-`           | `sections/svc-pdp-05.liquid`      |

**Come generare il prefisso di default**: prendere le iniziali di ogni parola del nome template (separate da `-`), lowercase, più `-`.

Esempio: `crema-borse-occhiaie-pdp` → `c` + `b` + `o` + `e` → `cboe-`.

L'utente può comunque scegliere un prefisso diverso se preferisce (es. per leggibilità).

## Struttura del suffix dopo il prefisso

Mantieni la parte identificativa della sezione originale dopo il prefisso:

| Sezione originale                         | Duplicato (con prefisso `npp-`)       |
| ----------------------------------------- | ------------------------------------- |
| `sections/cboe-hero-badge.liquid`         | `sections/npp-hero-badge.liquid`      |
| `sections/cboe-pdp-05.liquid`             | `sections/npp-pdp-05.liquid`          |
| `sections/cboe-pdp-06a-video.liquid`      | `sections/npp-pdp-06a-video.liquid`   |

**Sostituisci solo la prima occorrenza del prefisso**, il resto del nome resta identico. Questo preserva la leggibilità dell'elenco sezioni nel theme editor.

## Aggiornamento del template JSON

Nel nuovo `templates/product.<nome>.json`, sostituisci ogni riferimento al vecchio prefisso con il nuovo:

```diff
- "type": "cboe-pdp-05",
+ "type": "npp-pdp-05",
```

Questo va fatto per ogni blocco `"type"` del JSON, inclusi i blocchi dentro `blocks` nidificati se presenti.

## Aggiornamento dello schema dentro il file `.liquid`

In ogni sezione duplicata, aggiorna `schema.name` e `presets[0].name` per distinguerla nel theme editor:

```diff
  {% schema %}
  {
-   "name": "CBOE PDP 05",
+   "name": "NPP PDP 05",
    ...
    "presets": [
      {
-       "name": "CBOE PDP 05"
+       "name": "NPP PDP 05"
      }
    ]
  }
  {% endschema %}
```

Questo evita che il theme editor mostri due sezioni con lo stesso nome (l'originale e la duplicata), confusionario per chi usa l'editor.

## Esempi storici (riusabili)

### Nimea — crema borse occhiaie (PDP)
Sezioni `cboe-*` duplicate da `product.berberina-pills.json` (12 file):
- `cboe-hero-badge`
- `cboe-pdp-03`, `cboe-pdp-05`, `cboe-pdp-06a-video`, `cboe-pdp-06b-doctors`, `cboe-pdp-07`
- `cboe-pdp-10`, `cboe-pdp-11`, `cboe-pdp-12`, `cboe-pdp-13`, `cboe-pdp-18`, `cboe-pdp-21`

### Glowria — autoabbronzante listicle (pre-PDP funnel)
Sezioni `listicle-*` (11 file, pagina chromeless, non PDP):
- `listicle-01-announcement` fino a `listicle-11-footer`
- Layout: `theme.gempages.blank` (nessun header/footer)

## Naming sui nomi che contengono numeri sequenziali

Per template PDP, la numerazione interna (`-05`, `-06a`, `-06b`, `-18`, `-21`) è **ereditata dal template originale**. Non rinumerarla per il nuovo template — lasciala identica. Questo rende più semplice mappare 1:1 sezione originale ↔ sezione duplicata nella fase di popolamento testi.

---

## Naming per FUNNEL (create-new-funnel)

Per i template di funnel (`templates/page.<nome>.json`) le regole sono leggermente diverse dalla PDP perché le sezioni **nascono da zero**, non vengono duplicate da un template già esistente. Quindi la numerazione è **incrementale** e segue l'ordine di scorrimento verticale della pagina.

### Regola del prefisso per funnel

Due convenzioni accettate, l'utente sceglie:

**Breve (consigliata)** — iniziali del nome template + `-`:

| Nome template                       | Prefisso  | Esempio file                         |
| ----------------------------------- | --------- | ------------------------------------ |
| `autoabbronzante-advertorial`       | `aa-`     | `sections/aa-01-hero.liquid`         |
| `berberina-listicle`                | `bl-`     | `sections/bl-01-announcement.liquid` |
| `crema-rassodante-quiz`             | `crq-`    | `sections/crq-01-hero-quiz.liquid`   |

**Esplicita** — nome template + tipo funnel + `-`:

| Nome template                       | Prefisso              | Esempio file                                      |
| ----------------------------------- | --------------------- | ------------------------------------------------- |
| `autoabbronzante-advertorial`       | `aa-adv-`             | `sections/aa-adv-01-hero.liquid`                  |
| `berberina-listicle`                | `bb-lst-`             | `sections/bb-lst-01-announcement.liquid`          |

La forma breve è più leggibile nel theme editor; quella esplicita è utile se lo stesso prodotto ha più funnel diversi coesistenti nel tema.

### Suffix per funnel — numero + ruolo

Dopo il prefisso, ogni sezione porta `<NN>-<ruolo>`:

- `01-hero` — sezione iniziale (intro articolo, promessa quiz, announcement)
- `02-problem` — hook sul dolore / apertura narrativa
- `03-story` — storia personale (solo advertorial)
- `04-solution` — introduzione prodotto/metodo
- `05-benefits` — lista benefici (listicle) o punti chiave (advertorial)
- `06-proof` — social proof, testimonianze, screenshot
- `07-faq` — obiezioni gestite
- `08-cta` — call-to-action ripetuta verso PDP
- `09-footer` — chiusura o disclaimer

Non è una lista rigida — segue l'ispirazione competitor raccolta in Fase 6 della skill. L'importante è mantenere la **numerazione** (01, 02, 03...) per leggere l'ordine a colpo d'occhio in `sections/`.

### Differenza chiave con PDP

| Aspetto                | PDP (create-new-pdp)                               | Funnel (create-new-funnel)                             |
| ---------------------- | -------------------------------------------------- | ------------------------------------------------------ |
| Origine file           | Duplicati da un template base esistente            | Creati da zero                                         |
| Numerazione            | Ereditata (05, 06a, 06b, 18, 21…)                  | Incrementale (01, 02, 03…)                             |
| Ruolo nel suffix       | Implicito nel nome del file base                   | Esplicito (`-hero`, `-problem`, `-cta`…)               |
| Template               | `templates/product.<nome>.json`                    | `templates/page.<nome>.json`                           |
| Layout                 | Sempre `theme` (header+footer sempre presenti)     | Scelta: `theme` o `theme.chromeless` (funnel full)     |

### Esempi storici

- **Glowria — autoabbronzante listicle** (11 file, chromeless, prefisso `listicle-`):
  - `listicle-01-announcement`, `listicle-02-hero`, `listicle-03-reason-01`..`listicle-03-reason-04`, `listicle-04-comparison`, `listicle-05-testimonials`, `listicle-09-offer`, `listicle-10-faq`, `listicle-11-footer`.

Per nuovi funnel, seguire un pattern simile ma con il prefisso derivato dal nome template (non `listicle-` generico).
