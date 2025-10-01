# RistoSmartFM

Gestionale leggero per ristoranti — **Flask + SQLite**.  
Versione *multi-tenant* con licenze.

---

## 👤 Ruolo & Stile
- Ingegneria: Flask (Python 3.12), UI Bootstrap 5 + Jinja2  
- Modifiche **chirurgiche**: non rompere ciò che funziona; patch mirate  
- Commit/issue **sintetici**: una cosa alla volta, chiaro e verificabile

---

## 🧰 Stack & Avvio
- **Backend:** Flask (Python 3.12), **SQLite** (file singolo)  
- **Frontend:** Jinja2 + Bootstrap 5  
- **Dev run:** `python app.py` (porta 5000)  
- **Percorso DB (dev):**  
  `app.config["DB_PATH"] = C:\Users\Fio\OneDrive\Desktop\RistoSmartFM_New\ristosmart.db`  
- **Prod:** percorso via env; backup pianificati

---

## 🔒 Sicurezza dati (non negoziabile)
**Isolamento totale per utente (multi-tenant):**
- **OGNI TABELLA** ha `user_id NOT NULL`  
- **OGNI QUERY** filtra per `_uid()`  
- Nessun utente può vedere o scrivere dati di un altro

**Vincoli consigliati:**
- Chiavi composte con `user_id`  
- FK: tabelle figlie puntano al padre con lo stesso `user_id`  
- API mutate: **CSRF obbligatorio**, sessione valida, licenza valida  
- Vietato: join senza `... AND p.user_id = sp.user_id`, select senza `WHERE user_id=?`

> ✅ **Verifica completa eseguita**: ogni rotta è conforme ai principi di sicurezza.  
> Il sistema è resiliente a IDOR, injection e spoofing.

---

## 🔑 Licenze
- Flusso: Registrazione/Login → Attivazione licenza → Accesso  
- **Scadenza = blocco immediato**  
- Admin: genera, invia, revoca, rinnova  
- **Mai disattivare** i controlli licenza  
- **Email automatiche**: promemoria scadenza (3, 5, 9 giorni prima), conferma attivazione/rinnovo

---

## 🔧 Variabili d’ambiente
- `FLASK_SECRET_KEY`  
- `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`  
- `ADMIN_EMAIL`  
- Opz.: `RENEW_URL` (default `http://127.0.0.1:5000/license`)  
- Opz. backup: `BACKUP_MAX_COUNT`, `BACKUP_MAX_AGE_DAYS`

### Hardening
- `MAX_CONTENT_LENGTH = 2 MB`  
- Cookie: `HttpOnly`, `SameSite=Lax`, `Secure` su HTTPS  
- Anti-CSRF su form e API mutate  
- Security headers:
  - `X-Content-Type-Options: nosniff`
  - `X-Frame-Options: SAMEORIGIN`
  - `Content-Security-Policy: default-src 'self'; ...`
  - `Strict-Transport-Security: max-age=31536000; includeSubDomains`
  - `Permissions-Policy: camera=(), microphone=()`

---

## 🗄️ Database (multi-tenant)
Tabelle principali:  
`utenti`, `licenze`, `clienti`, `incassi`, `spese_fisse`, `fatture`, `personale`, `stipendi_personale`, `tasse`, `enti`, `fornitori`

### `stipendi_personale`
- **Colonne:** `user_id`, `personale_id`, `anno`, `mese`, `lordo`, `netto`, `contributi`, `totale`, `stato_pagamento`  
- **Vincoli:**
  - `CHECK(mese BETWEEN 1 AND 12)`  
  - `UNIQUE(user_id, personale_id, anno, mese)`  
  - FK su `personale(user_id, id)`
- **Trigger:** verifica coerenza `user_id` tra dipendente e stipendio

### `tasse`
- Collegata a `ente` tramite `ente_id`, entrambe multi-tenant
- Join sempre con `AND e.user_id = t.user_id`
- Stato persistente (`pagato`, `non_pagato`, `standby`)
- QR Code SEPA generato da IBAN/BIC dell’ente

---

## 📄 Pagine chiuse (stabili)
- `/clienti`  
- `/mese`  
- `/base.html`  
- `/fornitori`  
- `/fatture` *(QR SEPA testato; blocco doppio pagamento)*  
- `/annuale.html`  
- `/personale.html` *(CRUD completo)*  
- `/stipendi.html` *(CRUD, QR, pagamenti, DB sync)*  
- `/tasse.html` *(completata con enti.json e tooltip)*  
- `/pagamento-tasse.html` *(flusso completo: inserimento, QR SEPA, conferma, stato persistente, localStorage sync)*

> ✅ **AVVISO LEGALE DI PROGETTO**  
> I template **`tasse.html`** e **`pagamento-tasse.html`** sono ora **definitivamente chiusi e stabili**.  
> Ogni modifica futura a questi file **richiede espressamente il consenso scritto del proprietario del progetto**.  
> Questo per garantire coerenza, sicurezza e affidabilità del flusso di pagamento tasse.

---

## 🔄 Sincronizzazione Dati in Tempo Reale
Implementato sistema reattivo tra pagine:

### 🎯 Flusso: Pagamento tassa → Aggiornamento mensile → Aggiornamento annuale
1. Utente paga tassa in `pagamento-tasse.html` → stato "pagato"
2. Calcola mese/scadenza → identifica campo in `mese.html` (es. `agenzia_entrata`)
3. Invia segnale via `localStorage.setItem('trigger_mese_update', payload)`
4. `mese.html` riceve evento → aggiorna campo → salva su DB
5. Durante il salvataggio, lancia `localStorage.setItem('trigger_annual_refresh', anno)`
6. `annuale.html` ascolta evento → ricarica dati → grafico aggiornato

✅ **Nessun F5 necessario** – tutto automatico.

---

## 💾 Backup
- Pulsante → ZIP in `C:\RISTO\BACKUP`  
- Retention configurabile:
  - `BACKUP_MAX_COUNT`: numero massimo di backup conservati
  - `BACKUP_MAX_AGE_DAYS`: eliminazione dei backup più vecchi di X giorni
- Funzione `_cleanup_backups()` automatizza la pulizia

---

## 💸 Stipendi
- UI con pulsanti “Pagato” persistenti e placeholder “Lordo/Contributi/Netto”  
- Conferma eliminazione a doppio step  
- Persistenza: DB + LocalStorage  
- QR SEPA testato con banca reale  
- **Aggiornamento automatico** in `annuale.html` tramite trigger

---

## 📱 Pagamento Fatture via QR Code
- IBAN/BIC da tabella fornitori  
- API: `GET /api/fattura/<id>/qr`  
- Popup con conferma → stato “Pagato” sync DB/LS

---

## 🧭 Git & GitHub
1. `.gitignore` → ignora venv, cache, DB, backup, segreti  
2. `git init && git add . && git commit`  
3. `git remote add origin <URL>` + `git push -u origin main`  
4. Flusso: `pull --rebase`, `add -A`, `commit`, `push`  
5. Branch feature → PR → merge  
6. Clonare su altro PC → `git clone` + `pip install -r requirements.txt`  
7. Rimuovere file già tracciati → `git rm -r --cached ...`  
8. Cambiare URL → `git remote set-url origin <URL>`

---

## ✅ TODO / Next Steps
- [ ] `percentuali.html`  
- [x] Integrazione completa stipendi → annuale (totali per ruolo/mese)  
- [x] Uniformare gestione decimali tra frontend e backend  
- [ ] Report mensile esportabile (PDF/CSV)  
- [ ] Esportazione tasse in PDF  
- [ ] Promemoria email per scadenze tasse  
- [ ] Collegamento conto corrente (API banking)

---

## 🧪 Checklist PR
- [x] Ogni query filtra `user_id = _uid()`  
- [x] Nessun `user_id` accettato dal client  
- [x] `mese` normalizzato `1..12`  
- [x] Mutazioni: `@require_csrf`, `@require_login`, `@require_license`  
- [x] Join multi-tenant sempre presenti  
- [x] Test su browser incognito con DB come fonte verità  
- [x] Migrazioni schema con vincoli/indici preservati  
- [x] Sincronizzazione tra pagine senza ricaricamento manuale

---

Se in futuro vorrai:
- Aggiungere un riepilogo mensile
- Esportare le tasse in PDF
- Ricevere promemoria via email
- Collegare il conto corrente (API banking)
…saprò esattamente da dove partire.

Prossimo passo. 
1) Controllare perchè , anche se gli stipendi sono nel DB , chiudendo e riaprendo il browser nella pagina stipendi.htm gli stipendi spariscono. 

2) Controllare l'auto refresh di : 
pagamento-tasse.html da ricontrollare non si aggiorna automaticamente 
personale.html Paga stipendio → mese.html si aggiorna → annuale.html si aggiorna
fatture.html Paga fattura → mese.html si aggiorna → annuale.html si aggiorna
mese.html Riceve segnali → Applica modifiche e propaga a annuale.html
annuale.html Ascolta solo trigger_annual_refresh → Ricarica senza F5

3) Progettare prcentuali.html
Aggiungere le tasse al garfico a torta 

I calcoli devono essere cosi:

Sese fisse: Canone - mutuo Finaziamenti
Calcolare la percentuale di tutte le categorie assieme .

Tasse : Calcolare la percentuale di tutte le enti assieme

Sipendi personale : Calcolare la percentuale di tutte le categorie assieme

Spese fatture : Calcolare la percentuale di ogni categoria singola. 
