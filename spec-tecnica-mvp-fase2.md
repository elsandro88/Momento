# Specifica tecnica — MVP "Social payment emozionale"
### Fase 2 del Piano operativo 90 giorni — blueprint per lo sviluppo

---

## 1. Obiettivo e perimetro

Costruire una web app mobile-first (no app nativa) che copre il flusso:

**scegli persona → scegli gesto → messaggio → paga → condividi link → il destinatario apre pagina web → sceglie locale → mostra QR → il locale conferma → settlement**

Perimetro Fase 2 (da costruire ora):
- Pagina di invio (mittente)
- Pagina di ricezione pubblica (destinatario, no login)
- Dashboard locale minima
- Notifica al mittente al riscatto (email)
- Pagamenti con Stripe Connect (split automatico)

Esplicitamente FUORI perimetro (vedi sezione 10 del piano operativo): app nativa, push proprietarie, feed sociale, wallet, gruppi, WhatsApp Business API.

---

## 2. Stack consigliato

| Livello | Scelta | Motivo |
|---|---|---|
| Frontend + Backend | **Next.js 14 (App Router)** | Un solo repo per pagine pubbliche, API routes e dashboard; deploy semplice su Vercel |
| Database | **Postgres via Supabase** | Auth pronta per la dashboard locali, Row Level Security, hosting gestito |
| Pagamenti | **Stripe Connect (Standard o Express accounts)** | Split automatico piattaforma/locale, gestisce KYC dei locali |
| QR code | **libreria `qrcode` (npm)** generazione lato server + `jsQR` o scanner nativo browser per la lettura | Nessuna infrastruttura dedicata |
| Email | **Resend** o **Amazon SES** | Costo bassissimo per notifiche di riscatto |
| Hosting | **Vercel** (frontend/API) + **Supabase Cloud** (DB) | Zero DevOps nella fase MVP |

---

## 3. Schema del database (Postgres / Supabase)

```sql
-- Locali partner
create table locali (
  id uuid primary key default gen_random_uuid(),
  nome text not null,
  indirizzo text,
  citta text not null,
  stripe_account_id text, -- ID Stripe Connect del locale
  stato text default 'attivo', -- attivo, sospeso
  created_at timestamptz default now()
);

-- Prodotti disponibili per locale (caffè, drink, aperitivo...)
create table prodotti (
  id uuid primary key default gen_random_uuid(),
  locale_id uuid references locali(id),
  categoria text not null, -- 'caffe', 'drink', 'aperitivo', 'primo_giro'
  nome text not null,
  prezzo_locale_cents int not null, -- quanto riceve il locale
  fee_servizio_cents int not null,  -- quanto trattiene la piattaforma
  attivo boolean default true
);

-- Utenti (mittenti) - auth leggera, anche solo email/telefono
create table utenti (
  id uuid primary key default gen_random_uuid(),
  nome text,
  email text,
  telefono text,
  created_at timestamptz default now()
);

-- Il "gesto" inviato
create table gesti (
  id uuid primary key default gen_random_uuid(),
  mittente_id uuid references utenti(id),
  destinatario_nome text not null, -- solo nome, no account obbligatorio
  destinatario_telefono text,
  destinatario_email text,
  prodotto_id uuid references prodotti(id),
  messaggio text,
  importo_totale_cents int not null,
  stripe_payment_intent_id text not null,
  stato text default 'pagato', -- pagato, riscattato, scaduto, rimborsato
  token_pubblico text unique not null, -- usato nell'URL di ricezione, es. /r/{token}
  scade_il timestamptz, -- es. 30 giorni da creazione
  created_at timestamptz default now()
);

-- Riscatto presso il locale
create table riscatti (
  id uuid primary key default gen_random_uuid(),
  gesto_id uuid references gesti(id) unique,
  locale_id uuid references locali(id),
  qr_code_hash text not null, -- hash monouso, invalidato dopo lo scan
  validato_da text, -- operatore/dispositivo che ha confermato
  riscattato_at timestamptz
);
```

Note:
- `token_pubblico` è l'unico identificativo esposto nell'URL pubblico (`/r/{token}`) — non esporre mai gli id incrementali o gli id Stripe.
- `qr_code_hash` deve essere generato al momento della scelta del locale (non alla creazione del gesto), monouso, con scadenza breve (es. 15 minuti) per limitare la superficie di frode via screenshot.

---

## 4. Flusso pagamenti — Stripe Connect

**Tipo di account consigliato per i locali: Express.**
Onboarding più rapido di Standard, la piattaforma mantiene più controllo sull'esperienza — adatto a un locale che "non deve installare niente".

Flusso split payment:

1. Il mittente paga l'importo totale (prezzo prodotto + fee servizio) con `PaymentIntent`.
2. Si usa `application_fee_amount` = fee di servizio (in cents) e `transfer_data[destination]` = `stripe_account_id` del locale.
3. Stripe accredita automaticamente il locale per la parte prodotto e trattiene la fee per la piattaforma — non serve calcolare o girare manualmente lo split.
4. Il locale riceve i fondi secondo il suo payout schedule Stripe (consigliato: settimanale, per allinearsi al piano dei 90 giorni).

Onboarding locale (una tantum, fuori dal flusso utente):
- Link di onboarding Stripe Express generato dalla dashboard locale al primo accesso.
- Finché l'onboarding non è completo, il locale non compare tra le opzioni di riscatto disponibili per i destinatari.

Gestione contestazioni: dato il rischio evidenziato nel documento di analisi (fee di ~20€ per disputa su Stripe), impostare fin dal MVP un limite di importo massimo per transazione (es. 50€) e monitorare manualmente le prime settimane.

---

## 5. Pagine da costruire

### A. `/invia` — Pagina di invio (mittente)
Step in un unico flusso (no wizard multi-pagina, per restare sotto i 30 secondi indicati nel documento di prodotto):
1. Nome destinatario + telefono o email (per la notifica del link)
2. Scelta gesto: caffè / drink / aperitivo / primo giro (icone, non testo "gift card")
3. Messaggio personale (campo libero, placeholder tipo "Spacca tutto all'esame ☕")
4. Riepilogo prezzo: "Caffè 1,50€ + 0,80€ per farlo arrivare subito, ovunque sia" (tono da documento di prodotto, mai "commissione")
5. Pagamento (Stripe Elements embedded)
6. Schermata di conferma con link generato e pulsante "Condividi su WhatsApp" (`wa.me/?text=`)

### B. `/r/[token]` — Pagina di ricezione (destinatario, pubblica, no login)
1. Messaggio del mittente in evidenza: "Marco ti ha offerto un caffè" + testo personale
2. Lista locali disponibili per quel prodotto (filtrati per città/quartiere del pilota)
3. Bottone "Riscatta qui" → genera QR code monouso con scadenza breve
4. Dopo lo scan confermato dal locale: schermata di ringraziamento + CTA "Ricambia" (crea un nuovo gesto pre-compilato verso il mittente originale)

### C. `/locale/dashboard` — Dashboard locale (autenticata, Supabase Auth)
1. Login semplice (magic link email, no password)
2. Campo per inserire/scansionare il codice del QR
3. Conferma riscatto in un tap
4. Storico riscatti della settimana + totale da incassare (riferimento, il payout è automatico via Stripe)

---

## 6. API routes principali (Next.js)

| Route | Metodo | Funzione |
|---|---|---|
| `/api/gesti` | POST | Crea il gesto, crea il PaymentIntent Stripe Connect |
| `/api/gesti/[token]` | GET | Recupera i dati del gesto per la pagina di ricezione |
| `/api/riscatti` | POST | Genera il QR monouso per un gesto + locale scelto |
| `/api/riscatti/conferma` | POST | Il locale conferma lo scan → aggiorna stato, invia notifica email al mittente |
| `/api/stripe/webhook` | POST | Ascolta `payment_intent.succeeded`, `charge.dispute.created` |
| `/api/locali/onboarding` | POST | Genera link Stripe Express onboarding per un nuovo locale |

---

## 7. Cosa gestire manualmente all'inizio (per risparmiare tempo di sviluppo)

Coerente con la Fase 0/1 del piano operativo:
- Caricamento locali e prodotti: via inserimento diretto in Supabase (no pannello admin dedicato nel MVP).
- Settlement: monitorabile dalla dashboard Stripe Connect stessa, non serve costruire reportistica custom in Fase 2.
- Assistenza clienti: email diretta, nessun sistema di ticketing.

---

## 8. Ordine di build consigliato (per sessioni con Claude Code)

1. Setup progetto Next.js + Supabase (schema sopra) + variabili ambiente Stripe
2. API `/api/gesti` (POST) + integrazione PaymentIntent Stripe Connect base (senza split, solo pagamento semplice) per validare il flusso pagamento end-to-end
3. Pagina `/invia` collegata all'API
4. Pagina `/r/[token]` in sola lettura (senza QR ancora)
5. Generazione e validazione QR monouso (`/api/riscatti`)
6. Dashboard locale minima + Supabase Auth (magic link)
7. Attivare lo split payment reale (`application_fee_amount` + `transfer_data`) e l'onboarding Express dei locali
8. Webhook Stripe + notifica email al mittente
9. CTA "Ricambia" e rifinitura microcopy secondo il tono di voce del documento di prodotto

---

## 9. Variabili d'ambiente necessarie

```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
STRIPE_SECRET_KEY=
STRIPE_PUBLISHABLE_KEY=
STRIPE_WEBHOOK_SECRET=
RESEND_API_KEY=
NEXT_PUBLIC_BASE_URL=
```

---

## 10. Riferimento al piano operativo

Questa specifica copre la Fase 2 (Settimane 3-8) del Piano operativo 90 giorni. Il gate di uscita resta: *il flusso completo invio → riscatto → settlement funziona end-to-end con almeno un locale reale in test*, prima di passare alla Fase 3 (acquisizione locali su scala) e alla Fase 4 (pilota).
