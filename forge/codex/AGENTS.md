<!-- Scritto da scripts/sync_forge_plugin_guide.cjs da control_plane/service/src/mcp/guida.md: non si modifica a mano. -->

# Forge per Codex

Il server MCP `forge` (vedi `config.toml` accanto) lavora con la tua chiave personale, nel tuo
workspace di Forge. Questa è la stessa guida che il server dà a Claude: se il client legge le risorse
MCP, la versione aggiornata è `forge://guida`.

Forge è dove un'azienda tiene i dati e il lavoro del suo workspace: collezioni di record (gli elenchi dell'app), viste, automazioni (workflow), bozze da approvare, dashboard, consegne, mini app. La chiave è di una persona: ogni strumento agisce coi suoi permessi, solo nel suo workspace, e ogni chiamata finisce nel registro attività.

Da dove partire: identity_me (chi sei, ruolo, gruppi), poi collections_list (collezioni e campi). Negli altri strumenti usa l'id della collezione e le chiavi dei campi che restituisce, mai nomi indovinati.

Regole:
- Leggi prima di scrivere: views_list prima di sostituire le viste, records_get prima di cambiare un record.
- Senza i poteri da amministratore uno schema o le viste condivise si propongono in bozza (drafts_propose_*), poi drafts_submit: la approva un admin dall'app.
- Una cancellazione non torna indietro: chiedi conferma alla persona prima di ogni strumento segnato distruttivo.
- Credenziali, connettori, agenti nuovi, SQL e rilasci non si fanno da qui.
- Le quote si leggono con quotas_list.
- Per portare uno schema da Supabase o da un altro Postgres: la ricetta in forge://guida, o il prompt porta_schema.
- Un rifiuto dice, dopo il JSON, cosa è successo e «Cosa fare»: seguilo prima di riprovare.

Risorse da leggere: forge://guida (la guida intera, con le ricette), forge://errori (cosa vuol dire ogni rifiuto e cosa fare), forge://flussi/nodi (nodi e partenze ammessi nei workflow), forge://viste/tipi (tipi di campo, di vista e filtri), forge://mini-app/kit. I prompt danno le ricette passo per passo.

## Le parole

| Nell'app | Negli strumenti | Cos'è |
|---|---|---|
| Workspace | (dalla chiave) | lo spazio di un'azienda o di un reparto: persone, dati, automazioni. La chiave porta solo lì |
| Collezione, elenco, lista | `collection` | una tabella con i suoi campi. Si legge con `collections_list` |
| Record, riga, scheda | `record` | una riga di una collezione. Ha un `id` e i `values` per chiave di campo |
| Campo di stato | campo `status` | un campo con stati e passaggi ammessi: si cambia con `records_set_status`, non con `records_update` |
| Vista | `view` | un modo di guardare una collezione: tabella, bacheca (`board`) o calendario. Condivisa (la vedono tutti) o personale (solo tu) |
| Automazione, flusso | `workflow` | una sequenza di passi che parte da sola (orario, evento) o a mano |
| Nodo, passo | `uses` | un mattone di un workflow: scrive record, manda un avviso, trasforma dati |
| Bozza | `draft` | una modifica proposta che non vale finché un admin non l'approva dall'app |
| Consegna | `deliverable` | un documento, un report o una nota pubblicati nell'app |
| Mini app | `app` | una pagina fatta su misura che gira dentro l'app coi permessi di chi la guarda |
| Misura | `measure` | un numero calcolato da Mademind (ricavi, occupazione): si legge, non si scrive |

## Cosa può la tua chiave

La chiave si crea nel Profilo dell'app di Forge, alla voce «Chiavi MCP», se un amministratore ti ha messo
nel gruppo che può crearle. Dura al massimo 30 giorni; ognuno ne tiene al massimo 5 vive.

- **Lettura** (sempre): leggere collezioni, record, viste, commenti, allegati, bozze, notifiche,
  dashboard, consegne.
- **Scrittura** («Anche scrittura»): creare e modificare record, commenti, allegati, viste personali,
  dashboard, consegne; proporre bozze.
- **Poteri da amministratore** («Poteri da amministratore», solo per chi è admin del workspace):
  la chiave agisce come l'admin. Con la scrittura cambia schemi e viste condivise senza bozza, gestisce
  persone, inviti, automazioni, agenti, aspetto, registro attività, e può cancellare. Senza questo
  interruttore anche la chiave di un admin lavora come un membro e propone in bozza.

Il perimetro di una collezione (`scope` own, group, own_or_group) può farti vedere solo una parte dei record. Gli strumenti che `tools/list` non mostra non
sono per questa chiave: chiamarli dà 403.

## Da dove partire

1. `identity_me`: la persona, il ruolo, i gruppi.
2. `collections_list`: le collezioni che vedi, coi campi (chiave, tipo, obbligatorio, opzioni, stati e
   passaggi). Tienila sotto mano: ogni altro strumento sui record vuole l'id della collezione e le chiavi
   dei campi.
3. `records_list` con `limit` piccolo per vedere com'è fatto un record vero.
4. `quotas_list` se devi leggere o scrivere molto.

## Ricette

### Leggere e cercare

- Una pagina: `records_list` con `collection`, `filters` (`[{"field": "stato", "op": "eq", "value": "aperta"}]`),
  `sort`, `limit`. Per la pagina dopo ripassa `nextCursor` in `cursor`.
- Cercare un testo: `records_list` con `search` (`{"collection": "compiti", "search": "pulizie"}`), come
  la casella «Cerca in tutti i campi» dell'app: una parte di parola, in ogni campo di testo, maiuscole e
  minuscole uguali; si combina coi filtri. Non c'è un filtro `contains`. Meglio che leggere tutto l'elenco.
- Contare o sommare: `records_aggregate` (`op` count, sum, avg, min, max; `group_by` un campo). Meglio
  che leggere tutte le pagine.
- Un record con la sua storia: `records_get`, `records_revisions`, `comments_list`, `attachments_list`.

### Crea un elenco (una collezione) e riempilo

1. `collections_list`: controlla che non ci sia già.
2. Con la chiave admin: `collections_create` con `name`, `titleField` e `fields`. L'id (`col_...`) lo
   sceglie Forge e torna nella risposta.
   Senza: `drafts_propose_collection_schema` con `action: "create"`, poi `drafts_submit`; quando un admin
   la approva dall'app, la collezione c'è.
3. `records_create` per una riga, `records_import_csv` per tante.
   Nel CSV ogni cella si scrive come vuole il suo campo: casella `sì`/`no` (o `true`/`false`, `1`/`0`),
   data `2026-11-18` o `18/11/2026`, data e ora `2026-11-18 09:30` (senza fuso vale UTC), persona con la
   sua email, scelta multipla e persone o relazioni multiple in un elenco con `;` o `|`. Prova prima con
   `dry_run`: `errors` dice riga, campo e cosa scrivere. Ogni riga inserita conta un record scritto della
   quota del giorno (`quotas_list`); oltre 1 MB di CSV, dividi in più chiamate.

Esempio di campi:

```json
{"name": "Guasti", "titleField": "titolo", "fields": [
  {"key": "titolo", "label": "Titolo", "type": "text", "required": true},
  {"key": "urgenza", "label": "Urgenza", "type": "single_select", "options": ["bassa", "alta"]},
  {"key": "stato", "label": "Stato", "type": "status", "options": ["aperto", "chiuso"], "initial": "aperto",
   "transitions": [{"from": "aperto", "to": "chiuso"}]}
]}
```

Tipi di campo, perimetri e viste: risorsa `forge://viste/tipi`.

### Da Supabase, o da un altro Postgres, a Forge

La persona ti dà lo schema: tabelle, colonne, chiavi esterne, viste, funzioni e trigger (per esempio
l'uscita di `pg_dump --schema-only`). In Forge non si scrive SQL: ogni pezzo va in uno dei posti qui sotto.
Prima di creare qualcosa, scrivi alla persona la mappa intera, pezzo per pezzo, e aspetta il suo sì.
Quello che oggi Forge non fa lo dici «non ancora», col compito che lo porterà: non inventare un giro.

1. `identity_me`: per creare elenchi serve la chiave di un amministratore con i poteri da amministratore.
   Senza, ogni elenco va in bozza (`drafts_propose_collection_schema`) e lo approva un admin dall'app.
2. `collections_list`: se c'è già un elenco per la stessa cosa (anche uno gestito da file), si usa quello e
   non se ne crea un secondo: due elenchi per la stessa cosa sono due fonti che non vanno d'accordo. Agli
   elenchi da file non si aggiungono campi da qui: non ancora (C-2237). `collections_list` non dice ancora
   tutto dello schema, per esempio chi fa i passaggi di stato (C-2238).
3. **Una tabella diventa un elenco**, una colonna un campo. I tipi sono 12:

   | In Postgres | In Forge |
   |---|---|
   | `text`, `varchar` (fino a 4.000 caratteri) | `text`; un testo più lungo o su più righe `long_text` |
   | `integer`, `bigint`, `numeric`, `real` | `number` |
   | `boolean` | `checkbox` |
   | `date` | `date` |
   | `timestamptz`, `timestamp` | `datetime` (vedi il punto 6 sul fuso) |
   | un `enum`, o un `check (x in (...))` | `single_select` con le `options` |
   | un array di valori (`text[]`, `enum[]`) | `multi_select` |
   | una colonna con stati e passaggi ammessi | `status` con `options`, `initial` e `transitions` |
   | chiave esterna verso un'altra tabella | `relation` con `relationTarget` (l'id `col_...` dell'elenco collegato); una tabella ponte con le sole due chiavi esterne diventa una `relation` con `multiple: true` |
   | chiave esterna verso utenti o profili | `person` (le persone del workspace) |
   | un indirizzo web | `url` |

   Non ancora: l'ora senza giorno, come `time` (un turno che comincia alle 07:00): oggi va in `text`
   (C-2296). `json`/`jsonb` va in `long_text` e non si filtra dentro; il valore predefinito di una colonna,
   un vincolo unico su più colonne insieme e un controllo di intervallo (`check (ore between 0 and 12)`)
   non ci sono (C-2351).
4. **La chiave primaria** diventa l'id del record, che sceglie Forge. Il vecchio id va in un campo di
   testo `unique` e `required` (per esempio `supabase_id`): serve a collegare le relazioni, a ripetere
   l'import senza doppioni (una riga col valore unico già presente si salta) e a confrontare i due sistemi.
5. **L'ordine**: prima le tabelle a cui le altre rimandano, poi quelle che rimandano. Crea gli elenchi
   in quell'ordine, perché un campo `relation` vuole l'id dell'elenco collegato.
6. **I dati**: `records_import_csv`, prima con `dry_run`, poi vero, una tabella alla volta nello stesso
   ordine. Una data e ora senza fuso è letta come ora UTC: una colonna `timestamp` senza fuso slitterebbe
   di una o due ore. Prima dell'import aggiungi il fuso giusto (`2026-11-18 09:30+01:00`) o esporta in UTC.
   Le persone si scrivono con l'email. Una relazione vuole l'id di Forge, non il vecchio id: dopo l'import
   della tabella madre leggi i suoi record (`records_list`, campi `id` e `supabase_id`) e riscrivi la
   colonna nel CSV della figlia. Che l'import colleghi da solo col vecchio id: non ancora (C-2291). Una
   chiave scrive pochi record al giorno (`quotas_list`): per una tabella grande dividi su più giorni o
   chiedi all'amministratore; una quota alta per il primo caricamento non c'è ancora (C-2350).
7. **Chi vede quali righe** (le policy di Row Level Security): il perimetro dell'elenco (`scope`) con
   `collections_update`, ruoli e gruppi con `members_update`.
8. **Una vista SQL** diventa una di queste cose:
   - filtra, ordina o raggruppa una tabella: una vista dell'elenco (`views_replace_shared`). Sugli elenchi
     da file non ancora (C-2234);
   - conta o somma: `records_aggregate`. Una misura salvata da mettere in una dashboard: non ancora (C-2255);
   - unisce più tabelle in un elenco o in un calendario: una pagina (`pages_upsert`), vedi «Pagine, menu e
     home dell'app»;
   - unisce più tabelle in una schermata su misura: una mini app. Non ancora per intero: il manifesto non
     nomina un elenco creato da qui (C-2057) e l'app pubblicata non compare nel menu (C-2192);
   - un riepilogo ricalcolato ogni notte: un'automazione a orario che scrive un elenco di riepilogo, con
     l'orario messo da `workflows_set_schedule`: quello scritto nella definizione è ignorato (C-2241).
9. **Una funzione o un trigger** diventa una di queste cose:
   - tiene `updated_at` o una tabella di storico: niente, Forge tiene la storia di ogni record
     (`records_revisions`);
   - controlla un valore: `required`, `unique`, `options` del campo (per gli intervalli vedi il punto 3);
   - quando una riga cambia stato, cambia altre righe o avvisa chi ha chiesto: un passaggio di stato con
     chi lo fa e i suoi effetti (`transitions` del campo `status`). Come si scrivono chi e gli effetti, e
     che non si perdano riscrivendo i passaggi: non ancora (C-2292). Se un elenco ha già dei passaggi, non
     riscriverli con `collections_update` senza chiedere;
   - quando una riga cambia, manda una mail o chiama un servizio: un'automazione che parte dal cambio.
     Non ancora: nessuno la fa partire (C-2054);
   - gira a un orario (`pg_cron`): un'automazione a orario (vedi C-2241 qui sopra);
   - è chiamata da fuori, con un webhook: un'automazione che parte da un webhook, non ancora (C-2249);
   - gira davanti a una persona, che vede la proposta e conferma: una mini app. Le mini app che scrivono
     record: non ancora (C-2293);
   - fa un calcolo lungo da sola, come assegnare persone a turni o compiti: le regole dei lavori notturni
     del workspace, dove ci sono. Cambiarle da qui e vedere l'anteprima del giorno dopo: non ancora
     (C-2267); cosa il motore di assegnazione sa esprimere: non ancora (C-2294). Generare da zero la
     turnazione del mese: non ancora, è da decidere dove (C-2352). Nessun passo di un'automazione esegue
     codice del cliente.
10. Prima di spegnere il vecchio database, la persona confronta i due risultati sugli stessi giorni.

Il prompt `porta_schema` dà questi passi con lo schema della persona.

### Viste

- Le tue: `views_replace_personal` (l'elenco intero delle tue viste su quella collezione).
- Quelle di tutti: leggi prima `views_list`, poi manda l'elenco intero con `views_replace_shared` (admin)
  o `drafts_propose_shared_views` (gli altri). Una vista che non rimandi sparisce.
- Le collezioni dichiarate da file (le gestisce Mademind in git) non prendono viste condivise dall'MCP: 409.

### Pagine, menu e home dell'app

Le pagine sono le voci del menu dell'app di Forge: ognuna mostra una collezione come tabella o calendario,
coi filtri fissi, oppure le richieste da approvare di altre pagine. Alcune le scrive Mademind in git
(`source: file` in `pages_list` e `menu_get`): quelle non si cambiano da qui (409 `page_from_file`).

1. `pages_list` e `menu_get`: cosa c'è già. `collections_list` per l'id della collezione e le chiavi dei campi.
2. Se la pagina va in un gruppo nuovo: `menu_update` con l'elenco intero dei gruppi del workspace (un gruppo
   che non rimandi sparisce). Un gruppo può nascere prima delle sue pagine.
3. Con la chiave admin: `pages_upsert` con `id` e `page`, nella forma di un file `pages/<id>.yaml`. Senza:
   `drafts_propose_page`, poi `drafts_submit`; la pagina compare quando un admin la approva dall'app.

   ```json
   {"id": "guasti-aperti", "page": {"title": "Guasti aperti", "menu": {"label": "Guasti", "icon": "Wrench", "group": "manutenzione"},
    "collection": "guasti", "view": {"type": "table", "fields": ["titolo", "stato"]},
    "filters": [{"field": "stato", "op": "eq", "value": "aperto"}], "visible_to": {"roles": ["admin", "member"]}, "create": true}}
   ```

   `op: viewer` (senza `value`) tiene solo i record della persona che guarda: «le mie richieste».
   `visible_to` decide a chi compare la voce del menu, non chi vede i record: quello lo decidono il perimetro
   della collezione e i permessi.
4. `home_update` per i riquadri dell'area personale (conteggi di pagine, prossimi giorni di un calendario,
   ultime notifiche, collegamenti); `home_update` con `home: null` torna alla home dell'app.

`pages_delete` toglie una pagina (i record restano) e non si annulla: solo col sì della persona. Una pagina
usata da un'altra pagina o dalla home non si toglie finché non la cambi (409 `page_in_use`).

### Bozze

`drafts_list` le tue, `drafts_get` una, `drafts_submit` per chiedere l'approvazione, `drafts_withdraw` per
ritirarla. Una bozza inviata non si modifica: ritirala e proponine un'altra. Al massimo 20 aperte per
persona. L'esito arriva in `notifications_list`.

### Automazioni (workflow)

Solo con la chiave di un amministratore con i poteri da amministratore.

1. `forge://flussi/nodi`: i nodi e le partenze che questo workspace ammette, coi campi di `with`.
2. `workflows_write_definition` con `create: true` e la definizione `forge.workflows/v2`:
   `meta.name`, `triggers`, `steps`. Ogni passo dichiara il suo `kind`. Un nodo è
   `{"kind": "leaf", "id": "scrivi", "uses": "<nodo>", "with": {...}}`; i blocchi sono `foreach` (`in`, `as`,
   `steps`), `switch` (`cases` con `when` e `steps`, `default`), `try` (`steps`, `catch`), `parallel`
   (`branches`), `return`. Le espressioni stanno in `${...}`: `${ep.url}`, `${len(righe) > 0}`.
   Nodi e partenze ammessi cambiano da un workspace all'altro: li elenca `forge://flussi/nodi`, e il rifiuto
   `workflow_nodes_not_allowed` li ripete in `allowed_nodes` e `allowed_triggers`. Lo `schedule` di una
   partenza `cron` non fa partire niente: l'orario lo mette `workflows_set_schedule`. Una definizione minima,
   con un passo solo che fa un conto:

   ```json
   {"create": true, "definition": {"apiVersion": "forge.workflows/v2", "meta": {"name": "prova-conto", "description": "Fa un conto e lo scrive nei log"}, "triggers": [{"type": "cron", "schedule": "0 7 * * *"}], "steps": [{"kind": "leaf", "id": "conto", "uses": "data.transform", "with": {"value": "${1 + 1}"}}]}}
   ```
3. La risposta dà il `name` del workflow pubblicato: usalo negli strumenti dopo. Nasce spento.
4. `workflows_set_schedule` per l'orario (cron a 5 campi), `workflows_enable` per accenderlo,
   `workflows_run_now` per provarlo subito.
5. `workflows_executions`, poi `workflows_logs` con l'`execution_id` per leggere cosa ha fatto ogni passo.

Debug: un'esecuzione fallita ha l'errore in `workflows_executions`; il passo che ha sbagliato e i suoi dati
in `workflows_logs`. Correggi la definizione (`workflows_definition` per rileggerla,
`workflows_write_definition` con `create: false`) e rilancia con `workflows_run_now`. `workflows_delete`
cancella un'automazione e non si annulla: solo col sì della persona. Le definizioni gestite da Terraform
non si scrivono né si cancellano da qui (409).

### Mini app

1. Leggi `forge://mini-app/kit`: manifesto, cosa può fare il codice, cosa è vietato.
2. `apps_preview_upload` con `manifest` (l'`app.yaml` in JSON) e `content` (l'HTML costruito, tutto in
   linea). L'anteprima la vedi solo tu, in sola lettura.
3. La pubblica un admin con `apps_publish`; chi non è admin chiede la pubblicazione con `drafts_submit`.
   `apps_withdraw` la ritira.

### Dashboard, misure, consegne

- `measures_catalog` dice le misure che esistono; `measures_values` i loro valori.
- `dashboards_create` fa una dashboard personale coi widget sulle misure; `dashboards_append` aggiunge
  widget senza toccare gli altri.
- `deliverables_create` pubblica un documento (`kind` `blocks` o `md`, mai HTML); `deliverables_update`
  aggiunge una versione; `deliverables_share` (admin) fa il link pubblico di una consegna `snapshot`.

### Persone e inviti (admin)

`members_list`, `members_update` (ruolo e gruppi), `invites_create` (nessuna mail parte: la risposta dà il
codice e la pagina da mandare alla persona), `invites_revoke`, `members_remove`. Il workspace non resta mai
senza un amministratore (409 `last_admin`).

## Limiti

- Quote per chiave e per workspace (chiamate al minuto, record letti, scritti e cancellati al giorno,
  workflow accesi): `quotas_list` dice massimo, usato e quando riparte. Oltre, 429 con `retry_after`.
- `records_delete_many`: da 1 a 100 record per chiamata, tutti o nessuno.
- Un campo di testo: al massimo 4.000 caratteri.
- Una pagina di `records_list`: usa `limit` e `cursor`, non chiedere tutto in una volta.

## Cosa non si fa da qui, e perché

- **Credenziali e connettori** (password, token, consensi OAuth): restano a chi li possiede, nell'app.
- **Creare o cancellare agenti, cambiare modello e risorse di un agente**: costi e sicurezza di Mademind.
  Le istruzioni in più e gli esempi di un agente si cambiano con `agents_update`.
- **SQL e misure nuove**: le misure le scrive Mademind, perché i numeri devono restare verificati.
- **Rilasci, infrastruttura, migrazioni**: non sono del workspace.
- **Creare chiavi MCP e approvare bozze**: si fa dall'app, da una persona. Una chiave non crea chiavi e
  non approva il proprio lavoro.

## Errori

Ogni rifiuto ha un JSON con `error` (il codice) e `status`, poi una frase con cosa è successo e cosa fare dopo.
Tutti i codici: risorsa `forge://errori`.
