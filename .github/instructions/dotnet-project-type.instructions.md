---
applyTo: "**"
---

# Progetto .NET — Rilevamento del tipo e convenzioni di base

Si applica a ogni progetto in cui è installato `dr-dotnet-backend`. Target: **.NET 11** (`net11.0`), cioè la versione di `defaults.dotnetFramework` del catalogo del core: è l'unico punto in cui il default si legge.

Rileva il tipo dal codice prima di generare o modificare codice applicativo.

---

## Tipo di progetto

In una solution il tipo si rileva **per progetto** (`.csproj`), non per l'intera cartella.

| Segnale nel codice | Tipo | Istruzione modulare | Pacchetto che la porta |
|--------------------|------|---------------------|------------------------|
| `Workers/*.cs` presente **e** `AddWindowsService` o `UseWindowsService` in `Program.cs` | Windows Service | `windows-service.instructions.md` | `dr-winsvc` |
| `Endpoints/*.cs` presente | Minimal API | `minimal-api-architecture.instructions.md` | `dr-minimalapi` |
| `Endpoints/*.cs` più job in-process (`IHostedService`, `BackgroundService`, Quartz.NET), anche in una cartella `Workers/`, **senza** `AddWindowsService`/`UseWindowsService` | Minimal API con job in-process | `minimal-api-architecture.instructions.md` | `dr-minimalapi` |
| Un progetto Windows Service e un progetto Minimal API nella stessa solution | Soluzione multi-progetto | Leggi entrambe le istruzioni modulari | `dr-winsvc` + `dr-minimalapi` |
| `.csproj` con SDK `Microsoft.NET.Sdk`, senza `Program.cs` né `Endpoints/` né `Workers/`, referenziato da un altro progetto della solution | Class library | Nessuna istruzione di architettura propria: valgono le convenzioni di questo file e l'istruzione del progetto che la referenzia | — |
| `package.json` presente, nessun `.csproj` | Frontend | `frontend-organization.instructions.md` | `dr-fe` |
| Nessun segnale riconoscibile | Tipo non rilevato | Fermati. Chiedi: "Questo è un Minimal API, un Windows Service o una class library?" | — |

**Un host ASP.NET Core con job schedulati non è un Windows Service.** Un `IHostedService` o un job Quartz.NET dentro una Minimal API gira nello stesso processo dell'API, anche in container: non applicare `windows-service.instructions.md` e non proporre `dr-winsvc`. A decidere è `AddWindowsService`/`UseWindowsService`, non il nome della cartella.

**Accesso dati con EF Core** — segnale che si somma al tipo, in qualsiasi progetto, class library compresa:

| Segnale nel codice | Istruzioni modulari | Pacchetto che le porta |
|--------------------|---------------------|------------------------|
| Classe che eredita `DbContext`, oppure pacchetto `Microsoft.EntityFrameworkCore.*` in un `.csproj` | `database-provider.instructions.md` (provider, filtri, DTO) e `database-startup-resilience.instructions.md` (avvio con il DB irraggiungibile) | `dr-efdb` |

Se l'istruzione modulare corrispondente al tipo rilevato, o al segnale EF Core, non è presente in `.github/instructions/`, il pacchetto che la contiene non è installato: segnalalo e proponi di installarlo, invece di procedere a memoria.

Leggi sempre l'istruzione modulare corretta prima di generare o modificare codice.

---

## Convenzioni .NET

- **Primary constructors** per l'iniezione delle dipendenze
- `async`/`await` per ogni operazione di I/O, con `CancellationToken` propagato
- Logging strutturato con placeholder — mai interpolazione di stringa nel log
- Naming: namespace `snake_case`, classi `PascalCase`, variabili `camelCase`
- Validazione input: ogni endpoint con body usa `IValidator<T>`; segui `input-validation.instructions.md`
- Acquisizione da una sorgente ed elaborazione per elemento: separale con una coda FIFO bounded, produttori e N worker; segui `producer-consumer-queue.instructions.md`

---

## File di progetto comuni

`dr-dotnet-backend` installa nella radice del progetto host:

| File | Contenuto |
|------|-----------|
| `Directory.Build.props` | Impostazioni MSBuild condivise da tutti i progetti della solution: `TargetFramework` (oggi `net11.0`), `Nullable`, `ImplicitUsings`, analisi del codice, configurazioni Debug/Release. `LangVersion` non è impostata: il default del compilatore segue il TFM (C# 15 su `net11.0`) |
| `global.json` | Versione minima dell'SDK .NET (`11.0.100`) con `rollForward: latestMajor` |

**Perché `latestMajor`.** Il minimo resta `11.0.100`, ma un SDK più nuovo installato sul PC non blocca il progetto: con `latestMinor` o `latestFeature` bastava avere solo un SDK di una major successiva perché ogni comando `dotnet` nella cartella fallisse. Un SDK più vecchio del minimo continua invece a essere rifiutato.

Modifica questi file nel progetto host quando servono impostazioni specifiche (per esempio `Authors`, `Company`, `Product`, che il file non imposta): l'installer non li sovrascrive senza `-Update` esplicito.

### Progetto .NET già esistente

Se l'host contiene già un `*.csproj`, `*.sln` o `*.slnx`, l'installer **non impone** `Directory.Build.props` e `global.json`: li scrive come `Directory.Build.props.new` e `global.json.new` e lo segnala con un avviso. Il motivo: `EnforceCodeStyleInBuild`, `AnalysisLevel=latest` e `GenerateDocumentationFile` alzano il livello del lint su codice esistente e possono far fallire `dotnet format --verify-no-changes` che prima passava (dr-dotnet-backend#7).

Per adottarli:

1. confronta ogni `.new` con il file che hai già (se c'è) e decidi cosa tenere;
2. rinomina il file (`Directory.Build.props.new` → `Directory.Build.props`, `global.json.new` → `global.json`);
3. esegui il lint dalla radice del repo, per esempio `dotnet format src/Ordini.Api/Ordini.Api.csproj --verify-no-changes`, e correggi le violazioni prima del commit.

Nel **nuovo scaffolding** (progetti nati nella stessa esecuzione di `/dr-scaffold-solution`, anche quando ci si arriva da `/dr-scaffold`) la skill adotta i due file da sola: un progetto appena creato non ha codice da proteggere. `/dr-scaffold-project` non installa pacchetti e quindi non adotta i `.new`.

---

## Gate di Push — Lint .NET

⛔ Prima di qualsiasi `git push` su un progetto .NET, esegui il comando dalla **radice del repository**, con il percorso relativo del `.csproj` o della solution, lo stesso usato dalla CI:

```powershell
dotnet format src/Ordini.Api/Ordini.Api.csproj --verify-no-changes
```

| Situazione | Comando |
|------------|---------|
| Un solo progetto | `dotnet format src/Ordini.Api/Ordini.Api.csproj --verify-no-changes` |
| Solution (`.sln` o `.slnx`) | `dotnet format Ordini.slnx --verify-no-changes` |
| Più `.csproj` senza solution | Un comando per progetto, ciascuno con il suo exit code: `dotnet format src/Ordini.Api/Ordini.Api.csproj --verify-no-changes`, poi `dotnet format src/Ordini.Worker/Ordini.Worker.csproj --verify-no-changes`, e così via |

| Exit code | Azione |
|-----------|--------|
| `0` | Lint pulito — push consentita |
| Non-zero | **BLOCCA la push** — elenca i file con violazioni, chiedi conferma prima di correggere, poi riesegui |

**Cita nell'output il comando eseguito e il suo exit code** (per esempio `dotnet format src/Ordini.Api/Ordini.Api.csproj --verify-no-changes` → exit 0), uno per ogni progetto: lo chiede il Gate di Push del core in `.github/copilot-instructions.md`. Dire "lint pulito" senza citarli non vale come verifica (dr-dotnet-backend#9).

I template `dotnet new` scrivono UTF-8 **con BOM**: su un progetto appena creato serve un `dotnet format` in scrittura prima del primo push, altrimenti il check fallisce.

Questo comando è il "comando di verifica dichiarato dall'istruzione di dominio" a cui rimanda il gate di push generico in `.github/copilot-instructions.md`.

---

## Checklist Post-Generazione

- [ ] Tipo rilevato correttamente, istruzione modulare letta
- [ ] Primary constructors e `async`/`await` usati dove servono
- [ ] Logging strutturato con placeholder
- [ ] Acquisizione ed elaborazione per elemento separate da una coda bounded (`producer-consumer-queue.instructions.md`), oppure dichiarato perché non serve
- [ ] `dotnet format --verify-no-changes` pulito prima della push

---

*Istruzione v1.3 - Progetto .NET - 2026-11-27 — claude-sonnet-5-5 — adozione dei `.new` attribuita a `/dr-scaffold-solution`, non a `/dr-scaffold-project` (davraf-amuro/dr-guidelines#37). v1.2: .NET 11 e `global.json` `latestMajor`; `Directory.Build.props`/`global.json` come `.new` su progetti esistenti (dr-dotnet-backend#7); Gate di Push con esempi e citazione dell'esito (dr-dotnet-backend#9). v1.1: class library, EF Core → dr-efdb, job in-process distinti dal Windows Service (dr-dotnet-backend#1); rimando a producer-consumer-queue (dr-dotnet-backend#2)*
