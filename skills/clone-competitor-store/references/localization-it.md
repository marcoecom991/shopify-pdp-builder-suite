# Localizzazione ENG → IT

Riferimento per Fase 6.x.5 di `/clone-competitor-store`. Spiega come tradurre i testi del competitor (di solito inglese US) in italiano, mantenendo il **tono di voce** ma adattando ai vincoli normativi e culturali del mercato IT.

## Obiettivo

Output: testi italiani che:
1. Mantengono il **tono di voce** del competitor (urgent / scientific / casual / luxury)
2. Sono **fluenti**, non sembrano traduzioni automatiche
3. Rispettano **best practice IT** per claim cosmetici/integratori (più moderati di quelli US)
4. Adattano **unità di misura, valute, formati** ai canoni italiani

## Mantenimento tono di voce

Il tono di voce estratto in Fase 4 (`brand.tone`) è la bussola. Esempi di tono e relativa traduzione:

### Tono "scientific / clinical"

**ENG**: "Clinically proven to reduce wrinkles by 47% in 28 days"

**IT corretta**: "Testato clinicamente, riduce le rughe del 47% in 28 giorni"

**IT da evitare**: "Dimostrato scientificamente: meno rughe del 47% in 4 settimane!" (troppo claim-y, perde tono clinico)

### Tono "urgent / scarcity"

**ENG**: "Last 24 hours — 70% off ends midnight"

**IT corretta**: "Solo per oggi — 70% di sconto fino a mezzanotte"

**IT da evitare**: "Ultime 24 ore!! Prendilo prima che finisca!!" (troppo informale per un brand serio)

### Tono "luxury / editorial"

**ENG**: "The cream that transforms your skincare ritual"

**IT corretta**: "La crema che trasforma il tuo rituale di bellezza"

**IT da evitare**: "La crema che cambia la tua routine skincare" (mix anglicismi rovinano il tono lusso)

### Tono "casual / friendly"

**ENG**: "Hey gorgeous! Ready to glow?"

**IT corretta**: "Ciao bella! Pronta a brillare?"

**IT da evitare**: "Ehi gorgeous! Pronta per il glow?" (anglicismi inutili)

## Regole sui claim cosmetici/integratori

Il mercato IT è molto più regolamentato del US. Claim che funzionano in US potrebbero essere illegali o problematici in IT.

### Claim cosmetici (creme, sieri, autoabbronzanti, sun care)

Riferimento normativo: **Regolamento UE 1223/2009** + linee guida CPNP.

**SAFE in IT**:
- "Idrata e nutre la pelle"
- "Aiuta a contrastare i segni del tempo"
- "Pelle visibilmente più luminosa"
- "Rinforza la barriera cutanea"
- "Pelle morbida e setosa al tatto"

**RISCHIOSO in IT** (modera o riformula):
- ❌ "Cura le rughe" → ✓ "Aiuta a ridurre l'aspetto delle rughe"
- ❌ "Elimina i pori dilatati" → ✓ "Aiuta a minimizzare visivamente i pori"
- ❌ "Distrugge le cellule morte" → ✓ "Aiuta a rinnovare lo strato superficiale della pelle"
- ❌ "Risultati garantiti in 7 giorni" → ✓ "Risultati visibili dopo le prime applicazioni" (lascia margine)

**VIETATO in IT**:
- "Cura/guarisce/tratta <patologia>" (cosmetico ≠ medicinale)
- "Effetti dimostrati come un farmaco"
- "Sostituisce il dermatologo"

### Claim integratori alimentari

Riferimento normativo: **Regolamento UE 1924/2006** + Direttiva 2002/46/CE + EFSA.

**SAFE**:
- "Contiene Vitamina C che contribuisce alla normale formazione del collagene"  (claim approvato EFSA)
- "Magnesio che contribuisce alla riduzione di stanchezza e affaticamento" (approvato)
- "Integratore alimentare di X" (descrittivo)

**RISCHIOSO**:
- ❌ "Cura l'insonnia" → ✓ "Favorisce un sonno regolare" + claim approvato EFSA su melatonina
- ❌ "Brucia grassi" → ✓ "Supporta il metabolismo dei lipidi" + claim approvato (es. tè verde)

**VIETATO**:
- "Previene/cura <patologia>"
- "Funziona meglio dei farmaci"
- "Approvato dal Ministero della Salute" (non lo dichiara mai)

**Disclaimer obbligatori (sempre)**:
- "Gli integratori non vanno intesi come sostituti di una dieta varia ed equilibrata e di uno stile di vita sano."
- "Tenere fuori dalla portata dei bambini sotto i 3 anni."
- "Non superare la dose giornaliera raccomandata."

Questi disclaimer vanno messi nel footer della PDP/funnel come **paragrafo dedicato** (mai nascosti).

## Adattamenti formato

### Valute

- $ → € (sempre in IT, anche se vendi anche in altri mercati)
- "$29.99" → "29,90 €" (virgola decimale, € a destra con spazio non-breaking)
- "Free shipping over $50" → "Spedizione gratuita sopra i 50 €"

### Unità di misura

- "1 fl oz" → "30 ml"
- "5 oz" → "150 g" (peso) o "150 ml" (liquido — verifica context)
- "6 inches" → "15 cm"
- "30 day money-back guarantee" → "Garanzia 30 giorni soddisfatto o rimborsato"

### Date e orari

- "December 31, 2026" → "31 dicembre 2026"
- "12:00 PM EST" → "Mezzogiorno (ora italiana)" o "12:00 CET"
- "Mon-Fri 9-5 EST" → "Lunedì–venerdì, 9:00–17:00 CET"

### Indirizzi e telefoni

- Format US street address → format IT (Via X, NumeroCivico, CAP Città (Provincia))
- Telefono US (+1 XXX XXX XXXX) → telefono IT (+39 XXX XXX XXXX) o europeo se applicabile

### Recensioni e testimonial

I testimonial del competitor (con foto, nomi, città US) NON vanno copiati. Sono di **clienti reali del competitor**, non nostri. Sostituirli sarebbe falsificazione.

Opzioni:
1. **Lascia placeholder generico** (`Testimonial #1 — sostituire con recensione reale`) — più sicuro
2. **Genera testimonial neutri/generici** ("Una clienta di Milano, 45 anni, dopo 4 settimane...")
3. **Rimuovi la sezione** se l'utente non ha testimonianze proprie
4. **Chiedi all'utente** se ha già testimonial autentici da sue clienti

In generale: meglio NIENTE testimonial che testimonial finti. Le clienti italiane sono diffidenti e rilevano il falso facilmente.

## Frasi feel-good che funzionano in IT

Da inserire come default nei `setting` schema dove serve:

### Hero / Headlines

- "Il segreto per [risultato desiderato] anche [nuance: sin dalla prima applicazione, in modo naturale, senza compromessi]"
- "La [tipo prodotto] che [valore unico]"
- "Cosa cambia se [pain point]: [solution]"

### CTA buttons

- "Scopri di più" (neutro, sicuro)
- "Ordina ora" (transactional)
- "Aggiungi al carrello" (e-com standard)
- "Inizia ora" (per servizi/lead)
- "Ricevi a casa" (enfasi spedizione)

❌ Da evitare: "Click here", "Buy now", "Get yours" (suonano traduzioni automatiche)

### Trust badges

- "Spedizione gratuita"
- "Soddisfatto o rimborsato"
- "Pagamenti sicuri"
- "Spedizione tracciata"
- "Risultati provati"
- "Ingredienti naturali" (se vero — non barare)

### Urgency / Scarcity

- "Solo oggi" (non "Last day")
- "Offerta limitata" (non "Limited offer")
- "Ultimi pezzi disponibili"
- "Affrettati, scorte in esaurimento"
- "Promo valida fino a [data]"

❌ Da evitare: "Hurry up!", "Don't miss out!", contatori countdown finti.

### Social proof numerici

- "Oltre 50.000 clienti soddisfatte"
- "Più di 10.000 recensioni a 5 stelle"
- "1 prodotto venduto ogni 30 secondi"

⚠️ I numeri devono essere **veri o plausibili**. Se non li hai, non barare. Sostituisci con testimonianza qualitativa o rimuovi.

## Workflow operativo per la skill

### Per ogni testo del competitor da tradurre

1. **Identifica il tono dominante** (da `brand.tone` di Fase 4)
2. **Traduci letteralmente** come baseline mentale (per non perdere niente)
3. **Riformula in IT idiomatico** rispettando il tono
4. **Verifica claim**: se è un claim cosmetico/integratore, applica le regole sopra
5. **Adatta numeri/unità/valute**
6. **Inserisci come `default`** del setting nello schema

### Quando un claim è insostenibile

Esempio: il competitor dice "Eliminate dark circles in 7 days — 100% of users saw results".

In IT non puoi dire questo (claim 100% non sostenibile, "elimina" è troppo forte per un cosmetico).

Riformula a:
- "Riduci visibilmente le occhiaie in 4-6 settimane di uso costante"
- (Rimuovi il "100% of users" oppure sostituisci con un dato reale che hai)

Comunica all'utente nella riepilogo finale di Fase 8: "Ho moderato N claim per conformità normativa IT. Verifica e dimmi se va bene così o se hai dati specifici da inserire."

## Tool / risorse esterne (per riferimento)

- **EFSA Database** — https://www.efsa.europa.eu/en/data — claim approvati per integratori
- **Cosmetics Europe** — linee guida claim cosmetici
- **Codice del consumo IT** — Decreto Legislativo 206/2005 — pubblicità ingannevole

(La skill non chiede all'utente di consultarli — moderiamo i claim in autonomia secondo le regole sopra. L'utente è libero di sovrascrivere se ha dati specifici autorizzati.)

## Disclaimer finale (FOOTER)

In ogni footer di pagina (PDP, funnel, home), inserisci come paragrafo piccolo:

```
Le informazioni contenute in questo sito hanno scopo informativo e non sostituiscono il parere di un medico o specialista. <product> è un prodotto cosmetico/integratore alimentare e non è destinato a diagnosticare, curare o prevenire alcuna patologia.
```

Adatta in base al tipo prodotto:
- **Cosmetico** → "...prodotto cosmetico..."
- **Integratore** → "...integratore alimentare. Non sostituisce una dieta varia ed equilibrata..."
- **Dispositivo medico** → richiede disclaimer regolamentare specifico (CE marking, ecc.) — chiedi all'utente

## Note operative

- Mai usare **traduzione automatica letterale** (Google Translate). I claim cosmetici/integratori richiedono attenzione che un traduttore automatico non ha.
- Quando un'espressione US non ha equivalente diretto in IT, **adatta il senso**, non le parole.
- Se l'utente vuole un tono "molto US-style" (pushy, scarcity, urgency), avvisa: "Questo tono converte molto su mercato IT segmento giovane/social, ma rischio di sembrare un fake. Vuoi confermare?"
- Ogni testo IT deve poter essere letto a voce alta da un madrelingua italiano senza inciampi.
