---
applyTo: "**"
---

# Progetto .NET — Rilevamento del tipo e convenzioni di base

Si applica a ogni progetto in cui è installato `dr-dotnet-backend`. Target: **.NET 10**.

Rileva il tipo dal codice prima di generare o modificare codice applicativo.

---

## Tipo di progetto

| Segnale nel codice | Tipo | Istruzione modulare | Pacchetto che la porta |
|--------------------|------|---------------------|------------------------|
| `Workers/*.cs` presente | Windows Service | `windows-service.instructions.md` | `dr-winsvc` |
| `Endpoints/*.cs` presente | Minimal API | `minimal-api-architecture.instructions.md` | `dr-minimalapi` |
| Entrambi presenti (`Workers/` e `Endpoints/`) | Soluzione multi-progetto | Leggi entrambe le istruzioni modulari | `dr-winsvc` + `dr-minimalapi` |
| `package.json` presente, nessun `.csproj` | Frontend | `frontend-organization.instructions.md` | `dr-fe` |
| Nessun segnale riconoscibile | Tipo non rilevato | Fermati. Chiedi: "Questo è un Minimal API o un Windows Service?" | — |

Se l'istruzione modulare corrispondente al tipo rilevato non è presente in `.github/instructions/`, il pacchetto che la contiene non è installato: segnalalo e proponi di installarlo, invece di procedere a memoria.

Leggi sempre l'istruzione modulare corretta prima di generare o modificare codice.

---

## Convenzioni .NET

- **Primary constructors** per l'iniezione delle dipendenze
- `async`/`await` per ogni operazione di I/O, con `CancellationToken` propagato
- Logging strutturato con placeholder — mai interpolazione di stringa nel log
- Naming: namespace `snake_case`, classi `PascalCase`, variabili `camelCase`
- Validazione input: ogni endpoint con body usa `IValidator<T>`; segui `input-validation.instructions.md`

---

## File di progetto comuni

`dr-dotnet-backend` installa nella radice del progetto host:

| File | Contenuto |
|------|-----------|
| `Directory.Build.props` | Impostazioni MSBuild condivise da tutti i progetti della solution: `TargetFramework`, `LangVersion`, `Nullable`, `ImplicitUsings`, analisi del codice, configurazioni Debug/Release |
| `global.json` | Versione dell'SDK .NET e politica di roll-forward |

Modifica questi file nel progetto host quando servono impostazioni specifiche (per esempio `Authors`, `Company`, `Product`): l'installer non li sovrascrive senza `-Update` esplicito.

---

## Gate di Push — Lint .NET

⛔ Prima di qualsiasi `git push` su un progetto .NET:

```powershell
dotnet format <percorso>.csproj --verify-no-changes
```

| Exit code | Azione |
|-----------|--------|
| `0` | Lint pulito — push consentita |
| Non-zero | **BLOCCA la push** — elenca i file con violazioni, chiedi conferma prima di correggere, poi riesegui |

I template `dotnet new` scrivono UTF-8 **con BOM**: su un progetto appena creato serve un `dotnet format` in scrittura prima del primo push, altrimenti il check fallisce.

Questo comando è il "comando di verifica dichiarato dall'istruzione di dominio" a cui rimanda il gate di push generico in `.github/copilot-instructions.md`.

---

## Checklist Post-Generazione

- [ ] Tipo rilevato correttamente, istruzione modulare letta
- [ ] Primary constructors e `async`/`await` usati dove servono
- [ ] Logging strutturato con placeholder
- [ ] `dotnet format --verify-no-changes` pulito prima della push

---

*Istruzione v1.0 - Progetto .NET - 2026-09-16 — claude-opus-5 — contenuto .NET scorporato dal core `dr-guidelines`*
