---
applyTo: "**/*.cs"
---

# Fan-out parallelo su operazioni indipendenti (AI Agent)

Versione .NET dello **Schema A** di `concurrency.instructions.md` (core `dr-guidelines`): un insieme noto di operazioni indipendenti, lanciate insieme con un limite e attese tutte prima di proseguire. Esempio tipico: la stessa richiesta a più provider, oppure un'operazione di I/O per ogni elemento di una lista già in memoria.

Il principio, il controllo di indipendenza e la dichiarazione della serie stanno in `concurrency.instructions.md`. Questo file dice come si scrive in C#.

Quando applichi il pattern, dichiaralo nell'output in una riga: "Fan-out su `<operazione>`: `Parallel.ForEachAsync`, limite da `<classe Options>`".

## Quale istruzione si applica

| Caso | Istruzione |
|---|---|
| Insieme noto di operazioni indipendenti, si attendono tutte | **questa** |
| Flusso di elementi da acquisire da una sorgente ed elaborare uno per uno | `producer-consumer-queue.instructions.md` (fonte unica per `Channel<T>`, non si duplica qui) |
| Operazioni dipendenti, ordine che conta, volume piccolo e fisso senza vantaggio, limite del servizio | serie dichiarata, come da `concurrency.instructions.md`, sezione 6 |

Se gli elementi vanno prima **letti** da una sorgente (API paginata, casella di posta, tabella) e poi elaborati, non è un fan-out: è produttore/consumatore. Leggere ed elaborare nello stesso `Parallel.ForEachAsync` è un anti-pattern.

## Scelta dell'API

| Situazione | API |
|---|---|
| Una collezione di elementi, la stessa operazione per ciascuno | `Parallel.ForEachAsync` con `ParallelOptions.MaxDegreeOfParallelism` da configurazione |
| Poche operazioni **diverse**, note a priori (2-3 chiamate a servizi diversi) | `Task.WhenAll` sui task avviati; il limite è il numero stesso delle operazioni |
| Lavoro di calcolo senza I/O | fuori da questa istruzione (`Parallel.For`/PLINQ); qui si parla di I/O asincrono |

`Task.WhenAll(items.Select(i => DoAsync(i)))` su una collezione **non ha limite**: avvia un task per elemento, tutti insieme. Su una collezione si usa `Parallel.ForEachAsync`.

## Elementi obbligatori

1. **Limite da configurazione**: `MaxDegreeOfParallelism` in una classe di opzioni, con default prudente e validazione (`[Range]`, `ValidateOnStart`). Mai un letterale nel codice, mai `-1` (illimitato). Segui `no-hardcoded-values.instructions.md`.
2. **`CancellationToken` a ogni ramo**: `ParallelOptions.CancellationToken` = token del chiamante; dentro il ramo si usa il token che `Parallel.ForEachAsync` passa al delegate, non quello esterno.
3. **Errore raccolto per ramo**: `Parallel.ForEachAsync` **annulla tutti i rami al primo errore non gestito**. Ogni ramo cattura quindi i propri errori e li registra nel proprio risultato, con l'identificativo del ramo. Un errore non si inghiotte: finisce nel risultato e nel log.
4. **Cancellazione distinta dal timeout**: il timeout di `HttpClient` arriva come `TaskCanceledException`, sottoclasse di `OperationCanceledException`. Il filtro corretto è `catch (Exception ex) when (!ct.IsCancellationRequested)`: il timeout di un ramo è un errore di quel ramo, l'annullamento del chiamante risale e non si registra come errore.
5. **Uno scope DI per ramo**: `IServiceScopeFactory.CreateAsyncScope()` dentro il ramo. `DbContext` e servizi scoped non sono thread-safe e non si condividono fra rami. I client registrati con `IHttpClientFactory` e i singleton thread-safe si possono usare da più rami.
6. **Esito deciso in modo esplicito**: dopo l'attesa il chiamante decide fra risultato completo, parziale dichiarato (log `Warning` con quanti rami sono falliti) o fallimento. Mai un risultato parziale presentato come completo.
7. **Ordine dei risultati**: se conta, ogni ramo scrive nella propria posizione di un array preallocato (indice dell'elemento). Nessuna collezione concorrente da riordinare dopo.

## Esempio di riferimento — `Parallel.ForEachAsync`

```csharp
// Options/ProviderQueryOptions.cs — sezione "ProviderQuery" di appsettings.json
public sealed class ProviderQueryOptions
{
    [Range(1, 32)]
    public int MaxDegreeOfParallelism { get; set; } = 4;
}

// Program.cs
builder.Services.AddOptions<ProviderQueryOptions>()
    .BindConfiguration("ProviderQuery")
    .ValidateDataAnnotations()
    .ValidateOnStart();

/// <summary>Outcome of one fan-out branch: either a quote or the error kind, never both.</summary>
public sealed record ProviderResult(string ProviderId, Quote? Quote, string? ErrorKind);

/// <summary>Queries every provider in parallel within the configured limit and returns one result per provider.</summary>
public sealed class QuoteAggregator(
    IServiceScopeFactory scopeFactory,
    IOptions<ProviderQueryOptions> options,
    ILogger<QuoteAggregator> logger)
{
    /// <summary>Returns results in the same order as <paramref name="providerIds"/>; caller cancellation propagates.</summary>
    public async Task<ProviderResult[]> QueryAllAsync(
        IReadOnlyList<string> providerIds,
        QuoteRequest request,
        CancellationToken cancellationToken)
    {
        // Una posizione per ramo: l'ordine resta quello dell'input, nessuna collezione concorrente
        var results = new ProviderResult[providerIds.Count];
        var parallelOptions = new ParallelOptions
        {
            MaxDegreeOfParallelism = options.Value.MaxDegreeOfParallelism,
            CancellationToken = cancellationToken
        };

        await Parallel.ForEachAsync(Enumerable.Range(0, providerIds.Count), parallelOptions, async (index, ct) =>
        {
            var providerId = providerIds[index];

            // Scope per ramo: servizi scoped e DbContext non si condividono fra rami
            await using var scope = scopeFactory.CreateAsyncScope();
            try
            {
                // Risoluzione dentro il try: un provider che non si costruisce e' un errore del suo ramo
                var provider = scope.ServiceProvider.GetRequiredKeyedService<IQuoteProvider>(providerId);
                var quote = await provider.GetQuoteAsync(request, ct);
                results[index] = new ProviderResult(providerId, quote, null);
            }
            catch (Exception ex) when (!ct.IsCancellationRequested)
            {
                // Errore del ramo (timeout compreso): registrato, gli altri rami proseguono
                logger.LogWarning(ex, "Provider {ProviderId} failed", providerId);
                results[index] = new ProviderResult(providerId, null, ex.GetType().Name);
            }
        });

        return results;
    }
}
```

Il chiamante decide l'esito:

```csharp
var results = await aggregator.QueryAllAsync(providerIds, request, cancellationToken);
var failed = results.Count(r => r.ErrorKind is not null);

if (failed == results.Length)
{
    throw new ProviderUnavailableException("No provider answered");
}
if (failed > 0)
{
    logger.LogWarning("Partial result: {Failed} of {Total} providers failed", failed, results.Length);
}
```

## Esempio di riferimento — `Task.WhenAll` su operazioni diverse

```csharp
// Due chiamate a servizi diversi, note a priori: il limite e' il loro numero (2)
var customerTask = customerClient.GetAsync(customerId, cancellationToken);
var ordersTask = orderClient.ListAsync(customerId, cancellationToken);

try
{
    await Task.WhenAll(customerTask, ordersTask);
}
catch (Exception) when (!cancellationToken.IsCancellationRequested)
{
    // Task.WhenAll rilancia solo il primo errore: l'esito di ogni ramo si legge dal suo task
}

// IsCompletedSuccessfully, non solo IsFaulted: il timeout di HttpClient lascia il task Canceled, non Faulted
if (!customerTask.IsCompletedSuccessfully) logger.LogWarning(customerTask.Exception, "Customer {CustomerId} lookup failed or timed out", customerId);
if (!ordersTask.IsCompletedSuccessfully) logger.LogWarning(ordersTask.Exception, "Orders of {CustomerId} lookup failed or timed out", customerId);
```

`.Result` si legge solo dai task con `IsCompletedSuccessfully`: su un task fallito o annullato rilancia l'eccezione.

Vale solo se le due operazioni non usano la stessa risorsa non thread-safe. Due repository sullo stesso `DbContext` iniettato **non** si eseguono così: o uno scope per ramo, o serie dichiarata.

## Anti-pattern

| Codice | Problema |
|---|---|
| `foreach (var id in ids) { await client.GetAsync(id, ct); }` su elementi indipendenti, senza dichiarazione | Tempo totale = somma delle attese; la serie non è dichiarata |
| `await Task.WhenAll(ids.Select(id => client.GetAsync(id, ct)))` su una collezione | Nessun limite: un task per elemento, quote e connessioni saturate |
| `MaxDegreeOfParallelism = 8` scritto nel codice, oppure `-1` | Non si regola senza ricompilare / parallelismo illimitato |
| Ramo di `Parallel.ForEachAsync` senza `try/catch` | Il primo errore annulla tutti gli altri rami: lavoro buttato senza decisione |
| `catch (Exception ex) when (ex is not OperationCanceledException)` | Il timeout di `HttpClient` (`TaskCanceledException`) sfugge alla raccolta e annulla tutto |
| `DbContext` iniettato nella classe e usato dentro i rami | `DbContext` non è thread-safe: eccezioni intermittenti o dati corrotti |
| Token esterno usato dentro il ramo al posto di quello del delegate | Il ramo non vede l'annullamento deciso da `Parallel.ForEachAsync` |
| `ConcurrentBag` riordinato a fine ciclo | Ordine perso e lavoro in più: array preallocato per indice |
| Leggere da una sorgente ed elaborare nello stesso `Parallel.ForEachAsync` | È produttore/consumatore: segui `producer-consumer-queue.instructions.md` |

## Test minimi

- Il limite è rispettato: il servizio finto conta le chiamate contemporanee e non supera `MaxDegreeOfParallelism`.
- Un ramo che fallisce non ferma gli altri: i rami restanti producono il loro risultato, quello fallito ha `ErrorKind`.
- Il timeout di un ramo (`TaskCanceledException` senza annullamento del chiamante) è registrato come errore del ramo.
- L'annullamento del chiamante risale come `OperationCanceledException` e non produce risultati di errore.
- L'ordine dei risultati coincide con quello dell'input.

## ✅ Checklist Post-Generazione

- [ ] Controllo di indipendenza eseguito (`concurrency.instructions.md`, sezione 2); se in serie, riga "Resto in serie: <motivo>"
- [ ] Pattern dichiarato nell'output (operazione, API, classe di opzioni)
- [ ] Collezione → `Parallel.ForEachAsync`; `Task.WhenAll` solo su poche operazioni diverse note a priori
- [ ] `MaxDegreeOfParallelism` da configurazione, validato, con default prudente
- [ ] `CancellationToken` del delegate usato in ogni ramo
- [ ] Errore catturato per ramo con `when (!ct.IsCancellationRequested)`, registrato nel risultato e nel log
- [ ] Uno scope DI per ramo; nessun `DbContext` condiviso
- [ ] Esito completo / parziale / fallito deciso in modo esplicito dal chiamante
- [ ] Test minimi presenti

---

*Istruzione v1.1 - Fan-out parallelo - 2026-10-16 — claude-opus-5-5 (dr-dotnet-backend#4) — esempio `Task.WhenAll`: ramo in timeout (Canceled) registrato come errore*
