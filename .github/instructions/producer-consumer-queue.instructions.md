---
applyTo: "**/*.cs"
---

# Produttore/consumatore con coda bounded (AI Agent)

Regola obbligatoria quando il codice ha **due fasi**:

- **acquisizione**: uno o più componenti leggono o ricevono elementi da una sorgente (API esterne, caselle di posta, file, tabelle DB, broker, feed);
- **elaborazione**: per ogni elemento c'è del lavoro da fare (trasformare, salvare, applicare regole, scaricare allegati, notificare).

> Chi **acquisisce** non elabora, chi **elabora** non acquisisce. I due lati comunicano solo attraverso una coda FIFO bounded. Il parallelismo si regola **separatamente** sui due lati: di solito pochi produttori (spesso uno, sequenziale) e N worker.

Quando applichi il pattern, dichiaralo nell'output in una riga: "Separo acquisizione ed elaborazione con una coda FIFO bounded: produttore X, N worker per Y".

## Perché

Acquisizione ed elaborazione nello stesso ciclo parallelo (`Parallel.ForEachAsync` o `Task.WhenAll` che fanno "leggi e lavora" insieme):

- il parallelismo pensato per l'elaborazione colpisce la sorgente: quote superate, connessioni esaurite, throttling, lock;
- un solo numero governa due colli di bottiglia diversi, che non si dimensionano più in modo indipendente;
- un errore su un lato blocca l'altro senza una semantica chiara: non si sa cosa è stato acquisito e cosa elaborato;
- senza contropressione l'acquisizione corre più veloce dell'elaborazione e la memoria cresce.

I test con sorgenti finte non vedono il problema, perché la sorgente finta non ha limiti: emerge solo in produzione.

## Quando non si applica

- Una sola operazione per richiesta, senza elaborazione per elemento (un endpoint che legge e risponde).
- Volumi piccoli e fissi, per cui un ciclo sequenziale basta: **dichiaralo** nell'output.
- L'infrastruttura offre già la coda (broker con consumer group, Azure Service Bus, RabbitMQ): usa quella, con gli stessi principi su worker, scope, idempotenza ed errori.

## Elementi obbligatori

1. **Coda FIFO bounded** tra i due lati: `Channel.CreateBounded<T>` con `BoundedChannelFullMode.Wait`. A coda piena il produttore attende (**contropressione**): la memoria resta limitata.
2. **Produttori**: quanti ne servono alla sorgente, spesso uno solo e sequenziale. Ognuno salva il proprio avanzamento (cursore, pageToken, offset) **insieme** agli elementi acquisiti, in modo atomico, così un riavvio riprende dal punto giusto. A fine lavoro, anche in errore, chiude la coda (`Writer.TryComplete()`).
3. **Worker**: N in parallelo, con N da configurazione. Ogni worker apre il **proprio scope DI** (`IServiceScopeFactory.CreateAsyncScope()`): `DbContext` e servizi scoped non sono thread-safe. Contatori condivisi con `Interlocked`; i singleton usati dai worker devono essere thread-safe.
4. **Elementi in coda leggeri**: un identificativo o un DTO immutabile, mai entità tracciate da un altro `DbContext`. Il worker ricarica ciò che gli serve nel proprio scope.
5. **Worker idempotenti**: dopo un crash un elemento può essere rielaborato senza duplicare effetti (file, notifiche, righe).
6. **Gestione degli errori**, tre casi distinti:
   - errore sul **singolo elemento** → si registra e si passa al successivo; la coda non si ferma;
   - errore fatale del **produttore** → la coda si chiude, i worker **finiscono gli elementi già in coda** (già acquisiti e salvati), poi il lavoro termina con l'errore del produttore;
   - errore fatale di un **worker** (credenziali revocate, DB irraggiungibile) → annulla produttori e worker con un `CancellationTokenSource` collegato; risale l'errore del worker, non l'annullamento che ne deriva nel produttore.
7. **Risorse condivise tra i lati**: se produttori e worker chiamano lo stesso servizio con una quota o un rate limit, usano **un solo limitatore** per identità o chiave (es. `PartitionedRateLimiter` con `TokenBucketRateLimiter`, registrato singleton). Il retry con backoff deve riconoscere gli errori di quota del servizio (es. `403 rateLimitExceeded`), non solo 429 e 5xx.
8. **Configurazione**: numero di worker, capienza della coda ed eventuali limiti di quota vengono dalla configurazione, con default dichiarati nella classe di opzioni (segui `no-hardcoded-values.instructions.md`).

### FIFO con N worker

Con N worker l'ordine FIFO vale per **l'uscita dalla coda**, non per l'ordine in cui le elaborazioni finiscono. Se l'ordine di completamento conta (es. eventi sullo stesso aggregato da applicare in sequenza): un solo worker, oppure **partizioni per chiave** con un worker per partizione.

### Lato database

- **Avanzamento atomico**: cursore ed elementi acquisiti nella stessa transazione, cioè un solo `SaveChangesAsync` sullo stesso `DbContext`.
- **Idempotenza nel database**: vincolo `UNIQUE` sulla chiave naturale dell'elemento, più upsert o gestione della violazione. Un controllo applicativo "esiste già?" seguito da un insert non regge con N worker.
- **Riga assente o già elaborata**: il worker che ricarica per id deve tollerarla, senza trattarla come errore fatale.
- **Più produttori sullo stesso cursore**: concorrenza ottimistica sulla riga del cursore (`rowversion` su SQL Server, `xmin` su PostgreSQL).

## Esempio di riferimento

```csharp
// Options/IngestionOptions.cs — sezione "Ingestion" di appsettings.json
public sealed class IngestionOptions
{
    public int WorkerCount { get; set; } = 4;
    public int QueueCapacityPerWorker { get; set; } = 4;
}

// Eccezione con cui il processore segnala un errore che deve fermare tutto
public sealed class FatalPipelineException(string message, Exception? inner = null) : Exception(message, inner);

/// <summary>Acquires items with one sequential producer and processes them with N workers through a bounded FIFO queue.</summary>
public sealed class IngestionPipeline(
    IServiceScopeFactory scopeFactory,
    IOptions<IngestionOptions> options,
    ILogger<IngestionPipeline> logger)
{
    /// <summary>Runs one ingestion cycle; rethrows the producer error or the first fatal worker error.</summary>
    public async Task RunAsync(CancellationToken cancellationToken)
    {
        var settings = options.Value;
        // Coda bounded: a coda piena il produttore attende (contropressione)
        var queue = Channel.CreateBounded<long>(new BoundedChannelOptions(settings.WorkerCount * settings.QueueCapacityPerWorker)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleWriter = true
        });

        // Annullamento comune: l'errore fatale di un worker ferma anche il produttore
        using var fatal = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);

        var producer = ProduceAsync(queue.Writer, fatal.Token);
        var workers = Enumerable.Range(0, settings.WorkerCount)
            .Select(workerId => ConsumeAsync(workerId, queue.Reader, fatal))
            .ToArray();

        Exception? producerError = null;
        try
        {
            await producer;
        }
        catch (OperationCanceledException) when (fatal.IsCancellationRequested && !cancellationToken.IsCancellationRequested)
        {
            // Annullato da un worker: l'errore vero risale da Task.WhenAll qui sotto
        }
        catch (Exception ex)
        {
            // Coda gia' chiusa dal produttore: i worker finiscono gli elementi acquisiti, poi l'errore risale
            producerError = ex;
        }

        await Task.WhenAll(workers);
        if (producerError is not null) ExceptionDispatchInfo.Throw(producerError);
        cancellationToken.ThrowIfCancellationRequested();
    }

    /// <summary>Reads the source sequentially, saves each page with its cursor, enqueues the saved ids.</summary>
    private async Task ProduceAsync(ChannelWriter<long> writer, CancellationToken cancellationToken)
    {
        try
        {
            await using var scope = scopeFactory.CreateAsyncScope();
            var source = scope.ServiceProvider.GetRequiredService<ItemSource>();

            // Una pagina alla volta: verso la sorgente la concorrenza e' 1
            await foreach (var page in source.ReadPagesAsync(cancellationToken))
            {
                // Elementi e cursore nella stessa transazione: un riavvio riprende da qui
                var savedIds = await source.SavePageWithCursorAsync(page, cancellationToken);
                foreach (var id in savedIds)
                {
                    await writer.WriteAsync(id, cancellationToken);
                }
            }
        }
        finally
        {
            // Chiude la coda anche in errore: i worker smaltiscono cio' che e' gia' acquisito
            writer.TryComplete();
        }
    }

    /// <summary>Processes queued ids in its own DI scope; item errors are logged, fatal errors cancel everything.</summary>
    private async Task ConsumeAsync(int workerId, ChannelReader<long> reader, CancellationTokenSource fatal)
    {
        try
        {
            await foreach (var itemId in reader.ReadAllAsync(fatal.Token))
            {
                // Scope per elemento: DbContext e servizi scoped non si condividono tra worker
                await using var scope = scopeFactory.CreateAsyncScope();
                var processor = scope.ServiceProvider.GetRequiredService<ItemProcessor>();
                try
                {
                    await processor.ProcessAsync(itemId, fatal.Token);
                }
                catch (Exception ex) when (ex is not OperationCanceledException and not FatalPipelineException)
                {
                    // Errore del singolo elemento: si registra e la coda prosegue
                    logger.LogWarning(ex, "Item {ItemId} failed on worker {WorkerId}", itemId, workerId);
                }
            }
        }
        catch (OperationCanceledException) when (fatal.IsCancellationRequested)
        {
            // Annullato da un altro worker o dallo stop dell'host: nessun errore proprio da far risalire
        }
        catch (Exception ex)
        {
            // Errore fatale: ferma produttore e altri worker, poi risale come errore principale
            logger.LogError(ex, "Worker {WorkerId} stopped by a fatal error", workerId);
            await fatal.CancelAsync();
            throw;
        }
    }
}
```

`ItemSource` e `ItemProcessor` sono servizi scoped del dominio. `RunAsync` si invoca da un `BackgroundService`, da un job schedulato o da un endpoint che avvia il ciclo: il pattern non dipende da chi lo ospita.

## Anti-pattern

| Codice | Problema |
|---|---|
| `Parallel.ForEachAsync(source, (item, ct) => { leggi; elabora; })` | Il grado di parallelismo dell'elaborazione colpisce la sorgente |
| Acquisizione parallela, poi elaborazione sequenziale | Il contrario di ciò che serve: la sorgente sotto carico, il lavoro lento in fila |
| `Channel.CreateUnbounded<T>()` tra sorgente ed elaborazione | Nessuna contropressione: la memoria cresce con il ritardo dei worker |
| Un `DbContext` iniettato nella classe e usato da più task | `DbContext` non è thread-safe: eccezioni intermittenti o dati corrotti |
| Entità tracciate passate nella coda | Agganciate al `DbContext` del produttore, usate da un altro scope |
| `catch (Exception)` che ingoia tutto nel worker | Un errore fatale diventa N errori per elemento e il lavoro "riesce" |
| Grado di parallelismo o capienza scritti nel codice | Non si regolano senza ricompilare |

## Test minimi

- La concorrenza verso la sorgente è quella dichiarata (es. al massimo 1 chiamata contemporanea): la sorgente finta conta le chiamate in corso.
- Gli elementi già in coda vengono elaborati anche se il produttore fallisce dopo averli accodati.
- L'errore fatale di un worker annulla tutto e risale come errore principale, non come `OperationCanceledException`.
- L'errore sul singolo elemento non ferma la coda: gli altri elementi vengono elaborati.

## ✅ Checklist Post-Generazione

- [ ] Acquisizione ed elaborazione separate da una `Channel` bounded con `FullMode = Wait`
- [ ] Pattern dichiarato nell'output (produttore, N worker, cosa fanno)
- [ ] Produttore: avanzamento salvato insieme agli elementi, coda chiusa in `finally`
- [ ] Worker: scope DI proprio, elementi in coda leggeri (id o DTO immutabile)
- [ ] Idempotenza garantita da un vincolo `UNIQUE` e da upsert, non da un controllo applicativo
- [ ] Tre casi di errore gestiti: elemento, produttore, worker
- [ ] Un solo limitatore condiviso se i due lati usano la stessa quota; retry che riconosce gli errori di quota
- [ ] Numero di worker, capienza e limiti da configurazione, con default dichiarati
- [ ] Test minimi presenti
- [ ] Ordine di completamento: se conta, un solo worker o partizioni per chiave

---

*Istruzione v1.0 - Produttore/consumatore con coda bounded - 2026-10-11 — claude-opus-5-5 (dr-dotnet-backend#2)*
