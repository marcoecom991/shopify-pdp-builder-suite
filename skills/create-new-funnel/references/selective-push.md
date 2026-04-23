# Selective push — come pubblicare solo i file modificati

Regola: **non fare mai un push completo**. Rischi di sovrascrivere file di altri template o di azzerare lavoro non sincronizzato. Usa sempre `--only` per pubblicare solo i file che hai effettivamente toccato.

## Comando standard

```bash
cd "<workdir-path>"
set -a; source "<env-path>"; set +a
npx @shopify/cli@latest theme push \
  --theme <THEME_ID> \
  --nodelete \
  --allow-live \
  --only "sections/<file1>.liquid" \
  --only "sections/<file2>.liquid" \
  --only "templates/product.<nome>.json"
```

## Flag spiegate

- `--theme <ID>`: theme ID numerico del tema target (preso da `stores.json` per lo store corrente). Evita ambiguità se lo store ha più temi.
- `--nodelete`: NON cancella file remoti che non esistono localmente. **Sempre presente.** Senza questo flag, un push selettivo potrebbe rompere altri template che non hai in locale.
- `--allow-live`: permette di pushare sul tema pubblicato (il tema live dello store). Senza, Shopify CLI blocca il push per sicurezza. Usalo solo quando sei sicuro.
- `--only "<path>"`: include solo quel file nel push. Ripetibile per più file.

## Pattern per tipo di cambiamento

### Hai modificato UNA sezione
```bash
--only "sections/<prefisso>-<suffix>.liquid"
```

### Hai creato un template nuovo + sezioni nuove
```bash
--only "templates/product.<nome>.json" \
--only "sections/<prefisso>-01.liquid" \
--only "sections/<prefisso>-02.liquid" \
# ... ripeti per tutte le sezioni duplicate
```

### Hai modificato un snippet condiviso
```bash
--only "snippets/<nome-snippet>.liquid"
```

**Attenzione**: se modifichi un snippet usato anche da altri template (es. berberina vs crema-occhiaie), il cambio impatta anche quelli. Duplica il snippet con un nuovo nome se vuoi isolare.

## Cosa NON fare

- ❌ `theme push` senza `--only` → push completo, rischio di sovrascrivere.
- ❌ `theme push --only "sections/*"` → glob pattern, include TUTTE le sezioni (anche quelle non toccate).
- ❌ Push diretto a `main` theme senza `--allow-live` non previsto dallo script.
- ❌ Omettere `--nodelete` → può cancellare file remoti non presenti in locale.

## Verifica dopo il push

Output atteso:
```
╭─ success ─...
│  The theme '<NOME TEMA>' (#<THEME_ID>) was pushed successfully.
```

Se ricevi un errore di validazione Liquid, il file NON è stato pushato. Correggi il Liquid e rilancia. Shopify valida ogni file prima di applicarlo.
