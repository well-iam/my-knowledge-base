# Guida a posh-git

**posh-git** è un modulo per PowerShell che arricchisce l'interfaccia da riga di comando con informazioni contestuali in tempo reale sullo stato del repository Git e abilita il completamento automatico avanzato con `Tab`.
## 1. Installazione e Configurazione
Esegui i passaggi direttamente nella finestra di PowerShell:
```PowerShell
# 1. Installa il modulo per l'utente corrente
Install-Module posh-git -Scope CurrentUser -Force

# 2. Apri il file di configurazione profilo
notepad $PROFILE
```

All'interno del blocco note, aggiungi la riga:
```PowerShell
Import-Module posh-git
```

Salva il file (`Ctrl + S`), chiudi Notepad e applica le modifiche:
```PowerShell
. $PROFILE
```

## 2. Decodifica dello Stato nel Prompt
La struttura classica del prompt segue questo schema:
`[<nome-branch> <stato-sync> | <staging-area> | <working-tree> <stato-finale>]`

Esempio: `[feature/login ≡ +1 ~0 -0 | +0 ~2 -0 !]`

| **Elemento** | **Colore** | **Posizione**     | **Significato**                                         |
| ------------ | ---------- | ----------------- | ------------------------------------------------------- |
| **`≡`**      | Ciano      | Sync              | Locale e remoto sono allineati allo stesso commit       |
| **`↑n`**     | Ciano      | Sync              | Avanti di $n$ commit locali non ancora inviati (`push`) |
| **`↓n`**     | Ciano      | Sync              | Indietro di $n$ commit remoti da scaricare (`pull`)     |
| **`+n`**     | Verde      | Staging (sx)      | $n$ nuovi file aggiunti con `git add`                   |
| **`~n`**     | Verde      | Staging (sx)      | $n$ file modificati pronti per il commit                |
| **`-n`**     | Verde      | Staging (sx)      | $n$ file rimossi pronti per il commit                   |
| **`\|`**     | Grigio     | Centro            | Separatore tra Staging Area e Working Tree              |
| **`+n`**     | Rosso      | Working Tree (dx) | $n$ file non tracciati (_untracked_) sul disco          |
| **`~n`**     | Rosso      | Working Tree (dx) | $n$ file modificati non ancora indicizzati              |
| **`-n`**     | Rosso      | Working Tree (dx) | $n$ file cancellati ma non rimossi dall'indice          |
| **`!`**      | Rosso      | Fine prompt       | Presenza di modifiche non in staging nel working tree   |
| **`~`**      | Verde      | Fine prompt       | Working tree pulito; modifiche pronte per `git commit`  |

## 3. Funzionalità di Autocompletamento

> [!TIP] Scorciatoie da tastiera
> 
>   
> 
> - **Completamento Branch (`Tab`):** Digita `git switch feat` e premi `Tab` per scorrere i branch locali e remoti (`origin/*`).
>     
>       
>     
> - **Menu Grafico Interattivo (`Ctrl + Spazio`):** Digita `git switch` o `git merge` e premi `Ctrl + Spazio` per aprire l'elenco navigabile con le frecce direzionali.
>     
>       
>     
> - **Opzioni e Comandi:** Completa automaticamente comandi lunghi, opzioni CLI (es. `--set-upstream`, `--force-with-lease`) e hash abbreviati di commit.
>