# Plugin «forge» per Claude Code (e configurazione per Codex)

Collega Claude Code al server MCP di Forge del tuo workspace. Da lì Claude legge e scrive collezioni e
record, prepara viste, bozze, automazioni, dashboard, consegne e mini app, con i tuoi permessi. Il plugin
porta anche una skill che spiega a Claude come usare il server.

Il plugin non contiene chiavi né indirizzi. La chiave è tua e resta nel tuo computer.

## Cosa serve

- **La chiave MCP.** Nell'app di Forge: Profilo, sezione «Chiavi MCP», «Crea la chiave». La chiave si vede
  una volta sola: copiala subito. Dura al massimo 30 giorni. Spunta «Anche scrittura» se vuoi che Claude
  crei e modifichi record; «Poteri da amministratore» c'è solo per gli amministratori e serve a cambiare
  elenchi, automazioni e persone senza passare dalle bozze.
- **L'indirizzo del server MCP** del tuo workspace, quello che finisce con `/mcp`. Te lo diamo noi.
- **Claude Code** installato sul computer. Il plugin non funziona su claude.ai, Claude Desktop o
  sul telefono: per ora il server accetta la chiave solo da Claude Code.

Prima di aprire Claude Code metti indirizzo e chiave nell'ambiente, per esempio in `~/.zshrc` o
`~/.bashrc`:

```bash
export FORGE_MCP_URL="https://…/mcp"
export FORGE_MCP_KEY="fmk_…"
```

## Installare in Claude Code

Dal terminale:

```bash
claude plugin marketplace add nottixorg/forge-plugin
claude plugin install forge@mademind
```

Oppure dentro Claude Code:

```text
/plugin marketplace add nottixorg/forge-plugin
/plugin install forge@mademind
```

Riapri Claude Code. Il comando `/mcp` deve mostrare `forge` collegato. Per provare, chiedi «Chi sono in
Forge e quali elenchi vedo?».

Per prendere una versione nuova del plugin: `claude plugin marketplace update mademind`, poi
`claude plugin update forge@mademind`.

## Codex

Copia il blocco di [`forge/codex/config.toml`](forge/codex/config.toml) in `~/.codex/config.toml` e
scrivi l'indirizzo giusto al posto di `INDIRIZZO-DEL-SERVER-MCP`. La chiave Codex la prende da
`FORGE_MCP_KEY`. Il file [`forge/codex/AGENTS.md`](forge/codex/AGENTS.md) è la guida del server: mettilo
nella cartella del progetto in cui lavori con Codex. Con Codex il plugin non è ancora stato provato.

## Se qualcosa non va

- `/mcp` non mostra `forge`, o lo mostra non collegato: mancano `FORGE_MCP_URL` o `FORGE_MCP_KEY`
  nell'ambiente da cui hai aperto Claude Code.
- Il server risponde 401: la chiave è sbagliata, scaduta o revocata. Creane un'altra dal Profilo.
- Se perdi la chiave, o pensi che qualcuno l'abbia vista, revocala dal Profilo (Revoca, poi Conferma la
  revoca) e creane un'altra.

Ogni operazione fatta con la chiave resta nel registro delle attività del workspace, che vede
l'amministratore.
