---
applyTo: "**"
---

# Serilog — configurazione e soggetto del log (AI Agent)

Scopo: configurazione obbligatoria di Serilog nei progetti .NET 10 (Minimal API, Windows Service, worker). Versione .NET di `logging.instructions.md` (core): le regole generali (placeholder, soggetto del log) stanno lì, qui c'è come si realizzano con Serilog. Fonte unica: Minimal API e Windows Service rimandano a questo file.

## Dipendenze NuGet

| Pacchetto | Quando |
|-----------|--------|
| `Serilog.AspNetCore` | Minimal API (include `Serilog.Extensions.Hosting` e `Serilog.Settings.Configuration`) |
| `Serilog.Extensions.Hosting` + `Serilog.Settings.Configuration` | Windows Service / worker |
| `Serilog.Sinks.Console` | sempre |
| `Serilog.Sinks.File` | sempre |
| `Serilog.Enrichers.Thread` | sempre (enricher `ThreadId`) |

## Regole core (sempre)

1. **Livelli, sink e template in `appsettings.json`** (sezione `Serilog`), letti con `ReadFrom.Configuration`. Mai sink o template scritti in codice: i programmi possono girare in container (Docker, Swarm, Kubernetes) e ogni ambiente deve poter cambiare livello, percorso o sink con le variabili d'ambiente (`SERILOG__MINIMUMLEVEL__DEFAULT`, `SERILOG__WRITETO__1__ARGS__PATH`), senza ricompilare
2. **Enricher in codice**, nell'ordine: `FromLogContext()`, `WithThreadId()`, `With<TaskIdEnricher>()`, `With<SubjectDefaultEnricher>()`. Niente sezione `Enrich` in `appsettings.json`: duplicherebbe gli enricher
3. Ogni evento di log espone `{ThreadId}`, `{TaskId}` (`-` fuori da un Task) e `{Subject}` (`-` senza soggetto)
4. Template Console e File identici nella parte variabile: `T:{ThreadId} A:{TaskId} S:{Subject}`
5. In `WriteTo` il sink `Console` è all'indice `0` e `File` all'indice `1`: gli override per ambiente dipendono dall'indice
6. In container il sink che conta è `Console` (raccolto da `docker logs` o dal driver di logging); il percorso di `File` si sposta su un volume con `SERILOG__WRITETO__1__ARGS__PATH`
7. Mai thread, task o soggetto scritti nel testo del messaggio: arrivano dagli enricher e dallo scope

## Soggetto del log

Regola generale: `logging.instructions.md`, sezione "Soggetto del log". Proprietà `Subject`, valore `<Campo>=<valore>` (es. `OrderId=42`), aperta una volta all'ingresso del flusso.

- Si apre con `LogSubject.Begin` all'inizio del flusso (handler dell'endpoint, elaborazione di un elemento nel worker) e si chiude col `using`: ogni log scritto dentro, anche dai service e dai provider chiamati, porta il soggetto
- Il valore è composto **dentro la proprietà**, non nel template del messaggio: la regola "mai interpolazione nei log" resta rispettata
- Scope annidati: vince il soggetto più interno. Un flusso, un soggetto
- L'errore di un elemento si logga **dentro** lo scope (`try`/`catch` dentro il `using`): un `catch` esterno o il middleware delle eccezioni scrivono `S:-` e proprio il log d'errore sfugge al filtro
- Scope di `ILogger.BeginScope` (quello di `LogSubject`): lo gestisce il provider Serilog di `Microsoft.Extensions.Logging`. `FromLogContext()` serve per `LogContext.PushProperty`, usato da codice che scrive direttamente con l'API di Serilog
- Nome del campo: una costante, uguale in tutto il codice (`no-hardcoded-values.instructions.md`). Niente `nameof` sul parametro: produce il nome in minuscolo (`orderId`) e il filtro distingue le maiuscole

## Pattern richiesti (copiabili)

### appsettings.json — sezione `Serilog`
```json
{
  "Serilog": {
    "Using": [ "Serilog.Sinks.Console", "Serilog.Sinks.File" ],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft.AspNetCore": "Warning",
        "Microsoft.EntityFrameworkCore": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "{Timestamp:HH:mm:ss} [{Level:u3}] T:{ThreadId} A:{TaskId} S:{Subject} {Message:lj}{NewLine}{Exception}"
        }
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/app-.log",
          "rollingInterval": "Day",
          "outputTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss} [{Level:u3}] T:{ThreadId} A:{TaskId} S:{Subject} {Message:lj}{NewLine}{Exception}"
        }
      }
    ]
  }
}
```

### Program.cs — registrazione (Minimal API e Windows Service)
```csharp
// AddSerilog su IServiceCollection: compila sia con WebApplicationBuilder sia con HostApplicationBuilder
builder.Services.AddSerilog((_, lc) => lc
    .ReadFrom.Configuration(builder.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithThreadId()
    .Enrich.With<TaskIdEnricher>()
    .Enrich.With<SubjectDefaultEnricher>());
```

`SubjectDefaultEnricher` va per ultimo: aggiunge `Subject = "-"` solo se lo scope non l'ha già valorizzato.

### Infrastructure/Logging/TaskIdEnricher.cs
```csharp
using Serilog.Core;
using Serilog.Events;

/// <summary>Aggiunge l'ID del Task corrente; "-" fuori da un Task.</summary>
internal sealed class TaskIdEnricher : ILogEventEnricher
{
    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory factory) =>
        logEvent.AddPropertyIfAbsent(
            factory.CreateProperty("TaskId", Task.CurrentId?.ToString() ?? "-"));
}
```

### Infrastructure/Logging/LogSubject.cs
```csharp
using Microsoft.Extensions.Logging;

/// <summary>Apre lo scope del soggetto del log: ogni evento scritto dentro porta la proprietà Subject.</summary>
internal static class LogSubject
{
    public const string PropertyName = "Subject";

    public static IDisposable? Begin(ILogger logger, string field, object value) =>
        logger.BeginScope(new Dictionary<string, object> { [PropertyName] = $"{field}={value}" });
}
```

### Infrastructure/Logging/SubjectDefaultEnricher.cs
```csharp
using Serilog.Core;
using Serilog.Events;

/// <summary>Garantisce la proprietà Subject su ogni evento: "-" quando nessuno scope l'ha aperta.</summary>
internal sealed class SubjectDefaultEnricher : ILogEventEnricher
{
    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory factory) =>
        logEvent.AddPropertyIfAbsent(factory.CreateProperty(LogSubject.PropertyName, "-"));
}
```

Le tre classi stanno in `Infrastructure/Logging/`, `internal` (`sealed` gli enricher): non modificare la visibilità.

### Uso — handler di un endpoint
```csharp
private const string OrderSubjectField = "OrderId";

private static async Task<IResult> GetOrderAsync(int orderId, OrderService service, ILogger<OrderService> logger, CancellationToken ct)
{
    using var _ = LogSubject.Begin(logger, OrderSubjectField, orderId);

    logger.LogInformation("Lettura ordine richiesta");
    var order = await service.GetAsync(orderId, ct);   // i log del service portano S:OrderId=42
    return order is null ? TypedResults.NotFound() : TypedResults.Ok(order);
}
```

### Uso — elemento elaborato da un worker
```csharp
private const string OrderSubjectField = "OrderCode";

// dentro DoWorkAsync
foreach (var order in pending)
{
    using var _ = LogSubject.Begin(logger, OrderSubjectField, order.Code);
    try
    {
        logger.LogInformation("Sincronizzazione avviata");
        await SyncAsync(order, stoppingToken);
    }
    catch (Exception ex) when (ex is not OperationCanceledException)
    {
        // Dentro lo scope: anche il log d'errore porta S:OrderCode=<codice>
        logger.LogError(ex, "Sincronizzazione fallita");
    }
}
```

### Isolare i log di un soggetto
```powershell
Select-String -Path logs\app-*.log -Pattern "S:OrderCode=ORD-0042 "
docker service logs <stack>_<servizio> 2>&1 | Select-String "S:OrderCode=ORD-0042 "
```

Lo spazio finale nel filtro evita che `OrderId=4` trovi anche `OrderId=42`.

**Con il driver GELF** (`logging.driver: gelf` nello stack Swarm) `docker service logs` non mostra nulla: i log vanno solo al collector. Il driver spedisce la riga del sink `Console` come testo del messaggio, quindi `Subject` non è un campo strutturato: sul collector il filtro è una ricerca di testo `S:OrderCode=ORD-0042 ` nel messaggio. Controlla `.ai/dr-guidelines-packages.json`: se elenca `dr-devops`, la fonte unica sul driver è `docker-swarm-compose.instructions.md`, sezione "5. Sezione `logging` (GELF)" — seguila e dichiaralo nell'output. Se non lo elenca, ripiego: filtra sul collector con la stessa ricerca di testo e dichiara che applichi il ripiego.

## Errori comuni (rapidi)

- `S:-` su tutti i log del flusso → `LogSubject.Begin` non chiamato, oppure log scritto fuori dal `using`
- `S:` vuoto o `{Subject}` letterale nel file → `SubjectDefaultEnricher` non registrato
- `S:-` con `LogContext.PushProperty("Subject", ...)` → `FromLogContext()` mancante, oppure `SubjectDefaultEnricher` registrato **prima** di `FromLogContext()`
- Log d'errore dell'elemento con `S:-` → `catch` fuori dallo scope del soggetto
- Override `SERILOG__WRITETO__1__ARGS__PATH` senza effetto → sink scritti in codice invece che in `appsettings.json`, oppure `File` non all'indice `1`
- Enricher duplicati → sezione `Enrich` presente anche in `appsettings.json`
- `{ThreadId}` non appare → pacchetto `Serilog.Enrichers.Thread` mancante
- `{TaskId}` spesso `-` → normale: `Task.CurrentId` è `null` nel codice sincrono e anche nelle continuazioni dei metodi `async` (es. handler ASP.NET Core dopo un `await`)
- Template con `{Properties}` → stampa tutte le proprietà, verboso: da evitare in produzione
- `UseSerilog` su `HostApplicationBuilder` → non compila: usa `builder.Services.AddSerilog`

## ✅ Checklist Post-Configurazione

- [ ] Pacchetti della tabella aggiunti al `.csproj` per il tipo di progetto
- [ ] Sezione `Serilog` in `appsettings.json` con `Console` all'indice 0 e `File` all'indice 1
- [ ] Nessun sink, livello o template scritto in codice; nessuna sezione `Enrich` in `appsettings.json`
- [ ] `ReadFrom.Configuration` + enricher in codice nell'ordine `FromLogContext`, `WithThreadId`, `TaskIdEnricher`, `SubjectDefaultEnricher`
- [ ] `TaskIdEnricher`, `SubjectDefaultEnricher` e `LogSubject` in `Infrastructure/Logging/`
- [ ] Template Console e File con `T:{ThreadId} A:{TaskId} S:{Subject}`
- [ ] Soggetto indicato dall'utente aperto con `LogSubject.Begin` all'ingresso di ogni flusso che lo tratta; errori dell'elemento loggati dentro lo scope
- [ ] Nessun thread, task o soggetto nel testo dei messaggi; nessuna interpolazione nel template dei messaggi
- [ ] `logs/` in `.gitignore`

*Istruzione v1.1 - Serilog - 2026-10-20 — claude-opus-5-5 — filtro del soggetto con driver GELF (docker service logs non basta), rimando condizionale a `dr-devops`. v1.0: contenuto Serilog spostato dal core `logging.instructions.md`; configurazione da `appsettings.json` (container-ready) ed enricher in codice; soggetto del log `Subject`; errori loggati dentro lo scope, cause corrette negli errori comuni, nome del campo solo costante*
