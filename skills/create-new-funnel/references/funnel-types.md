# Funnel types — strutture base

Ogni tipologia di funnel ha una **struttura standard di partenza** che Claude propone all'utente in Fase 7. La struttura viene poi **raffinata** con gli esempi competitor raccolti in Fase 6.

## Advertorial

### Quando usarlo
Prodotto che richiede **contesto emotivo / storytelling** per essere capito. Target freddo che non cerca il prodotto direttamente ma ha un problema. Tipico: skincare, integratori, prodotti per risolvere problemi personali.

### Struttura base (8-11 sezioni)

| NN | Ruolo                | Cosa contiene                                                                     | Word count |
|----|----------------------|-----------------------------------------------------------------------------------|------------|
| 01 | announcement bar     | Badge "Articolo sponsorizzato" / "Guida completa"                                 | 5-10       |
| 02 | hero                 | Headline curiosity-driven + sub + 1 foto lifestyle + date/author                  | 30-60      |
| 03 | problem / hook       | "Anche a te succede X?" — apertura empatica sul dolore                            | 80-150     |
| 04 | story                | Storia personale dell'autore (o cliente): prima-dopo, la scoperta                 | 200-400    |
| 05 | solution intro       | "Poi ho scoperto questo..." — introduzione del prodotto/metodo                    | 100-200    |
| 06 | how it works         | 3-4 step di come funziona il prodotto + ingredienti chiave                        | 150-250    |
| 07 | social proof         | 2-3 testimonianze con foto + numeri (es. "oltre 10.000 clienti")                  | 100-150    |
| 08 | objections / faq     | 3-5 obiezioni comuni smontate ("Funziona sulla pelle sensibile?")                 | 200-300    |
| 09 | cta offer            | Offerta speciale + countdown + bottone verso PDP                                  | 50-100     |
| 10 | trust badges         | Garanzia soddisfatti, spedizione gratuita, pagamenti sicuri                       | 20-40      |
| 11 | footer disclaimer    | Legal note + link PDP                                                             | 20-30      |

### Pattern copy
- **Hook**: domanda retorica sul dolore del target ("Stanca di sentirti gonfia?").
- **Story**: prima persona, dettagli concreti (età, città, quanto tempo, cosa aveva provato prima).
- **Transition alla soluzione**: "Poi una mia amica mi ha consigliato..." / "Ho letto di..." — soft, non pushy.
- **CTA**: benefit-focused, non "compra", ma "prova" / "scopri" / "inizia".

## Listicle

### Quando usarlo
Prodotto con **benefici multipli chiari** o confronto forte con alternative. Target semi-caldo che sa cosa cerca ma vuole decidere quale prodotto. Tipico: integratori con X motivi per sceglierli, prodotti di bellezza con top N benefici.

### Struttura base (9-12 sezioni)

| NN | Ruolo                | Cosa contiene                                                                     | Word count |
|----|----------------------|-----------------------------------------------------------------------------------|------------|
| 01 | announcement bar     | Badge categoria o data                                                            | 5-10       |
| 02 | hero                 | Headline con numero ("5 motivi per scegliere X") + foto prodotto/categoria       | 30-60      |
| 03 | intro                | 1-2 paragrafi che introducono il topic e promettono la lista                      | 100-150    |
| 04 | reason 01            | Card: numero grande + titolo benefit + 2-3 frasi + immagine                       | 50-100 ea  |
| 05 | reason 02            | idem                                                                              | 50-100 ea  |
| 06 | reason 03            | idem                                                                              | 50-100 ea  |
| 07 | reason 04/05         | idem (solo se servono, a gusto)                                                   | 50-100 ea  |
| 08 | comparison           | Tabella "noi vs altri" con checkmark e X                                          | 80-120     |
| 09 | testimonials         | 3-5 card recensioni con foto + rating                                             | 100-200    |
| 10 | cta offer            | Offerta + bottone verso PDP                                                       | 50-100     |
| 11 | faq                  | 3-5 FAQ accordion                                                                 | 150-250    |
| 12 | footer               | Footer con disclaimer + secondo bottone                                           | 20-40      |

### Pattern copy
- **Hero number**: preferibilmente dispari (3, 5, 7) — suona più "curato".
- **Reason cards**: ogni card autonoma, deve reggere anche fuori contesto.
- **Comparison**: max 5 righe comparative, altrimenti diventa un wall of text.
- **Tone**: più informativo/oggettivo dell'advertorial, meno personale.

### Riferimento reale
Glowria autoabbronzante listicle (`listicle-01` → `listicle-11`) è un listicle completo già vivo, usabile come benchmark.

## Quiz funnel

### Quando usarlo
Prodotto **personalizzabile** (es. skincare per tipo di pelle) o dove l'engagement prolungato aumenta conversione. Target freddo da intrattenere/qualificare. Tipico: routine skincare, integratori personalizzati.

### Struttura base (5-8 sezioni)

| NN | Ruolo                 | Cosa contiene                                                                    | Word count |
|----|-----------------------|----------------------------------------------------------------------------------|------------|
| 01 | hero quiz             | Headline promessa ("Scopri il prodotto perfetto per te") + CTA "Inizia quiz"     | 30-50      |
| 02 | quiz wrapper          | Container con Q1 visibile, Q2-N nascoste finché risposta precedente non data     | —          |
| 03 | question 1            | Domanda + 2-4 opzioni card (radio)                                               | 30-50 ea   |
| 04 | question 2            | idem                                                                             | 30-50 ea   |
| 05 | question 3-5          | idem                                                                             | 30-50 ea   |
| 06 | result                | "Il tuo risultato è: [Prodotto X]" + descrizione + bottone verso PDP             | 100-200    |
| 07 | alternative social    | "Oltre 5.000 persone hanno fatto il quiz" + 2-3 quotes                           | 50-100     |
| 08 | footer                | Disclaimer                                                                       | 20-30      |

### Pattern tecnici
- **JS inline**: ogni Q invia il next-step tramite event listener o data-attribute state machine.
- **State**: può essere salvato in localStorage per evitare di ripartire se l'utente ricarica.
- **Risultato**: se hai un solo prodotto, il risultato è sempre quello ma con messaggio variabile in base alle risposte.
- **No full SPA**: tutto dentro la pagina singola, senza router.

### Pattern copy
- **Hero**: pone una domanda, non un'affermazione ("Qual è la tua routine ideale?").
- **Domande**: tono conversazionale, max 8 parole per domanda.
- **Opzioni**: 2-4 card, ogni una con icon + label + 1 frase breve.
- **Risultato**: personale ("Basandoci sulle tue risposte...") non generico.

## Altro (custom)

Se l'utente sceglie "Altro", Claude:
1. Chiede una descrizione testuale della struttura desiderata.
2. Chiede almeno 1 esempio competitor come ancora di riferimento.
3. Propone una struttura base inventata con N sezioni numerate e ruoli descrittivi.
4. L'utente conferma/modifica prima di procedere in Fase 7.

Non forzare pattern — segui quello che l'utente descrive + l'esempio competitor.

## Regole trasversali a tutti i tipi

1. **Ogni bottone CTA** punta all'URL PDP raccolto in Fase 6. Nessuna CTA "dummy" o `href="#"`.
2. **Ripeti la CTA** almeno 2 volte nel funnel (metà + fine). Per advertorial lunghi: 3 volte.
3. **Mobile-first**: tutte le sezioni devono leggersi bene a 375px di larghezza. Testi non troncati, bottoni a tutta larghezza.
4. **Chromeless = full-width immersive**: senza header/footer Shopify, le sezioni hero devono riempire tutto il viewport. Se il template ha header/footer, hero più compatto (80-100vh).
5. **Niente Tailwind né framework CSS esterni**: solo CSS scoped dentro la sezione, class-prefix = il prefisso della sezione (es. `.aa-01 .hero__title`).
