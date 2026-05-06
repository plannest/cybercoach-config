// Istruzioni Anthropic: modello, tool definitions, system prompt dinamico.
// Separato dal resto del codice cosi' si evolvono le policy AI senza toccare UI o auth.

export const MODEL = 'claude-haiku-4-5-20251001';
export const MAX_TOKENS = 1024;

// Tool di rendering: non chiamano Plannest, solo istruiscono la UI a
// visualizzare un componente. L'engine intercetta questi nomi e li gestisce
// localmente (niente roundtrip di dati).
export const RENDER_TOOL_NAMES = new Set([
  'render_client_card',
  'render_package_card',
  'render_status_badge',
]);

export const TOOLS = [
  {
    name: 'render_client_card',
    description: 'Mostra nella chat una card cliente/appuntamento usando dati gia\' recuperati con get_events/get_client/get_bookings. Usa sempre date/prezzi formattati in modo leggibile (es. "15/05/2026 — 13:00", "25,00€"). Pattern A: stato inline + prezzo nel recap. Pattern B: header con status-icon (success=svolto/pagato, warning=attesa/da riscuotere, destructive=cancellato-ritardo/cliente-assente) + righe di dettaglio.',
    input_schema: {
      type: 'object',
      properties: {
        client_name:    { type: 'string' },
        client_pic_url: { type: 'string', description: 'Opz. URL thumb della foto profilo del cliente. Usa il campo photo_thumb_url dal risultato di get_events.participants[]/bookings[] o da get_client. Se non c\'e\', ometti il campo: la card mostrera\' una silhouette di fallback.' },
        date:        { type: 'string', description: 'Display es. "15/05/2026 — 13:00"' },
        activity:    { type: 'string', description: 'Nome servizio / attivita\'' },
        venue:       { type: 'string', description: 'Nome location' },
        header: {
          type: 'object',
          description: 'Opz. Pattern B: mostra un header con label + status icon colorato.',
          properties: {
            label: { type: 'string', description: 'es. "Appuntamento prenotato"' },
            kind:  { type: 'string', enum: ['success', 'warning', 'destructive'] },
          },
          required: ['label', 'kind'],
        },
        inline_status: {
          type: 'object',
          description: 'Opz. Pattern A: badge di stato nella recap-row.',
          properties: {
            label: { type: 'string', description: 'es. "Da riscuotere"' },
            kind:  { type: 'string', enum: ['success', 'warning', 'destructive'] },
          },
          required: ['label', 'kind'],
        },
        price: {
          type: 'object',
          description: 'Opz. Prezzo nella recap-row.',
          properties: {
            current_eur: { type: 'string', description: 'es. "25,00€"' },
            old_eur:     { type: 'string', description: 'es. "38,00€" (se scontato)' },
          },
          required: ['current_eur'],
        },
        recap_rows: {
          type: 'array',
          description: 'Opz. Righe aggiuntive nel recap (pacchetto, sessioni residue, totale). Ogni riga ha label + value oppure label + value_strong (bold).',
          items: {
            type: 'object',
            properties: {
              label:        { type: 'string' },
              value:        { type: 'string' },
              value_strong: { type: 'string' },
            },
          },
        },
      },
      required: ['client_name'],
    },
  },
  {
    name: 'render_package_card',
    description: 'Mostra nella chat una card pacchetto del catalogo.',
    input_schema: {
      type: 'object',
      properties: {
        title:       { type: 'string', description: 'Nome pacchetto' },
        specs:       { type: 'string', description: 'es. "5 Sessioni, 1 Servizio, 1 Mese"' },
        price_eur:   { type: 'string', description: 'es. "300€" o "25,00€"' },
        description: { type: 'string' },
      },
      required: ['title', 'price_eur'],
    },
  },
  {
    name: 'render_status_badge',
    description: 'Mostra un badge di stato standalone (non dentro una card). Usa per messaggi di conferma rapida.',
    input_schema: {
      type: 'object',
      properties: {
        kind:  { type: 'string', enum: ['success', 'warning', 'destructive'] },
        label: { type: 'string' },
      },
      required: ['kind', 'label'],
    },
  },
  {
    name: 'get_clients',
    description: 'Lista tutti i clienti della facility.',
    input_schema: { type: 'object', properties: { expand: { type: 'string', enum: ['details'] } } },
  },
  {
    name: 'get_client',
    description: 'Ottieni il profilo completo di un cliente tramite ID.',
    input_schema: { type: 'object', properties: { id: { type: 'string' }, expand: { type: 'string', enum: ['details'] } }, required: ['id'] },
  },
  {
    name: 'create_client',
    description: 'Registra un nuovo cliente. Per clienti offline usa app_access=false.',
    input_schema: {
      type: 'object',
      properties: {
        first_name: { type: 'string' }, last_name: { type: 'string' },
        email: { type: 'string' }, password: { type: 'string' },
        language: { type: 'string' }, app_access: { type: 'boolean' }, timezone: { type: 'string' },
      },
      required: ['first_name', 'last_name'],
    },
  },
  {
    name: 'get_events',
    description: 'Lista gli eventi/sessioni del calendario. Usa date_from e date_to per qualsiasi periodo (settimana, mese, range custom, passato o futuro). Se non specifichi date vengono restituiti tutti gli eventi. Con expand="details,bookings" ottieni anche i partecipanti e le transazioni per ogni evento.',
    input_schema: {
      type: 'object',
      properties: {
        date_from: { type: 'string', description: 'Data inizio ISO 8601 es: 2026-04-01T00:00:00 — includi sempre ora per precisione' },
        date_to:   { type: 'string', description: 'Data fine ISO 8601 es: 2026-04-30T23:59:59' },
        expand:    { type: 'string', description: 'Valori separati da virgola: "details" (stanza, partecipanti), "bookings" (prenotazioni con transazioni)' },
        coach_id:  { type: 'string', description: 'UUID del coach (opzionale, default: tutti i coach della facility)' },
      },
    },
  },
  {
    name: 'create_event',
    description: 'Crea un nuovo evento/sessione. type obbligatorio: "single" (1 cliente, max_participants=1), "group" (piu clienti, max_participants>1 obbligatorio), "blocked" (blocco calendario, no service_id). Usa room_id dalla mappa location->stanze. client_ids e\' array {id, already_paid}.',
    input_schema: {
      type: 'object',
      properties: {
        type:                  { type: 'string', enum: ['single', 'group', 'blocked'] },
        service_id:            { type: 'integer', description: 'Obbligatorio per single/group. Non usare per blocked.' },
        datetime_start:        { type: 'string', description: 'ISO 8601 es: 2026-04-14T10:00' },
        datetime_end:          { type: 'string', description: 'Solo per blocked (obbligatorio). Per gli altri calcolato dalla durata servizio.' },
        room_id:               { type: 'integer', description: 'Obbligatorio per single/group. Opzionale per blocked.' },
        max_participants:      { type: 'integer', description: 'Per group deve essere > 1. Per single ignorato (sempre 1).' },
        notes:                 { type: 'string' },
        assigned_collaborator: { type: 'string', description: 'UUID collaboratore assegnato (opzionale)' },
        client_ids: {
          type: 'array',
          items: {
            type: 'object',
            properties: { id: { type: 'string' }, already_paid: { type: 'boolean' } },
            required: ['id'],
          },
        },
      },
      required: ['type', 'datetime_start'],
    },
  },
  {
    name: 'update_event',
    description: 'Modifica un evento. Con notify=true invia push ai clienti.',
    input_schema: {
      type: 'object',
      properties: {
        id: { type: 'integer' }, datetime_start: { type: 'string' },
        location_id: { type: 'integer' }, room_id: { type: 'integer' },
        notes: { type: 'string' }, notify: { type: 'boolean' },
      },
      required: ['id'],
    },
  },
  {
    name: 'delete_event',
    description: 'Elimina un evento.',
    input_schema: { type: 'object', properties: { id: { type: 'integer' }, cancel_future: { type: 'boolean' } }, required: ['id'] },
  },
  {
    name: 'get_bookings',
    description: 'Lista le prenotazioni.',
    input_schema: { type: 'object', properties: { expand: { type: 'string', enum: ['event'] } } },
  },
  {
    name: 'create_booking',
    description: 'Crea una prenotazione per un cliente a un evento.',
    input_schema: { type: 'object', properties: { event_id: { type: 'integer' }, client_id: { type: 'string' } }, required: ['event_id', 'client_id'] },
  },
  {
    name: 'delete_booking',
    description: 'Elimina una prenotazione.',
    input_schema: { type: 'object', properties: { id: { type: 'integer' } }, required: ['id'] },
  },
  {
    name: 'get_services',
    description: 'Lista aggiornata dei servizi.',
    input_schema: { type: 'object', properties: {} },
  },
  {
    name: 'create_service',
    description: 'Crea un nuovo servizio. price e duration sono obbligatori. facility_id viene aggiunto automaticamente.',
    input_schema: {
      type: 'object',
      properties: {
        name:                   { type: 'string' },
        price:                  { type: 'integer', description: 'Prezzo in CENTESIMI interi (es. 2550 = 25,50 EUR). Obbligatorio, >= 0.' },
        duration:               { type: 'integer', description: 'Durata in minuti. Obbligatoria, > 0.' },
        description:            { type: 'string' },
        color:                  { type: 'string', description: 'Colore esadecimale es. #FF5733' },
        currency:               { type: 'string', description: 'Default EUR', default: 'EUR' },
        assigned_collaborators: { type: 'array', items: { type: 'string' }, description: 'UUID dei collaboratori assegnati' },
      },
      required: ['name', 'price', 'duration'],
    },
  },
  {
    name: 'update_service',
    description: 'Modifica un servizio esistente.',
    input_schema: {
      type: 'object',
      properties: {
        id:                     { type: 'integer' },
        name:                   { type: 'string' },
        price:                  { type: 'integer', description: 'Prezzo in CENTESIMI interi' },
        duration:               { type: 'integer', description: 'Durata in minuti' },
        description:            { type: 'string' },
        color:                  { type: 'string' },
        currency:               { type: 'string' },
        assigned_collaborators: { type: 'array', items: { type: 'string' } },
      },
      required: ['id'],
    },
  },
  {
    name: 'delete_service',
    description: 'Elimina un servizio (bloccato se incluso in pacchetti attivi).',
    input_schema: { type: 'object', properties: { id: { type: 'integer' } }, required: ['id'] },
  },
  {
    name: 'get_packages',
    description: 'Lista i pacchetti del catalogo.',
    input_schema: { type: 'object', properties: {} },
  },
  {
    name: 'create_package',
    description: 'Crea un nuovo pacchetto nel catalogo. type, price, facility_id obbligatori. Per type="package" servono anche sessions_limited e services.',
    input_schema: {
      type: 'object',
      properties: {
        name:                  { type: 'string' },
        type:                  { type: 'string', enum: ['package', 'subscription'] },
        price:                 { type: 'integer', description: 'Prezzo in CENTESIMI interi. >= 0.' },
        description:           { type: 'string' },
        color:                 { type: 'string' },
        currency:              { type: 'string', default: 'EUR' },
        installments_enabled:  { type: 'boolean' },
        sessions_limited:      { type: 'boolean', description: 'Obbligatorio per type="package"' },
        duration_value:        { type: 'integer', description: 'Numero di settimane/mesi di validita\'' },
        duration_unit:         { type: 'string', enum: ['week', 'month'] },
        services: {
          type: 'array',
          description: 'Servizi inclusi. Obbligatorio per type="package".',
          items: {
            type: 'object',
            properties: {
              service_id:        { type: 'integer' },
              included_sessions: { type: 'integer', description: 'null = illimitate' },
            },
            required: ['service_id'],
          },
        },
      },
      required: ['name', 'type', 'price'],
    },
  },
  {
    name: 'update_package',
    description: 'Modifica un pacchetto del catalogo. Se passi services, sostituisce completamente quelli esistenti.',
    input_schema: {
      type: 'object',
      properties: {
        id:                   { type: 'integer' },
        name:                 { type: 'string' },
        type:                 { type: 'string', enum: ['package', 'subscription'] },
        price:                { type: 'integer' },
        description:          { type: 'string' },
        color:                { type: 'string' },
        currency:             { type: 'string' },
        installments_enabled: { type: 'boolean' },
        sessions_limited:     { type: 'boolean' },
        duration_value:       { type: 'integer' },
        duration_unit:        { type: 'string', enum: ['week', 'month'] },
        services: {
          type: 'array',
          items: {
            type: 'object',
            properties: { service_id: { type: 'integer' }, included_sessions: { type: 'integer' } },
            required: ['service_id'],
          },
        },
      },
      required: ['id'],
    },
  },
  {
    name: 'get_client_packages',
    description: 'Lista i pacchetti assegnati ai clienti. Include sessioni usate/rimanenti.',
    input_schema: { type: 'object', properties: { client_id: { type: 'string' } } },
  },
  {
    name: 'assign_package_to_client',
    description: 'Assegna un pacchetto del catalogo a un cliente. facility_id iniettato automaticamente.',
    input_schema: {
      type: 'object',
      properties: {
        client_id:       { type: 'string' },
        package_id:      { type: 'integer' },
        starts_at:       { type: 'string', description: 'ISO 8601 (opz.)' },
        activation_mode: { type: 'string', enum: ['on_assignment', 'on_first_use'] },
        paid:            { type: 'boolean' },
        purchased_at:    { type: 'string' },
        require_payment: { type: 'boolean' },
        services: {
          type: 'array',
          items: {
            type: 'object',
            properties: { service_id: { type: 'integer' }, sessions_used: { type: 'integer' } },
            required: ['service_id'],
          },
        },
      },
      required: ['client_id', 'package_id'],
    },
  },
  {
    name: 'update_client_package',
    description: 'Aggiorna lo stato o i dati di un pacchetto assegnato.',
    input_schema: {
      type: 'object',
      properties: {
        id:           { type: 'integer' },
        status:       { type: 'string', enum: ['active', 'paused', 'cancelled', 'completed'] },
        notes:        { type: 'string' },
        purchased_at: { type: 'string' },
        services: {
          type: 'array',
          items: {
            type: 'object',
            properties: { service_id: { type: 'integer' }, sessions_used: { type: 'integer' } },
            required: ['service_id'],
          },
        },
      },
      required: ['id'],
    },
  },
  {
    name: 'get_reminders',
    description: 'Lista i promemoria.',
    input_schema: { type: 'object', properties: { client_id: { type: 'string' }, status: { type: 'string', enum: ['pending', 'completed', 'all'] } } },
  },
  {
    name: 'create_reminder',
    description: 'Crea un promemoria per un cliente.',
    input_schema: { type: 'object', properties: { client_id: { type: 'string' }, text: { type: 'string' } }, required: ['client_id', 'text'] },
  },
  {
    name: 'update_reminder',
    description: 'Aggiorna o segna come completato un promemoria.',
    input_schema: { type: 'object', properties: { id: { type: 'integer' }, text: { type: 'string' }, checked: { type: 'boolean' } }, required: ['id'] },
  },
  {
    name: 'delete_reminder',
    description: 'Elimina un promemoria.',
    input_schema: { type: 'object', properties: { id: { type: 'integer' } }, required: ['id'] },
  },
  {
    name: 'get_events_income',
    description: 'Incassi: lista eventi passati e futuri con prenotazioni e transazioni. payment_type per booking: paid, charged, package_included, package_included_penalty, to_collect, cancelled, cancelled_late.',
    input_schema: {
      type: 'object',
      properties: {
        period:    { type: 'string', enum: ['last_week', 'last_month', 'last_3_months', 'last_6_months', 'last_year', 'all', 'future'] },
        date_from: { type: 'string' },
        date_to:   { type: 'string' },
        coach_id:  { type: 'string' },
        scope:     { type: 'string', enum: ['self'] },
        page:      { type: 'integer' },
        per_page:  { type: 'integer' },
      },
    },
  },
  { name: 'get_me',               description: 'Profilo del coach loggato.',          input_schema: { type: 'object', properties: {} } },
  { name: 'get_facility',         description: 'Dati della facility.',                input_schema: { type: 'object', properties: {} } },
  { name: 'get_facility_settings',description: 'Impostazioni della facility.',        input_schema: { type: 'object', properties: {} } },
  { name: 'get_stats_overview',   description: 'Statistiche generali.',               input_schema: { type: 'object', properties: {} } },
  { name: 'get_notifications',    description: 'Notifiche in-app.',                    input_schema: { type: 'object', properties: {} } },
  { name: 'get_locations',        description: 'Lista sedi.',                          input_schema: { type: 'object', properties: {} } },
  {
    name: 'create_location',
    description: 'Crea una nuova sede.',
    input_schema: {
      type: 'object',
      properties: {
        name: { type: 'string' }, type: { type: 'string', enum: ['physical', 'online'] },
        address: { type: 'string' }, zip: { type: 'string' }, city: { type: 'string' },
        status: { type: 'string', enum: ['active', 'archived'], default: 'active' },
      },
      required: ['name', 'type'],
    },
  },
  {
    name: 'update_location',
    description: 'Modifica una sede esistente.',
    input_schema: {
      type: 'object',
      properties: {
        id: { type: 'integer' }, name: { type: 'string' }, type: { type: 'string', enum: ['physical', 'online'] },
        address: { type: 'string' }, zip: { type: 'string' }, city: { type: 'string' },
        status: { type: 'string', enum: ['active', 'archived'] },
      },
      required: ['id'],
    },
  },
  { name: 'get_rooms', description: 'Lista stanze.', input_schema: { type: 'object', properties: {} } },
  {
    name: 'create_room',
    description: 'Crea una nuova stanza in una location esistente.',
    input_schema: {
      type: 'object',
      properties: {
        location_id: { type: 'integer' }, name: { type: 'string' }, color: { type: 'string' },
        description: { type: 'string' }, capacity: { type: 'integer' },
        status: { type: 'string', enum: ['active', 'archived'], default: 'active' },
      },
      required: ['location_id', 'name'],
    },
  },
  {
    name: 'update_room',
    description: 'Modifica una stanza.',
    input_schema: {
      type: 'object',
      properties: {
        id: { type: 'integer' }, name: { type: 'string' }, color: { type: 'string' },
        description: { type: 'string' }, capacity: { type: 'integer' },
        status: { type: 'string', enum: ['active', 'archived'] },
      },
      required: ['id'],
    },
  },
  { name: 'get_holidays', description: 'Lista ferie/chiusure.', input_schema: { type: 'object', properties: {} } },
  {
    name: 'create_holiday',
    description: 'Crea una ferie. Se date_end omesso, dura un solo giorno.',
    input_schema: {
      type: 'object',
      properties: {
        date_start: { type: 'string', description: 'YYYY-MM-DD' },
        date_end:   { type: 'string', description: 'YYYY-MM-DD' },
        notes:      { type: 'string' },
      },
      required: ['date_start'],
    },
  },
  {
    name: 'update_holiday',
    description: 'Modifica una ferie esistente.',
    input_schema: {
      type: 'object',
      properties: {
        id: { type: 'integer' }, date_start: { type: 'string' }, date_end: { type: 'string' }, notes: { type: 'string' },
      },
      required: ['id'],
    },
  },
  {
    name: 'delete_holiday',
    description: 'Elimina una ferie.',
    input_schema: { type: 'object', properties: { id: { type: 'integer' } }, required: ['id'] },
  },
];

/**
 * Costruisce il system prompt arricchito col contesto caricato al login.
 * Include clienti, servizi, location+stanze, ferie + regole operative.
 */
export function buildSystemPrompt(ctx, lang = 'it') {
  const today = new Date().toISOString().split('T')[0];

  const clients = ctx.clients || [];
  const clientList = clients.slice(0, 50)
    .map((c) => `  - ${c.first_name || ''} ${c.last_name || ''} (id: ${c.id})`).join('\n');
  const moreClients = clients.length > 50
    ? `\n  ... e altri ${clients.length - 50} clienti (usa get_clients per la lista completa)` : '';

  const services = ctx.services || [];
  const serviceList = services
    .map((s) => `  - ${s.name} (id: ${s.id}${s.duration ? ', durata: ' + s.duration + ' min' : ''}${s.price !== undefined ? ', prezzo: ' + s.price + ' ' + (s.currency || 'EUR') : ''})`).join('\n');

  const holidays = ctx.holidays || [];
  const holidayList = holidays.length
    ? holidays.map((h) => `  - ${h.date_start}${h.date_end && h.date_end !== h.date_start ? ' -> ' + h.date_end : ''}: ${h.notes || 'Chiusura'}`).join('\n')
    : '  Nessuna ferie configurata.';

  const locationList = (ctx.locationRoomMap || []).map((loc) => {
    const roomStr = loc.rooms.length === 0
      ? '    Nessuna stanza configurata'
      : loc.rooms.map((r) => `    - ${r.name || '(senza nome)'} (id: ${r.id})`).join('\n');
    const autoNote = loc.single_room
      ? `    ⚡ Stanza unica: usa automaticamente room_id=${loc.single_room.id} senza chiedere al coach.`
      : '';
    return `  - ${loc.name} (id: ${loc.id})\n${roomStr}${autoNote ? '\n' + autoNote : ''}`;
  }).join('\n');

  return `Sei il CyberCoach, l'assistente AI integrato nell'app Plannest Coach.
Aiuti personal trainer a gestire la loro attivita' tramite comandi in linguaggio naturale.
Rispondi sempre in ${lang === 'en' ? 'inglese' : 'italiano'}, tono professionale ma diretto. Sii conciso.
Quando elenchi dati mostra solo le info piu' rilevanti in formato compatto.
Se mancano info essenziali, chiedi solo i campi strettamente necessari.

## Formattazione risposte
La UI supporta solo markdown base: **grassetto**, *corsivo*, liste con "- " o "1. ".
NON usare tabelle markdown (|...|), codice con \`\`\`, titoli #, link [..](..): non vengono renderizzati.
Per elencare piu' colonne usa liste con bullet e separatori testuali, es.:
- **Nome servizio** — 25,00 EUR — 30 min

## Componenti ricchi — quando usarli
Hai a disposizione 3 tool di rendering (\`render_client_card\`, \`render_package_card\`, \`render_status_badge\`) che mostrano componenti grafici dentro la chat. Regole:
- Usa \`render_client_card\` quando mostri UNO o piu' appuntamenti/prenotazioni all'utente (dopo get_events/get_bookings). Una card per appuntamento. Intrecciale con brevi frasi di contesto.
- Usa \`render_package_card\` quando mostri pacchetti del catalogo (dopo get_packages).
- Usa \`render_status_badge\` per conferme rapide inline ("Appuntamento creato", "Cliente aggiunto").
- Converti SEMPRE prezzi da centesimi a euro formato italiano "25,00€" prima di passarli alle card.
- Converti date ISO in formato leggibile "15/05/2026 — 13:00".
- Sui cancelled_late / no-show usa kind="destructive". Svolto/pagato = "success". Da riscuotere / cancellato normale = "warning".
- Per elenchi di soli servizi (senza prezzi/durata strutturati), continua a usare bullet testuali, non card.
- Per render_client_card, se hai il campo photo_thumb_url dal partecipante/booking/cliente, passalo come client_pic_url. Altrimenti ometti il campo (fallback automatico).

## Data di oggi: ${today}
Usa sempre l'anno e la data corretti quando crei o cerchi eventi.

## Contesto caricato al login

### Clienti (${clients.length} totali)
${clientList || '  Nessun cliente.'}${moreClients}

### Servizi disponibili
${serviceList || '  Nessun servizio configurato.'}

### Location e stanze
${locationList || '  Nessuna location configurata.'}

### Ferie
${holidayList}

## Regola auto-selezione stanza
Quando il coach chiede di creare un evento in una location con UNA SOLA stanza,
usa AUTOMATICAMENTE quel room_id senza chiedere conferma.
Se la location ha piu' stanze, chiedi al coach quale preferisce.

## Calendario e appuntamenti
Per rispondere a qualsiasi domanda sugli appuntamenti usa get_events con date_from e date_to:
- "questa settimana" -> calcola lunedi' e domenica della settimana corrente
- "questo mese" -> primo e ultimo giorno del mese corrente
- "oggi" -> date_from=oggi 00:00, date_to=oggi 23:59
- "domani" -> date_from=domani 00:00, date_to=domani 23:59
- "prossimi 7 giorni" -> da oggi a oggi+7
Usa sempre expand="details,bookings" quando il coach vuole vedere i partecipanti o i dettagli.
Per gli incassi usa get_events_income. I payment_type per ogni prenotazione:
- paid / charged -> gia' incassato
- package_included / package_included_penalty -> coperto da pacchetto
- to_collect -> da incassare
- cancelled -> cancellato senza penale
- cancelled_late -> cancellazione tardiva (penale applicabile)

## Eventi
- type="single": sessioni individuali (max_participants=1 automatico)
- type="group": classi/gruppi (max_participants > 1 obbligatorio)
- type="blocked": blocchi calendario senza servizio (datetime_end obbligatorio)
- client_ids: array di oggetti [{id: "uuid", already_paid: false}]
- coach_id e facility_id vengono iniettati automaticamente.

## Prezzi — IMPORTANTE
I prezzi sono SEMPRE in CENTESIMI interi.
- "25,50 EUR" -> price=2550
- "50 EUR"    -> price=5000
- "gratuito"  -> price=0
Quando mostri un prezzo al coach, converti da centesimi a euro.

## Azioni VIETATE — rifiuta con messaggio chiaro
- Tutto /stripe-payments (addebiti/rimborsi/checkout)
- POST /transactions (transazioni manuali)
- PUT /transactions/:id (modifica transazioni)

Tutto il resto e' consentito. NON confondere queste con le vietate:
- "Abilita/disabilita pagamento rateale" -> update_package installments_enabled ✅
- Creare/modificare servizi, pacchetti, assegnare, eventi, prenotazioni, reminder, clienti, location, stanze, ferie ✅

Per le operazioni vietate rispondi:
"Non posso eseguire questa operazione. Per [azione], utilizza la sezione apposita nell'app Plannest."`;
}
