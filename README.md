# Nardi Technical Assistant

> Assistente tecnico AI per il supporto clienti dei compressori e booster Nardi Compressori S.r.l.

Single-page web application che fornisce supporto tecnico in tempo reale ai clienti utilizzando un agente AI alimentato dai manuali ufficiali e dai documenti tecnici Nardi. Quando una domanda non trova risposta nei documenti caricati, l'assistente applica formule e standard ingegneristici noti (es. dimensionamento elettrico) per dare comunque una risposta utile. Per domande molto specifiche (codici ricambio, optional, configurazioni dedicate per matricola), genera automaticamente un riepilogo email da inviare a `service@nardicompressori.com`.

## Caratteristiche

- **Chatbot multilingua** (IT, EN, DE, FR, ES, PT, NL)
- **Knowledge Base configurabile** via manifest JSON con documenti separati per linea prodotto (CM, CB, BM, CNG, BON2)
- **Retrieval contestuale** dei documenti più rilevanti per ogni domanda
- **Tre modalità di risposta:**
  - Risposta diretta dai manuali (con citazione fonti)
  - Calcolo ingegneristico per domande risolvibili con formule standard
  - Generazione automatica ticket email per domande che richiedono supporto umano
- **Single-file HTML** identico per stile alle altre app Nardi (Industry Calculator, BON2 Sizing Tool)
- **Deploy su GitHub Pages** in pochi minuti, senza backend

## Struttura del progetto

```
nardi-assistant/
├── index.html                       # Applicazione completa (single-file)
├── README.md                        # Questo file
├── .github/
│   └── workflows/
│       └── deploy.yml              # GitHub Actions per deploy su Pages
└── knowledge-base/
    ├── manifest.json                # Indice dei documenti tecnici
    └── docs/
        ├── general/                 # Documenti trasversali
        │   ├── safety.txt
        │   ├── maintenance-intervals.txt
        │   ├── oil-spec.txt
        │   ├── electrical-sizing.txt
        │   └── spare-parts.txt
        ├── CB/                      # Documenti specifici linea CB
        │   ├── manual.txt
        │   └── troubleshooting.txt
        ├── BON2/                    # Documenti specifici linea BON2
        │   ├── manual.txt
        │   └── sizing.txt
        ├── CM/                      # Da popolare
        ├── BM/                      # Da popolare
        └── CNG/                     # Da popolare
```

## Setup rapido (locale)

```bash
git clone https://github.com/<utente>/nardi-assistant.git
cd nardi-assistant
python3 -m http.server 8000
# Apri http://localhost:8000
```

Al primo avvio cliccare sull'icona Impostazioni (in alto a destra) e configurare:

1. **API Key** Anthropic (richiesta per le risposte AI)
2. **Modello** (default: `claude-sonnet-4-6`)
3. **Knowledge Base URL** (default: `./knowledge-base/manifest.json`)
4. **Email Supporto Tecnico** (default: `service@nardicompressori.com`)

> ⚠️ **Sicurezza chiave API**: in modalità "browser diretto" la chiave API viene salvata nel `localStorage` del browser dell'utente. Questo è accettabile per uso **interno Nardi** o in contesti controllati, ma **NON è adatto per un'app pubblica esposta ai clienti**. Vedi sezione "Deploy in produzione".

## Deploy su GitHub Pages (uso interno)

1. Crea un repository su GitHub e fai push del codice
2. Vai in `Settings` → `Pages`
3. Sotto `Build and deployment`, seleziona `GitHub Actions`
4. Il workflow `.github/workflows/deploy.yml` pubblica automaticamente l'app
5. L'app sarà disponibile su `https://<utente>.github.io/<repo>/`

## Deploy in produzione (clienti esterni)

Per esporre l'app ai clienti senza esporre la chiave API è necessario un **proxy backend** che inietti la chiave server-side. Le opzioni più rapide:

### Opzione A — Cloudflare Worker (consigliato, gratis fino a 100k richieste/giorno)

```javascript
// worker.js — esempio minimo
export default {
  async fetch(request, env) {
    if (request.method !== 'POST') {
      return new Response('Method not allowed', { status: 405 });
    }
    // Forward al servizio Anthropic con la chiave server-side
    const body = await request.text();
    const r = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'x-api-key': env.ANTHROPIC_API_KEY,
        'anthropic-version': '2023-06-01'
      },
      body
    });
    return new Response(r.body, {
      status: r.status,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Headers': 'Content-Type'
      }
    });
  }
};
```

Nelle Impostazioni dell'app inserire l'URL del Worker (es. `https://nardi-ai.tuo-account.workers.dev/`) e lasciare il campo API Key vuoto.

### Opzione B — Vercel / Netlify Functions

Stesso pattern: una function serverless che riceve il body, aggiunge la chiave API dalle env-vars del provider, e fa proxy a `api.anthropic.com`.

### Rate limiting e abuso

Per app pubblica considerare anche:
- Rate limiting per IP nel Worker
- Captcha/turnstile prima del primo messaggio
- Limite massimo di token per richiesta
- Logging delle conversazioni per monitoraggio qualità (rispettando GDPR)

## Aggiungere documenti alla Knowledge Base

### Metodo 1 — Documenti come file separati

1. Aggiungi il file di testo in `knowledge-base/docs/<prodotto>/`
2. Aggiungi una voce in `knowledge-base/manifest.json`:

```json
{
  "id": "cb-electrical-schemas",
  "product": "CB",
  "title": "Schemi elettrici linea CB",
  "type": "schemas",
  "file": "docs/CB/electrical-schemas.txt"
}
```

### Metodo 2 — Contenuto inline

Per documenti brevi puoi inserire il contenuto direttamente nel manifest:

```json
{
  "id": "warranty-terms",
  "product": "*",
  "title": "Termini di garanzia",
  "type": "warranty",
  "content": "La garanzia copre 24 mesi dalla data di consegna..."
}
```

### Convenzioni manifest

- `id`: identificatore univoco (kebab-case). Usato nelle citazioni delle fonti
- `product`: codice linea prodotto (`CM`, `CB`, `BM`, `CNG`, `BON2`) oppure `*` per documenti generali
- `title`: titolo mostrato nelle "Fonti consultate" sotto le risposte
- `type`: categoria (`manual`, `safety`, `maintenance`, `troubleshooting`, `engineering`, `consumables`, `process`, `schemas`, `warranty`)
- `content` oppure `file`: il contenuto del documento (testo) o il path relativo a un file di testo

### Conversione manuali PDF in testo

Per convertire i manuali PDF esistenti in testo per la KB:

```bash
# Installazione tool (una volta)
brew install poppler  # macOS
# oppure
sudo apt install poppler-utils  # Linux

# Conversione PDF → testo
pdftotext -layout manuale-CB.pdf manuale-CB.txt
```

Per PDF con immagini/scansioni servirà OCR (Tesseract). Considerare di mantenere ridotti e mirati i documenti: la KB funziona meglio con testi puliti e ben strutturati piuttosto che con manuali completi.

## Logica del chatbot — come funziona

Quando l'utente invia una domanda:

1. **Recupero contesto**: vengono cercati i 4 documenti più rilevanti nella KB tramite scoring TF (token overlap normalizzato), filtrati per linea prodotto se selezionata
2. **Costruzione prompt**: i documenti rilevanti vengono iniettati nel system prompt, insieme alle istruzioni operative
3. **Chiamata API Claude**: la conversazione viene inviata all'endpoint configurato (Anthropic diretto o proxy)
4. **Parsing risposta**: il prompt istruisce il modello a chiudere ogni risposta con `[SOURCES: id1, id2]` (per risposte da KB o da formule) oppure con `[TICKET_NEEDED]` seguito da `[TICKET_SUMMARY]` per domande che richiedono escalation
5. **Rendering**: la risposta viene mostrata nel chat con i chip delle fonti citate, oppure compare la card per aprire il ticket

### Categorie di risposta

| Categoria | Trigger | Azione |
|---|---|---|
| **A — Da KB** | La risposta è nei documenti caricati | Cita i documenti consultati |
| **B — Calcolo ingegneristico** | La domanda si risolve con formule note (dimensionamento, calcoli standard) | Mostra la formula e il calcolo |
| **C — Specifica Nardi non disponibile** | Codice ricambio specifico, optional, dato matricola, ecc. | Genera ticket per `service@nardicompressori.com` |
| **D — Best practice di settore** | Domanda generica risolvibile con conoscenze tecniche standard | Risposta conservativa con disclaimer |

## Personalizzazioni

### Cambiare logo e branding
Modifica il blocco `<svg>` dentro `.brand-mark` nel file `index.html` per inserire il logo Nardi originale (preferibilmente come SVG embedded).

### Aggiungere una linea prodotto
Modifica il campo `products` nel `manifest.json`. La linea apparirà automaticamente nella sidebar.

### Cambiare colori
I colori sono centralizzati nelle CSS variables in cima all'`index.html` (sezione `:root`). Modificare `--accent`, `--bg`, ecc. per ribrandizzazione.

### Cambiare testi predefiniti / aggiungere lingue
La sezione `I18N` nel JavaScript contiene tutti i testi UI. Per aggiungere una lingua, replicare la struttura di `it` o `en`.

## Tecnologie

- HTML5 + CSS3 + JavaScript vanilla (zero dipendenze runtime)
- Font Manrope (Google Fonts)
- API Anthropic Claude (configurabile)
- GitHub Pages compatibile (statica al 100%)

## Licenza e proprietà

Codice di proprietà di Nardi Compressori S.r.l. — uso interno e per i clienti autorizzati.

## Supporto e contributi

Per modifiche, segnalazioni di bug o nuove funzionalità contattare il team interno Nardi.

Per supporto tecnico ai clienti finali: `service@nardicompressori.com`.
