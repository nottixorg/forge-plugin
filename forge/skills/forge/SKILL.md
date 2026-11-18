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
- **Un'automazione**: `forge://flussi/nodi` per i nodi ammessi, `workflows_write_definition`, prova con
  `workflows_run_now`, lettura con `workflows_executions` e `workflows_logs`, orario con
  `workflows_set_schedule`, `workflows_enable` col sì della persona.
- **Una mini app**: `forge://mini-app/kit`, `apps_preview_upload`, poi `apps_publish` o `drafts_submit`.

## Quando uno strumento rifiuta

Il rifiuto ha un JSON con `error` e `status`, poi una frase con cosa è successo e «Cosa fare». Seguila
prima di riprovare. Su un 429 aspetta i secondi di `retry_after`. Tutti i codici: `forge://errori`.
