# Componenti renderizzabili nella chat CyberCoach

Riferimento dei componenti grafici che il modello può mostrare durante una conversazione, e regole di utilizzo.
Questo file è una guida per il team che mantiene [`system-prompt.md`](./system-prompt.md): se aggiungiamo nuove regole o pattern, le scriviamo qui prima e poi le riportiamo nel system prompt.

---

## 1. `render_client_card` — Card cliente / appuntamento

Card tabellare allegata a un messaggio bot. Mostra dati cliente, dettagli appuntamento, stato e prezzo. Esistono due pattern visivi.

### Pattern A — Inline status (compatto)

Senza header. Lo stato sta in basso, accanto al prezzo. Adatto a elenchi di appuntamenti esistenti.

### Pattern B — Header con icona stato

Etichetta in alto + icona colorata: ✓ success, ⏱ warning, ⚠ destructive. Adatto a conferme di azioni (creazione/aggiornamento) e situazioni che meritano risalto visivo.

### Parametri

| Campo | Tipo | Obbligatorio | Note |
|---|---|---|---|
| `client_name` | string | sì | Nome + cognome cliente |
| `client_pic_url` | string | no | URL `photo_thumb_url` se disponibile, altrimenti omettilo (fallback silhouette) |
| `date` | string | no | Display tipo `15/05/2026 — 13:00` |
| `activity` | string | no | Nome servizio |
| `venue` | string | no | Nome location |
| `header.label` | string | no | Es. `Appuntamento prenotato` — la presenza dell'header attiva il Pattern B |
| `header.kind` | enum | no | `success` / `warning` / `destructive` |
| `inline_status.label` | string | no | Es. `Da riscuotere` — la presenza attiva il Pattern A |
| `inline_status.kind` | enum | no | `success` / `warning` / `destructive` |
| `price.current_eur` | string | no | Es. `25,00€` |
| `price.old_eur` | string | no | Es. `38,00€` (mostra come barrato) |
| `recap_rows` | array | no | Righe extra: `{label, value}` o `{label, value_strong}` |

### Quando usarlo

- Dopo `get_events` o `get_bookings` → **Pattern A**, una card per ogni appuntamento.
- Dopo `create_event` riuscita → **Pattern B** con `header.kind="success"` label `Appuntamento prenotato`.
- Dopo `update_event` riuscita → **Pattern B** con `header.kind="success"` label `Appuntamento aggiornato`.
- Per appuntamenti già passati: Pattern A con `inline_status` adeguato (`Svolto`, `Pagato`, `Da riscuotere`, `Cancellato`).

### Quando NON usarlo

- Per `delete_event`: usa `render_status_badge` (sotto).
- Per elenchi enormi (>10 appuntamenti): meglio condensare in un riassunto testuale e offrire di mostrare i dettagli filtrando.

---

## 2. `render_package_card` — Card pacchetto catalogo

Card con bordo blu, header su sfondo grigio chiaro, prezzo grande in basso al titolo, descrizione sotto.

### Parametri

| Campo | Tipo | Obbligatorio | Note |
|---|---|---|---|
| `title` | string | sì | Nome pacchetto |
| `specs` | string | no | Es. `5 Sessioni, 1 Servizio, 1 Mese` |
| `price_eur` | string | sì | Es. `300€` o `25,00€` |
| `description` | string | no | Descrizione marketing |

### Quando usarlo

- Dopo `get_packages` → una card per ogni pacchetto del catalogo.
- Dopo `create_package` riuscita → mostra il nuovo pacchetto come conferma.
- Dopo `update_package` riuscita → mostra la versione aggiornata.

### Quando NON usarlo

- Per `delete_package`: usa `render_status_badge` con kind `destructive`.
- Per pacchetti assegnati a un cliente specifico (`get_client_packages`): non c'è ancora un componente dedicato, usa testo.

---

## 3. `render_status_badge` — Badge di stato standalone

Pallino colorato + label. Inline, non dentro card. Conferma rapida di un'azione completata.

### Parametri

| Campo | Tipo | Obbligatorio | Note |
|---|---|---|---|
| `kind` | enum | sì | `success` (verde) / `warning` (arancio) / `destructive` (rosso) |
| `label` | string | sì | Testo breve, max 4-5 parole |

### Quando usarlo (azioni di scrittura)

| Tool eseguito | Kind | Label suggerita |
|---|---|---|
| `delete_event` | destructive | Appuntamento eliminato |
| `delete_booking` | destructive | Prenotazione cancellata |
| `delete_service` | destructive | Servizio eliminato |
| `delete_package` | destructive | Pacchetto eliminato |
| `delete_holiday` | destructive | Ferie eliminate |
| `delete_reminder` | destructive | Promemoria eliminato |
| `create_client` | success | Cliente aggiunto |
| `create_holiday` / `update_holiday` | success | Ferie aggiornate |
| `create_reminder` / `update_reminder` | success | Promemoria salvato |
| `create_location` / `update_location` | success | Sede salvata |
| `create_room` / `update_room` | success | Stanza salvata |
| `assign_package_to_client` | success | Pacchetto assegnato |
| `update_client_package` | success | Pacchetto aggiornato |
| `create_service` / `update_service` | success | Servizio salvato |

### Quando NON usarlo

- Per operazioni di lettura (`get_*`): non c'è azione da confermare.
- Quando l'azione produce dati abbastanza ricchi per giustificare una card piena (es. `create_event`, `create_package`): preferisci la card.

---

## Mappa colori canonica

| Kind | Colore | Significato | Esempi label |
|---|---|---|---|
| `success` | Verde `#09AE00` | Azione completata, stato positivo | Svolto, Pagato, Appuntamento prenotato, Cliente aggiunto |
| `warning` | Arancione `#D28E28` | In attesa, attenzione, qualcosa da fare | Da riscuotere, Cancellato, In attesa di pagamento, Confermi questi dati? |
| `destructive` | Rosso `#D30000` | Errore grave, eliminazione, fallimento | Cancellato in ritardo, Cliente assente, Appuntamento eliminato |

**Regola critica per i payment_type di Plannest**:
- `cancelled_late` e `no-show` → SEMPRE `destructive`
- `cancelled` semplice → `warning`
- `paid` / `charged` / `completed` → `success`
- `to_collect` → `warning`

---

## Pattern di accompagnamento testuale

Ogni componente renderizzato deve avere **una sola frase breve** sopra o sotto, mai paragrafi.

| Componente | Esempio frase di accompagnamento |
|---|---|
| `render_client_card` (lista) | "Ecco i tuoi appuntamenti di domani:" |
| `render_client_card` (singola, conferma) | "Fatto. Ecco il riepilogo:" |
| `render_client_card` (singola, lettura) | (nessuna frase, la card parla da sola) |
| `render_package_card` (lista) | "Hai N pacchetti in catalogo:" |
| `render_package_card` (conferma) | "Pacchetto pronto:" |
| `render_status_badge` | (nessuna frase, il badge basta — al massimo "Tutto a posto.") |

### Anti-pattern da evitare

- Niente liste markdown che ripetono i dati già visibili nella card (es. *"Cliente: Mario, Data: domani, Prezzo: 25€"* + card sotto con gli stessi dati).
- Niente paragrafi lunghi che descrivono i campi della card.
- Niente conferme verbose tipo *"Perfetto! Ho creato l'appuntamento con successo. Tutti i dati sono stati salvati correttamente nel sistema."* — basta la card con header success.

---

## Riepilogo decisionale rapido

```
Hai eseguito un'azione di lettura (get_*)?
  ├── Hai dati di appuntamento? → render_client_card (Pattern A)
  ├── Hai dati di pacchetto?    → render_package_card
  └── Altrimenti                → testo + bullet

Hai eseguito un'azione di scrittura (create_/update_/delete_/assign_)?
  ├── Riguarda un appuntamento?
  │     ├── create / update → render_client_card (Pattern B, header success)
  │     └── delete           → render_status_badge (destructive)
  ├── Riguarda un pacchetto del catalogo?
  │     ├── create / update → render_package_card
  │     └── delete           → render_status_badge (destructive)
  └── Altre entità (cliente, ferie, location, ecc.) → render_status_badge
```

---

## Estensioni future (non ancora implementate)

I componenti CSS sotto esistono già nel bundle ma non sono renderizzabili (manca il tool JS). Quando il team ne avrà bisogno, basta chiedere a chi mantiene il codice di aggiungere il tool corrispondente.

| Classe CSS | Componente | Possibile tool |
|---|---|---|
| `.cb-field` | Input con label | `render_form` |
| `.cb-radio-group` | Scelta singola | `render_choice` |
| `.cb-cta--primary` / `--delete` | Pulsanti azione | `render_action_buttons` |
