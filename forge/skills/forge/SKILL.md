---
name: forge
description: Lavorare su Forge di Mademind con gli strumenti del server MCP «forge» - leggere e scrivere collezioni e record, viste, bozze da approvare, automazioni (workflow), dashboard, consegne e mini app del workspace della persona. Usare quando la persona nomina Forge, il suo workspace, una collezione o un elenco, un'automazione o un flusso, una bozza, una mini app, o chiede di leggere o cambiare dati che stanno in Forge.
---

# Forge

Forge è dove un'azienda tiene i dati e il lavoro del suo workspace. Il server MCP «forge» di questo plugin
lavora con la chiave personale della persona (`FORGE_MCP_KEY`): ogni strumento agisce coi suoi permessi,
solo nel suo workspace, e ogni chiamata finisce nel registro attività che vede l'amministratore.

## Prima di tutto

1. Se gli strumenti `forge` non ci sono, il server non è collegato: mancano `FORGE_MCP_URL` (l'indirizzo
   del server MCP del workspace, finisce con `/mcp`) o `FORGE_MCP_KEY` (la chiave, dal Profilo dell'app di
   Forge, voce «Chiavi MCP»). Dillo alla persona e fermati. Una risposta HTTP 401 vuol dire chiave
   sbagliata, scaduta o revocata: ne serve una nuova dal Profilo.
2. La guida vera è nel server, ed è sempre aggiornata: le istruzioni che il server dà al collegamento e la
   risorsa `forge://guida`. Leggila (con lo strumento che legge le risorse MCP) prima del primo lavoro
   che non sia una lettura semplice.
3. `identity_me`, poi `collections_list`: chi è la persona, cosa può la chiave, quali collezioni e campi
   ci sono. Negli strumenti usa gli id e le chiavi dei campi che restituiscono, mai nomi indovinati.

## Le regole che non cambiano

- Leggi prima di scrivere: `views_list` prima di sostituire le viste, `records_get` prima di cambiare un
  record.
- Una cancellazione non torna indietro: chiedi conferma alla persona prima di ogni strumento segnato
  distruttivo (`records_delete`, `records_delete_many`, `collections_delete`, `members_remove`).
- Senza i poteri da amministratore uno schema o le viste condivise si propongono in bozza
  (`drafts_propose_collection_schema`, `drafts_propose_shared_views`), poi `drafts_submit`: la approva un
  amministratore dall'app. Dillo alla persona.
- Accendere un'automazione, pubblicare una mini app, condividere fuori una consegna: solo col sì della
  persona.
- Credenziali, connettori, agenti nuovi, SQL e rilasci non si fanno dall'MCP: si fanno nell'app o li fa
  Mademind.

## Ricette

Il server ha un prompt per ognuna, con i passi: `crea_elenco`, `costruisci_flusso`, `debug_flusso`,
`pubblica_mini_app`. In breve:

- **Un elenco nuovo**: `forge://viste/tipi` per i tipi di campo, proposta dei campi alla persona,
  `collections_create` (chiave da amministratore) o la bozza, poi `records_create` o `records_import_csv`.
  Da Supabase: prima la tabella madre col vecchio id in un campo `unique` (`supabase_id`), poi la figlia
  con `{"field": "<relazione>", "match_by": "supabase_id"}` nel `mapping`: l'import collega da solo, e
  un vecchio id senza record torna in `errors` con riga e valore.
- **Un flusso di approvazione**: un campo `status` con i passaggi in `transitions`, ognuno con chi lo fa
  (`who`) e gli effetti (`effects`: `notify`, `update_records`), e `requesterField` per chi ha chiesto;
  `forge://guida`, «Un passaggio di stato con chi lo fa e i suoi effetti». Prima di cambiarli leggili
  interi con `collections_list`: con `collections_update` un passaggio rimandato con soli `from` e `to`
  tiene chi lo fa e i suoi effetti; `"who": null` toglie chi lo fa e lo lascia a chi può scrivere il record.
- **Un'automazione**: `forge://flussi/nodi` per i nodi ammessi, coi campi obbligatori e le chiavi di
  `runtime` di ognuno, `workflows_write_definition` (un campo che manca è un 400
  `workflow_definition_incomplete` col passo e il campo, prima di scrivere; quello che il compilatore,
  provato a secco, rifiuta è un 400 `workflow_definition_invalid` con passo, campo e testo in `errors`),
  prova con `workflows_run_now`, lettura con `workflows_executions` e `workflows_logs`, orario con
  `workflows_set_schedule`, `workflows_enable` col sì della persona. Una pubblicazione fallita è in
  `workflows_list`, `unpublished`, con `deploy_failed` e `compile_errors`, e `workflows_run_now` su quel
  nome è un 409 `workflow_publish_failed` con gli stessi `compile_errors`; una cancellazione fallita
  (`delete_failed`) è un 409 `workflow_delete_failed`; un nome che non c'è è 404 `workflow_not_found`.
- **Una mini app**: `forge://mini-app/kit`, `apps_preview_upload`, poi `apps_publish` o `drafts_submit`; `apps_list` dice quali sono pubblicate.
  Se scrive record, il manifesto elenca `records.create`, `records.update` o `records.set_status` e i
  campi che scrive per ogni collezione (`write.fields`); scrive coi permessi di chi la guarda e
  dall'anteprima ogni scrittura dà 403.
- **Un gruppo di persone**: è un record della collezione `groups` (nome nel campo `name`), si crea con
  `records_create` su `groups`. I `groupIds` di `members_update` e `invites_create` sono gli `id` che dà
  `records_list` su `groups`.

## Quando uno strumento rifiuta

Il rifiuto ha un JSON con `error` e `status`, poi una frase con cosa è successo e «Cosa fare». Seguila
prima di riprovare. Su un 429 aspetta i secondi di `retry_after`, e così su un 409 con `retry_after`
(un'automazione appena scritta è ancora in pubblicazione): non è un difetto, si riprova dopo quei secondi.
Tutti i codici: `forge://errori`.
