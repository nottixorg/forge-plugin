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
- Credenziali e consensi dei connettori, agenti nuovi, SQL e rilasci non si fanno da qui: lo stato dei connettori si legge (connectors_list).
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

`mcp_keys_list` elenca le chiavi della persona (con i poteri da amministratore, tutte quelle del
workspace), mai il segreto; `currentKeyId` è la chiave con cui stai parlando. `mcp_keys_revoke`
(scrittura) ne revoca una: se è quella in uso, dalla chiamata dopo ogni risposta è 401
`invalid_api_key`. Prima di revocarla chiedilo alla persona.

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
   - conta o somma: `records_aggregate`; una misura salvata, per periodo e gruppo: `measures_define` (admin),
     vedi «Dashboard, misure, consegne». Nei widget delle dashboard non ancora (C-2386);
   - unisce più tabelle in un elenco o in un calendario: una pagina (`pages_upsert`), vedi «Pagine, menu e
     home dell'app»;
   - unisce più tabelle in una schermata su misura: una mini app. Non ancora per intero: il manifesto non
     nomina un elenco creato da qui (C-2057) e l'app pubblicata non compare nel menu (C-2192);
   - un riepilogo ricalcolato ogni notte: un'automazione a orario che scrive un elenco di riepilogo, con
     l'orario scritto nella partenza `cron` (poi lo cambia `workflows_set_schedule`).
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
   - gira a un orario (`pg_cron`): un'automazione a orario, con lo `schedule` della partenza `cron`;
   - è chiamata da fuori, con un webhook: un'automazione che parte da un webhook, non ancora (C-2249);
   - gira davanti a una persona, che vede la proposta e conferma: una mini app. Le mini app che scrivono
     record: non ancora (C-2293);
   - fa un calcolo lungo da sola, come assegnare persone a turni o compiti: le regole dei lavori notturni
     del workspace, dove ci sono: si cambiano con `night_rules_preview` e `night_rules_save` (admin, vedi
     «Regole dei lavori notturni»); l'anteprima del giorno dopo che le giudica non ancora (C-2457); cosa
     il motore di assegnazione sa esprimere: non ancora (C-2294). Generare da zero la
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
   partenza `cron` è l'orario con cui il workflow nasce, nel fuso `timezone` (di base Europe/Rome); dopo
   lo cambia solo `workflows_set_schedule`, anche il fuso con `time_zone`. Una definizione minima,
   con un passo solo che fa un conto:

   ```json
   {"create": true, "definition": {"apiVersion": "forge.workflows/v2", "meta": {"name": "prova-conto", "description": "Fa un conto e lo scrive nei log"}, "triggers": [{"type": "cron", "schedule": "0 7 * * *"}], "steps": [{"kind": "leaf", "id": "conto", "uses": "data.transform", "with": {"value": "${1 + 1}"}}]}}
   ```
   Per leggere o contare i record di una collezione c'è `forge.records_query`: `collection_id`, `filters`
   (in AND, ognuno `{"field", "op", "value"}`, gli stessi operatori delle viste) e `limit` (0 = solo il
   conteggio). L'uscita del passo è `count` (tutti i record che passano i filtri), `records` e `has_more`:
   `${urgenti.count}` in un passo `notify.app` dopo manda il numero.
3. La risposta dà il `name` del workflow pubblicato: usalo negli strumenti dopo. Nasce spento. Per qualche
   secondo è ancora in pubblicazione: gli strumenti rispondono 409 `workflows_publishing` con `retry_after`,
   e si riprova dopo quei secondi.
4. `workflows_set_schedule` per l'orario (cron a 5 campi), `workflows_enable` per accenderlo,
   `workflows_run_now` per provarlo subito. Fra due partenze, a mano o dall'orario, c'è una pausa: 429
   `workflows_trigger_cooldown`, con `retry_after` (i secondi da aspettare) e `cooldown_s` (quanto dura).
5. `workflows_executions`, poi `workflows_logs` con l'`execution_id` per leggere cosa ha fatto ogni passo:
   ogni passo che ha girato (fuori da un `foreach`) ha una riga `step.done` col suo `id` in `step`. I dati che
   un passo produce non vanno nei log; per vederli mettili in un passo `return` alla fine
   (`{"kind": "return", "payload": {"saluto": "${saluto}"}}`) e leggi il risultato con
   `workflows_execution`.

Debug: un'esecuzione fallita ha l'errore in `workflows_executions`; il passo che ha sbagliato e i suoi dati
in `workflows_logs`. Correggi la definizione (`workflows_definition` per rileggerla,
`workflows_write_definition` con `create: false`) e rilancia con `workflows_run_now`. `workflows_delete`
cancella un'automazione e non si annulla: solo col sì della persona. Cancella subito la definizione
(`workflows_definition` dà 404); il workflow pubblicato e il suo orario li toglie il provisioner dopo, e per
qualche minuto (circa 4) `workflows_list` lo mostra ancora: la risposta lo dice con `published_removal: "pending"`.
Non è un errore e non si cancella di nuovo. Le definizioni gestite da Terraform
non si scrivono né si cancellano da qui (409).

### Mini app

1. Leggi `forge://mini-app/kit`: manifesto, cosa può fare il codice, cosa è vietato.
2. `apps_preview_upload` con `manifest` (l'`app.yaml` in JSON) e `content` (l'HTML costruito, tutto in
   linea). L'anteprima la vedi solo tu, in sola lettura.
3. La pubblica un admin con `apps_publish`; chi non è admin chiede la pubblicazione con `drafts_submit`.
   `apps_withdraw` la ritira.
4. Un'anteprima che non serve più si scarta con `apps_preview_delete` (la pubblicata resta).

### Regole dei lavori notturni

- Dove il tenant le usa (turni, assegnazioni, generatori di lavori, sostituzioni), i job della notte leggono
  la versione salvata dal workspace o, se non c'è, il file di Mademind. Solo con la chiave admin.
- `night_rules_list` le elenca; `night_rules_get` dà il documento in uso (`in_use`), le versioni e le
  bozze. Per cambiarne una: copia `in_use.body`, cambia solo quello che serve, `night_rules_preview`.
- `night_rules_preview` scrive una bozza che nessun job legge e la controlla con lo schema del job: un
  documento sbagliato è rifiutato con ogni motivo. Poi il job fa l'anteprima del giorno dopo con la
  bozza (chi lavora, quali lavori nascono) e scrive l'esito sulla bozza; non ancora (C-2457).
- Quando l'anteprima di una bozza è arrivata (`preview.page: true` nella versione), `night_rules_get` con
  `page_version` dà la sua pagina in `page.html`: chi lavora, quali lavori nascono, chi resta senza.
  Leggila prima di salvare; se manca, `page.missing` dice perché.
- L'anteprima calcola l'effetto delle regole di assegnazione (`assignment_rules`) e dei generatori di
  lavori (`task_generators`). Per i turni (`rota_rules`) e le sostituzioni (`cover_proposals`) il job
  controlla la bozza ma non ne mostra l'effetto: l'esito è `rejected` col motivo e il salvataggio
  risponde 409 `night_rule_effect_not_previewed`. Quelle si cambiano nel file di Mademind.
- `night_rules_save` la rende quella che i job usano, solo con l'anteprima riuscita sullo stesso
  documento e di meno di 24 ore. `night_rules_revert` riporta una versione di prima come bozza nuova,
  che ripassa dall'anteprima; `night_rules_reset_to_file` torna al file di Mademind.

### Dashboard, misure, consegne

- `measures_catalog` dice le misure che esistono; `measures_values` i loro valori.
- Con la chiave admin, `measures_define` crea o cambia una misura del workspace (chiave `custom_...`):
  conteggio, somma o media di una collezione per periodo e gruppo, oppure una SELECT sui dati del tenant
  che restituisce `period`, `value` e, se serve, `dimension`. Il server fa un dry run e rifiuta ciò che
  non è una SELECT, che legge fuori dai dataset del tenant o che supera il tetto di byte; poi misura le
  righe e rifiuta chiavi (`keys`) che non sono una chiave. `measures_custom_list` mostra definizione,
  misure fatte, chi e quando; `measures_remove` la toglie. Nei widget delle dashboard non ancora (C-2386).
- `reservations_pivot` dà le prenotazioni in tabella pivot o per paese (la mappa), sulle `pivotSources` di
  `measures_catalog`: solo totali per gruppo, mai righe o nomi degli ospiti. I filtri nominano una dimensione,
  mai una colonna; gli stessi filtri valgono nei widget `pivot_table` delle dashboard.
- `dashboards_create` fa una dashboard personale coi widget dell'app (misure, pivot, mappa) e il loro
  posto sulla griglia; `dashboards_append` aggiunge widget senza toccare gli altri. In `dashboards_update`
  `{"keep": i}` tiene il widget i di `dashboards_list` com'è: senza, un widget fatto nell'app si perde.
- Le dashboard condivise, che vede ogni persona del workspace: `dashboards_shared_list`; le cambia un
  amministratore con `dashboards_shared_upsert` (stessi widget e stesso `keep`) e `dashboards_shared_delete`.
  Quelle di Mademind in git (source file) si cambiano dopo `ui_adopt_files`.
- `deliverables_create` pubblica un documento (`kind` `blocks` o `md`, mai HTML); `deliverables_update`
  aggiunge una versione; `deliverables_share` (admin) fa il link pubblico di una consegna `snapshot`.

### Notifiche

`notifications_list` legge la campanella dell'app, solo le tue e solo di questo workspace (risposte
dell'agente, consegne, esiti e richieste, bozze); `unread: true` per quelle da leggere.
`notifications_mark_read` con l'id, `notifications_mark_all_read` per tutte. Con `kind` (`outcome`,
`request`) o `collection` leggi invece gli avvisi delle richieste come le pagine delle richieste
dell'app, con id numerici. Le preferenze (push ed email per genere, testo nella push, ore di silenzio):
`notifications_prefs_get`, e `notifications_prefs_update` con i soli campi da cambiare. Valgono per la
persona su tutta l'azienda, non solo in questo workspace.

### Modifiche alla configurazione (admin)

`activity_config_changes` dà le modifiche a connettori e credenziali, automazioni, agenti e persone, come la
scheda «Configurazione» della pagina Attività: quando, quale azione, chi (email abbreviata), severità.
`area` filtra una sezione (`workflows`, `connectors`, `agents`, `members`). Le chiamate delle chiavi MCP
stanno invece in `activity_list`.

### Connettori (admin)

`connectors_list` dice i connettori del workspace e se la credenziale c'è (`credential.status`:
`configured`, `missing`, `not_applicable`, `error`), come la pagina Connettori dell'app. `connectors_test`
prova la connessione di uno (`{"connector": "asana"}`): una lettura leggera verso il fornitore con la
credenziale salvata, risponde `ok`, `fail` col motivo o `not_testable`. Ogni prova è una chiamata vera al
fornitore: falla quando serve, non in un ciclo. Il segreto non esce mai; inserire o cambiare una
credenziale e dare un consenso si fa solo nell'app.

### Aspetto dell'app

`branding_get` dice l'aspetto che la persona vede (`look`: nome, icona, colore, logo, e in `sources` da
dove viene ognuno) e il nome, il colore e le icone dell'app installata sul telefono o sul computer
(`installedApp`). L'app installata prende nome e icone dai file del tenant quando Mademind rilascia
l'app: `branding_update` (admin) cambia `look`, non `installedApp`. Se la persona vuole un altro nome o
altre icone sull'app installata, dille che lo cambia Mademind. Tutti e due solo su un'installazione
dedicata a un'azienda (403 `branding_dedicated_only`).

### Cancellazione GDPR ed export (admin)

`gdpr_request` chiede a Mademind una cancellazione GDPR o un export: `erasure_subject` (i dati di una
persona, con `collection` e `record_id` del record che la riguarda), `erasure_workspace` (tutto il
workspace) o `export_workspace` (l'export di tutti i dati). Crea solo la richiesta: da qui non si cancella
e non si esporta niente. La esegue Mademind, dopo averne parlato con l'amministratore, e la richiesta
resta nel registro attività con chi l'ha fatta. Prima di mandarla chiedi conferma alla persona. Con `note`
(facoltativa, al massimo 500 caratteri) dici a Mademind qualcosa sulla richiesta, per esempio che è una prova
da non eseguire: va nella mail a Mademind e nel registro, quindi senza indirizzi email né numeri di telefono
(400 `gdpr_note_personal_data`).

### Persone e inviti (admin)

`members_list`, `members_update` (ruolo e gruppi), `invites_create` (nessuna mail parte: la risposta dà il
codice e la pagina da mandare alla persona), `invites_revoke`, `members_remove`. Il workspace non resta mai
senza un amministratore (409 `last_admin`).

Un gruppo di persone è un record della collezione `groups`, col nome nel campo `name` (unico): si crea con
`records_create` su `groups` e si rinomina con `records_update`. I `groupIds` e i `leadGroupIds` di
`members_update` e `invites_create` sono gli `id` che dà `records_list` su `groups`.

## Limiti

- Quote per chiave e per workspace (chiamate al minuto, record letti, scritti e cancellati al giorno,
  workflow accesi): `quotas_list` dice massimo, usato e quando riparte. Oltre, 429 con `retry_after`.
- `records_delete_many`: da 1 a 100 record per chiamata, tutti o nessuno.
- Un campo di testo: al massimo 4.000 caratteri.
- Una pagina di `records_list`: usa `limit` e `cursor`, non chiedere tutto in una volta.

## Cosa non si fa da qui, e perché

- **Credenziali e consensi dei connettori** (password, token, consensi OAuth): si inseriscono, si cambiano e si
  danno solo nell'app, da chi li possiede. Dall'MCP si leggono lo stato e la prova di connessione.
- **Creare o cancellare agenti, cambiare modello e risorse di un agente**: costi e sicurezza di Mademind.
  Le istruzioni in più e gli esempi di un agente si cambiano con `agents_update`, che lo accende e lo spegne
  (`enabled`).
- **SQL che scrive o legge fuori dal tenant**: una misura nuova è solo una SELECT sui dati del tenant,
  controllata dal server prima di salvarla e a ogni lettura.
- **Rilasci, infrastruttura, migrazioni**: non sono del workspace.
- **Cancellazione GDPR ed export di tutto**: li esegue Mademind; da qui si chiedono con `gdpr_request`.
- **Creare chiavi MCP e approvare bozze**: si fa dall'app, da una persona. Una chiave non crea chiavi e
  non approva il proprio lavoro. Elencarle e revocarle sì.

## Errori

Ogni rifiuto ha un JSON con `error` (il codice) e `status`, poi una frase con cosa è successo e cosa fare dopo.
Tutti i codici: risorsa `forge://errori`.
