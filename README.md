# CyberCoach Config

Configurazione del **system prompt** di CyberCoach, l'assistente AI dentro l'app Plannest Coach.

L'unico file che conta qui dentro è [`system-prompt.md`](./system-prompt.md): contiene le istruzioni che il modello Anthropic legge a ogni conversazione. Modificarlo significa cambiare il comportamento del CyberCoach **per tutti gli utenti**, in tempo reale (massimo qualche minuto di propagazione).

## Come modificarlo (workflow consigliato)

### Opzione 1 — Tuning rapido (qualche frase)

1. Vai direttamente su [`system-prompt.md`](./system-prompt.md) → clicca l'icona della **matita** in alto a destra.
2. Modifica il testo.
3. In fondo alla pagina, scrivi un breve messaggio di commit (es. *"chiarito comportamento sui pacchetti scaduti"*).
4. Clicca **Commit changes**.

### Opzione 2 — Riscrittura ampia con l'aiuto di Claude

1. Apri [claude.ai](https://claude.ai).
2. Apri [`system-prompt.md`](./system-prompt.md), clicca **Raw** → seleziona tutto → copia.
3. In Claude scrivi qualcosa tipo:
   > Questo è il system prompt del nostro chatbot CyberCoach. Voglio che tu mi aiuti a [chiarire la sezione X / aggiungere una regola Y / migliorare il tono]. Mantieni invariati i placeholder `{{LANGUAGE}}`, `{{TODAY}}`, `{{CONTEXT}}` e l'intero blocco "Componenti ricchi". Restituiscimi il file completo.
4. Incolla il prompt sotto.
5. Itera con Claude finché sei soddisfatto.
6. Copia la versione finale.
7. Torna su GitHub → editi `system-prompt.md` → cancella tutto il contenuto → incolla → **Commit changes**.

## Regole importanti

⚠️ **Tre placeholder devono restare presenti**, altrimenti il bot si rompe:

- `{{LANGUAGE}}` — viene sostituito con "italiano" o "inglese" in base alla lingua dell'utente.
- `{{TODAY}}` — viene sostituito con la data corrente (es. "2026-05-06").
- `{{CONTEXT}}` — viene sostituito con la lista clienti/servizi/location/ferie del coach loggato.

Se per errore li cancelli, basta committare un fix successivo per ripristinarli.

⚠️ **Niente codice JavaScript, JSON, backtick di chiusura** o roba simile. È solo testo markdown leggibile. Se vedi `${...}`, `export const`, o backtick singolo `` ` ``, qualcosa è andato storto: rivedi prima di committare.

⚠️ **Niente dati personali hardcodati** (nomi clienti, email, ID specifici). Sono già iniettati a runtime via `{{CONTEXT}}`.

## Come verificare che la modifica è live

1. Aspetta 1-2 minuti dopo il commit (cache CDN GitHub).
2. Apri il CyberCoach (o ricarica la pagina dell'app Plannest Coach).
3. Manda un messaggio che testa il nuovo comportamento.

Se vuoi essere certo della versione attiva, apri la console del browser e digita:
`fetch('https://raw.githubusercontent.com/plannest/cybercoach-config/main/system-prompt.md?t=' + Date.now())
.then(r => r.text())
.then(t => console.log(t.slice(0, 200)))`


## Rollback

Sbagliato qualcosa? Vai sulla [history dei commit](../../commits/main/system-prompt.md), clicca quello buono precedente, copia il contenuto, incollalo nel file, committa.

Oppure dal terminale (per chi sa usare git):
`git revert <commit-hash>
git push`

## Architettura

- Il bundle JS del CyberCoach (`cybercoach.min.js`, repo `plannest/cybercoach`) all'avvio fetcha `system-prompt.md` da questo repo via `raw.githubusercontent.com`.
- Sostituisce i placeholder a runtime con dati per-utente.
- Manda il prompt risultante ad Anthropic (Claude) ad ogni messaggio.
- Se questo file non è raggiungibile, il bundle ha una copia baseline incorporata come fallback (il bot non si rompe).

Per modifiche al **codice** del CyberCoach (UI, tool, engine), vai su [plannest/cybercoach](https://github.com/plannest/cybercoach).
