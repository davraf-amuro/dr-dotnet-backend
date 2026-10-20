# dr-dotnet-backend

Pacchetto base delle linee guida dr-* per i progetti backend .NET 10: Minimal API, Windows Service, worker e class library.

## 🧩 Cosa contiene

| File | Tipo | A cosa serve |
|------|------|--------------|
| `.github/instructions/dotnet-project-type.instructions.md` | Istruzione | Fa rilevare all'agente il tipo di ogni progetto della solution dai segnali nel codice (`Endpoints/*.cs`, `Workers/*.cs` con `AddWindowsService`, class library referenziata, `package.json`) e indica quale istruzione e quale pacchetto servono. I job in-process di una Minimal API non sono un Windows Service; `DbContext` o EF Core rimandano a `dr-efdb`. Senza segnali riconoscibili l'agente si ferma e chiede. Fissa le convenzioni .NET (primary constructors, `async`/`await` con `CancellationToken`, logging con placeholder, `IValidator<T>`) e il gate di push `dotnet format <percorso>.csproj --verify-no-changes`. |
| `.github/instructions/producer-consumer-queue.instructions.md` | Istruzione | Rende obbligatorio il pattern produttore/consumatore quando il codice acquisisce elementi da una sorgente e li elabora uno per uno: coda `Channel` bounded con contropressione, pochi produttori e N worker con il proprio scope DI, worker idempotenti, tre casi di errore (elemento, produttore, worker), un limitatore condiviso per le quote, parametri da configurazione. Include un esempio di riferimento, gli anti-pattern e i test minimi. |
| `.github/instructions/fan-out-parallelism.instructions.md` | Istruzione | Versione .NET dello Schema A di `concurrency.instructions.md` (core): fan-out su operazioni indipendenti con `Parallel.ForEachAsync` per le collezioni e `Task.WhenAll` solo per poche operazioni diverse note a priori. `MaxDegreeOfParallelism` da configurazione e validato, `CancellationToken` del delegate in ogni ramo, errore catturato per ramo (con il timeout di `HttpClient` distinto dall'annullamento), uno scope DI per ramo, esito completo/parziale/fallito deciso dal chiamante. Per sorgente + elaborazione rimanda a `producer-consumer-queue`. Include esempi, anti-pattern e test minimi. |
| `.github/instructions/serilog-logging.instructions.md` | Istruzione | Versione .NET di `logging.instructions.md` (core) e fonte unica della configurazione Serilog per Minimal API e Windows Service: livelli, sink e template in `appsettings.json` (sovrascrivibili con variabili d'ambiente, pronti per i container), enricher in codice (`FromLogContext`, `ThreadId`, `TaskId`, `Subject`). Il soggetto del log indicato dall'utente si apre con `LogSubject.Begin` all'ingresso del flusso e compare come `S:<Campo>=<valore>` in ogni riga, così i log di un'entità si isolano con un filtro. |
| `.claude/skills/dr-audit-api/SKILL.md` | Skill Claude Code | Skill `/dr-audit-api [focus]`: audit in sola lettura di un backend C# .NET 10, con una fase di orientamento e 7 fasi (sicurezza, EF Core, architettura, dead code, conformità Minimal API o Windows Service, qualità del codice, performance). Produce un report per severità `ERROR` / `WARNING` / `INFO` senza modificare file, poi chiede se attivare il plan mode per le correzioni. |
| `Directory.Build.props` | File di radice | Impostazioni MSBuild condivise da tutti i progetti della solution: `net10.0`, `LangVersion` `14.0`, `Nullable` e `ImplicitUsings` attivi, `EnforceCodeStyleInBuild`, `AnalysisLevel` `latest`, documentazione XML generata con il warning `1591` silenziato. Configurazione Debug con simboli completi e senza ottimizzazione, Release con `pdbonly` e ottimizzazione. |
| `global.json` | File di radice | Pinna l'SDK .NET alla versione `10.0.100` con `rollForward: latestMinor`. |

Note sui file:

| Tema | Nota |
|------|------|
| Pin dell'SDK | Se l'SDK .NET installato non soddisfa `global.json` (`10.0.100`, `rollForward: latestMinor`), ogni comando `dotnet` lanciato nella cartella fallisce. L'errore sembra un problema di `dotnet new`, ma la causa è il pin. |
| Segnaposto in `Directory.Build.props` | `Authors`, `Company` e `Copyright` riportano `YourCompany`; `Product` vale `Minimal API Template`. Vanno personalizzati nel progetto host. |
| Personalizzazioni e `-Update` | Senza `-Update` l'installer non tocca `Directory.Build.props` e `global.json` già presenti. Con `-Update` li sovrascrive e le personalizzazioni del progetto host si perdono. |
| Primo push | I template `dotnet new` scrivono UTF-8 con BOM: su un progetto appena creato serve un `dotnet format` in scrittura, altrimenti il gate `--verify-no-changes` fallisce. |
| Tipi riconosciuti | Rilevati per singolo `.csproj`. `Workers/` con `AddWindowsService`/`UseWindowsService` → Windows Service (`dr-winsvc`); `Endpoints/` → Minimal API (`dr-minimalapi`), anche con job in-process (`IHostedService`, Quartz.NET) che non sono un Windows Service; un progetto per tipo nella stessa solution → soluzione multi-progetto; `.csproj` senza `Program.cs` referenziato da un altro progetto → class library, senza istruzione propria; `package.json` senza `.csproj` → frontend (`dr-fe`). In aggiunta al tipo, `DbContext` o EF Core → `dr-efdb`. Se l'istruzione del tipo rilevato manca, l'agente propone di installare il pacchetto invece di procedere a memoria. |

## 🔗 Dipendenze e domini

- Dipende da: nessuna dipendenza dichiarata. Il core `dr-guidelines` arriva comunque per primo se manca dal manifest: è una dipendenza implicita di ogni pacchetto.
- Richiesto da: `dr-minimalapi` e `dr-winsvc`. Se `dr-dotnet-backend` manca dal manifest del progetto host, i loro installer lo installano in automatico prima di sé.
- Dominio del catalogo: `dotnet-backend` ("Backend .NET (API, servizi, worker)"), di cui è l'unico pacchetto. `appliesTo`: `dotnet`.
- Prerequisiti del tipo `dotnet` nel catalogo: SDK .NET 10, git, PowerShell 7.
- Frasi della `intentMap` che portano al dominio: "api rest", "minimal api", "endpoint", "backend", "windows service", "servizio windows", "worker", "servizio in background".

Tipologie di progetto del catalogo:

| Tipologia | Ruolo di `dr-dotnet-backend` | Altri pacchetti suggeriti | Opzionali |
|-----------|------------------------------|---------------------------|-----------|
| `minimal-api` — Minimal API (.NET 10) | Suggerito | `dr-minimalapi` | `dr-efdb`, `dr-devops` |
| `worker-service` — Windows Service / Worker (.NET 10) | Suggerito | `dr-winsvc` | `dr-efdb`, `dr-devops` |
| `classlib` — Class Library (.NET 10) | Suggerito | nessuno | nessuno |
| `xunit-test` — Progetto di test xUnit (.NET 10) | Non citato | nessuno | nessuno |

File di altri pacchetti citati dalle istruzioni e dalla skill. Il core `dr-guidelines` si installa da solo se manca dal manifest; gli altri pacchetti no:

| File citato | Chi lo cita | Pacchetto che lo porta |
|-------------|-------------|------------------------|
| `code-organization`, `sensitive-data`, `logging`, `input-validation` (`.instructions.md`) | Skill, fase 0; `input-validation` anche l'istruzione; `logging` anche `serilog-logging` | `dr-guidelines` (core) |
| `no-hardcoded-values.instructions.md` | `producer-consumer-queue` (configurazione di worker e coda), `fan-out-parallelism` (`MaxDegreeOfParallelism`) | `dr-guidelines` (core) |
| `concurrency.instructions.md` | `fan-out-parallelism` (principio, controllo di indipendenza, dichiarazione della serie) | `dr-guidelines` (core) |
| `database-provider.instructions.md`, `database-startup-resilience.instructions.md` | Istruzione (tabella EF Core) e skill, fase 0 | `dr-efdb` |
| `.github/copilot-instructions.md` | Skill, fase 0; istruzione, per il gate di push generico | `dr-guidelines` (core): l'installer del core lo copia in `.github/` |
| `minimal-api-architecture.instructions.md` | Istruzione e skill | `dr-minimalapi` |
| `windows-service.instructions.md` | Istruzione e skill | `dr-winsvc` |
| `frontend-organization.instructions.md` | Istruzione | `dr-fe` |

Se un'istruzione manca nel progetto host, la skill lo dichiara come `[istruzione mancante: nome-file]` e prosegue con le regole che contiene.

## 🚀 Come si installa

Di solito non serve installarlo a mano. Per una soluzione nuova segui la guida del core [Creare una soluzione da zero](https://github.com/davraf-amuro/dr-guidelines/blob/main/docs/guida-nuova-soluzione.md): `/dr-scaffold` installa da solo i pacchetti giusti. Il flusso completo non è ancora stato provato sul campo. Anche l'installazione di `dr-minimalapi` o `dr-winsvc` porta con sé questo pacchetto, se manca.

A mano. Il core `dr-guidelines` porta `CLAUDE.md`, configurazione e skill: se manca dal manifest, l'installer lo installa per primo, con entrambe le forme qui sotto. Prerequisiti: PowerShell 7, git, `gh auth status` autenticato (l'installer si scarica con `gh api`; i repo sono Public dal 2026-09-21). L'installer clona il repo da `github.com` con `git clone --depth 1`: con i repo Public non servono credenziali; su un repo Private anche git deve poterlo leggere (`gh auth setup-git`).

Lancia i comandi dalla **root del repository host**: l'installer usa la cartella corrente come destinazione e non avvisa se sbagli cartella.

Via core, un solo installer:

```powershell
Set-Location <root-del-progetto-host>
& ([scriptblock]::Create((gh api repos/davraf-amuro/dr-guidelines/contents/dr-guidelines-install.ps1 -H "Accept: application/vnd.github.raw" | Out-String))) -Package dr-dotnet-backend
```

Oppure con l'installer del pacchetto:

```powershell
& ([scriptblock]::Create((gh api repos/davraf-amuro/dr-dotnet-backend/contents/dr-dotnet-backend-install.ps1 -H "Accept: application/vnd.github.raw" | Out-String)))
```

Nota: la forma breve `irm https://raw.githubusercontent.com/... | iex` è un'alternativa valida, perché i repo sono Public. Su un repo Private risponderebbe 404: lì si usa `gh api`.

## 📦 Cosa finisce nel progetto host

| Percorso nel progetto host | Contenuto |
|----------------------------|-----------|
| `.github/instructions/dotnet-project-type.instructions.md` | Rilevamento del tipo di progetto, convenzioni .NET, gate di push |
| `.github/instructions/producer-consumer-queue.instructions.md` | Pattern produttore/consumatore con coda bounded |
| `.github/instructions/fan-out-parallelism.instructions.md` | Fan-out parallelo su operazioni indipendenti |
| `.claude/skills/dr-audit-api/` | Cartella della skill, con `SKILL.md` |
| `Directory.Build.props` | Impostazioni MSBuild comuni (`rootFiles` del catalogo) |
| `global.json` | Pin dell'SDK .NET (`rootFiles` del catalogo) |
| `.ai/dr-guidelines-packages.json` | Voce `dr-dotnet-backend` con data (`installedAt`) e commit installato (`commit`) |

Il pacchetto non modifica `CLAUDE.md` né i file di configurazione (`.editorconfig`, `.gitignore`, `.gitattributes`, `.claude/settings.json`, `.mcp.json`). Li scrive il core, solo se viene installato insieme perché mancava dal manifest.

Non vengono copiati `README.md`, `LICENSE`, `.github/ISSUE_TEMPLATE/` e l'installer `dr-dotnet-backend-install.ps1`.

Senza `-Update` un file già presente resta com'è e l'installer stampa `[SKIP]`. Per la skill il controllo vale sull'intera cartella `dr-audit-api/`.

Le skill si leggono all'avvio: dopo l'installazione riavvia Claude Code, oppure esegui `Developer: Reload Window` in VS Code.

## 🔄 Aggiornare

Tutti i pacchetti del progetto: `/dr-get-latest`.

Solo questo pacchetto: stesso comando dell'installer del pacchetto, con `-Update` in coda:

```powershell
& ([scriptblock]::Create((gh api repos/davraf-amuro/dr-dotnet-backend/contents/dr-dotnet-backend-install.ps1 -H "Accept: application/vnd.github.raw" | Out-String))) -Update
```

Nota: `-Update` sovrascrive le copie locali. Si installa sempre l'ultimo `main` pushato su GitHub: le modifiche a questo repo non pushate su `main` non arrivano nei progetti host.

Un file rimosso da questo repo resta nel progetto host: l'installer copia, non cancella. Fa eccezione un file elencato in `obsoleteArtifacts` del catalogo: `-Update` lo rimuove.

## 🐞 Segnalare un problema o una miglioria

Non correggere la copia nel progetto host: si perde al primo `-Update`.

Dal progetto host usa `/dr-file-feedback <descrizione>` (su Copilot il prompt `.github/prompts/dr-file-feedback.prompt.md` del core). La issue si apre in questo repo, dopo la tua conferma esplicita di titolo e corpo.

| Modello del repo | Quando usarlo |
|------------------|---------------|
| `.github/ISSUE_TEMPLATE/miglioria.md` | Richiesta evolutiva: regola nuova, precisazione, estensione del pacchetto |
| `.github/ISSUE_TEMPLATE/problema.md` | Malfunzionamento: regola sbagliata, ambigua o che porta l'agente fuori strada |

---

*Revisione v1.9 — 2026-10-20 18:20 — claude-opus-5-5*