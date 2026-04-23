# Auth pattern — Shopify Theme Access token

## Come è fatto il `.env` di uno store

Ogni cartella store ha un file `.env` locale (MAI nel plugin git):

```
SHOPIFY_CLI_THEME_TOKEN=shptka_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
SHOPIFY_FLAG_STORE=<store-handle>.myshopify.com
```

- `SHOPIFY_CLI_THEME_TOKEN`: token personale generato dall'app **Theme Access** (Shopify Admin → Apps → Theme Access).
- `SHOPIFY_FLAG_STORE`: handle `*.myshopify.com` dello store (non il dominio custom).

## Come caricare il `.env` nei comandi CLI

Pattern standard usato in tutti i comandi `theme push` / `theme pull`:

```bash
cd "<workdir-path>"
set -a
source "<env-path>"
set +a
npx @shopify/cli@latest theme <comando> ...
```

- `set -a` → esporta automaticamente tutte le variabili lette dopo.
- `source "<env-path>"` → legge il file `.env`.
- `set +a` → riattiva il comportamento normale.

Dopo il blocco `set -a/+a`, le variabili `SHOPIFY_CLI_THEME_TOKEN` e `SHOPIFY_FLAG_STORE` sono esportate nell'ambiente del processo, e `@shopify/cli` le legge automaticamente senza flag aggiuntivi.

## Verifica connessione

Prima di qualsiasi push, verifica che il token funzioni:

```bash
cd "<workdir-path>"
set -a; source "<env-path>"; set +a
npx @shopify/cli@latest theme list --no-color
```

Output atteso: lista dei temi dello store. Se ricevi `401 Unauthorized` o `Invalid API key`, il token è scaduto o è stato revocato → l'utente deve generarne uno nuovo da Shopify Admin.

## Come generare un nuovo Theme Access token

Istruzioni da dare all'utente quando il token manca o non funziona:

1. Apri l'admin dello store (es. `https://<store>.myshopify.com/admin`).
2. Vai in **Apps** → cerca **"Theme Access"**.
3. Se non è installata, installala (app ufficiale Shopify, gratis).
4. Dentro Theme Access, clicca **"Create password"** (o "Create access").
5. Compila nome e email (la tua email personale, così il token è intestato a te).
6. Shopify invia una email con un link → apri il link → copi il token (inizia con `shptka_`).
7. Incolla il token nel `.env` alla variabile `SHOPIFY_CLI_THEME_TOKEN`.

**Importante**: il token è visibile una sola volta. Se lo perdi, va rigenerato.

## Perché ogni membro del team ha il suo token

- Audit: ogni modifica al tema risulta tracciata all'account che ha usato il token.
- Revoca: se un membro lascia il team, basta revocare il suo token senza toccare gli altri.
- Zero shared secrets: il token non circola via chat/email/zip.
