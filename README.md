# dr-dotnet-backend

Pacchetto base delle linee guida dr-* per i progetti backend .NET 10: Minimal API, Windows Service, worker e class library.

## 🧩 Cosa contiene

| File | Tipo | A cosa serve |
|------|------|--------------|
| `.github/instructions/dotnet-project-type.instructions.md` | Istruzione | Fa rilevare all'agente il tipo di ogni progetto della solution dai segnali nel codice (`Endpoints/*.cs`, `Workers/*.cs` con `AddWindowsService`, class library referenziata, `package.json`) e indica quale istruzione e quale pacchetto servono. I job in-process di una Minimal API non sono un Windows Service; `DbContext` o EF Core rimandano a `dr-efdb`. Senza segnali riconoscibili l'agente si ferma e chiede. Fissa le convenzioni .NET (primary constructors, `async`/`await` con `CancellationToken`, logging con placeholder, `IValidator<T>`) e il gate di push `dotnet format <percorso>.csproj --verify-no-changes`. |
| `.github/instructions/producer-consumer-queue.instructions.md` | Istruzione | Rende obbligatorio il pattern produttore/consumatore quando il codice acquisisce elementi da una sorgente e li elabora uno per uno: coda `Channel` bounded con contropressione, pochi produttori e N worker con il proprio scope DI, worker idempotenti, tre casi di errore (elemento, produttore, worker), un limitatore condiviso per le quote, parametri da configurazione. Include un esempio di riferimento, gli anti-pattern e i test minimi. |
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
| Tipi riconosciuti | `Workers/` → Windows Service (`dr-winsvc`); `Endpoints/` → Minimal API (`dr-minimalapi`); entrambi → soluzione multi-progetto; `package.json` senza `.csproj` → frontend (`dr-fe`). Se l'istruzione del tipo rilevato manca, l'agente propone di installare il pacchetto invece di procedere a memoria. |

## 🔗 Dipendenze e domini

- Dipende da: nessuna dipendenza.
- Richiesto da: `dr-minimalapi` e `dr-winsvc`. Se `dr-dotnet-backend` manca dal manifest del progetto host, i loro installer lo installano in automatico prima di sé (comportamento dell'installer, non ancora provato sul campo).
- Dominio del catalogo: `dotnet-backend` ("Backend .NET (API, servizi, worker)"), di cui è l'unico pacchetto. `appliesTo`: `dotnet`.
- Prerequisiti del tipo `dotnet` nel catalogo: SDK .NET 10, git, PowerShell 7.
- Frasi della `intentMap` che portano al dominio: "api rest", "minimal api", "endpoint", "backend", "windows service", "servizio windows", "worker", "servizio in background".

Tipologie di progetto del catalogo:

| Tipologia | Ruolo di `dr-dotnet-backend` | Altri pacchetti suggeriti | Opzionali |
|-----------|------------------------------|---------------------------|-----------|
| `minimal-api` — Minimal API (.NET 10) | Suggerito | `dr-minimalapi` | `dr-efdb` |
| `worker-service` — Windows Service / Worker (.NET 10) | Suggerito | `dr-winsvc` | `dr-efdb` |
| `classlib` — Class Library (.NET 10) | Suggerito | nessuno | nessuno |
| `xunit-test` — Progetto di test xUnit (.NET 10) | Non citato | nessuno | nessuno |

File di altri pacchetti citati dall'istruzione e dalla skill. Il core `dr-guidelines` si installa da solo se manca dal manifest; gli altri pacchetti no:

| File citato | Chi lo cita | Pacchetto che lo porta |
|-------------|-------------|------------------------|
| `code-organization`, `sensitive-data`, `logging`, `input-validation` (`.instructions.md`) | Skill, fase 0; `input-validation` anche l'istruzione | `dr-guidelines` (core) |
| `.github/copilot-instructions.md` | Skill, fase 0; istruzione, per il gate di push generico | Non arriva nel progetto host: l'installer del core copia solo `.github/instructions/` e `.github/prompts/`. Difetto noto dell'installer, va copiato a mano |
| `minimal-api-architecture.instructions.md` | Istruzione e skill | `dr-minimalapi` |
| `windows-service.instructions.md` | Istruzione e skill | `dr-winsvc` |
| `frontend-organization.instructions.md` | Istruzione | `dr-fe` |

Se un'istruzione manca nel progetto host, la skill lo dichiara come `[istruzione mancante: nome-file]` e prosegue con le regole che contiene.

## 🚀 Come si installa

Di solito non serve installarlo a mano. Per una soluzione nuova segui la guida del core [Creare una soluzione da zero](https://github.com/davraf-amuro/dr-guidelines/blob/main/docs/guida-nuova-soluzione.md): `/dr-scaffold` installa da solo i pacchetti giusti. Il flusso completo non è ancora stato provato sul campo. Anche l'installazione di `dr-minimalapi` o `dr-winsvc` porta con sé questo pacchetto, se manca (comportamento dell'installer, non ancora provato sul campo).

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
| `.claude/skills/dr-audit-api/` | Cartella della skill, con `SKILL.md` |
| `Directory.Build.props` | Impostazioni MSBuild comuni (`rootFiles` del catalogo) |
| `global.json` | Pin dell'SDK .NET (`rootFiles` del catalogo) |
| `.ai/dr-guidelines-packages.json` | Voce `dr-dotnet-backend` con data (`installedAt`) e commit installato (`commit`) |

Nessuna modifica a `CLAUDE.md` né ai file di configurazione del core (`.editorconfig`, `.gitignore`, `.gitattributes`, `.claude/settings.json`, `.mcp.json`).

Non vengono copiati `README.md`, `LICENSE`, `.github/ISSUE_TEMPLATE/` e l'installer.

Senza `-Update` un file già presente resta com'è e l'installer stampa `[SKIP]`. Per la skill il controllo vale sull'intera cartella `dr-audit-api/`.

Le skill si leggono all'avvio: dopo l'installazione riavvia Claude Code, oppure esegui `Developer: Reload Window` in VS Code.

## 🔄 Aggiornare

Tutti i pacchetti del progetto: `/dr-get-latest`.

Solo questo pacchetto: stesso comando dell'installer del pacchetto, con `-Update` in coda:

```powershell
& ([scriptblock]::Create((gh api repos/davraf-amuro/dr-dotnet-backend/contents/dr-dotnet-backend-install.ps1 -H "Accept: application/vnd.github.raw" | Out-String))) -Update
```

Nota: `-Update` sovrascrive le copie locali. Si installa sempre l'ultimo `main` pushato su GitHub: le modifiche a questo repo non pushate su `main` non arrivano nei progetti host.

## 🐞 Segnalare un problema o una miglioria

Non correggere la copia nel progetto host: si perde al primo `-Update`.

Dal progetto host usa `/dr-segnala-miglioria <descrizione>` (su Copilot il prompt `.github/prompts/dr-segnala-miglioria.prompt.md` del core). La issue si apre in questo repo, dopo la tua conferma esplicita di titolo e corpo.

| Modello del repo | Quando usarlo |
|------------------|---------------|
| `.github/ISSUE_TEMPLATE/miglioria.md` | Richiesta evolutiva: regola nuova, precisazione, estensione del pacchetto |
| `.github/ISSUE_TEMPLATE/problema.md` | Malfunzionamento: regola sbagliata, ambigua o che porta l'agente fuori strada |

---

*Documento aggiornato: Ottobre 2026 — Revisione v1.4 — 2026-10-11 — claude-opus-5-5*
