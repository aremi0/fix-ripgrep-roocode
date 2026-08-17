# Fix: Ripgrep (`rg.exe`) non trovato da Roo Code su VS Code (Windows)

## Installazione del Database Vettoriale Docker (Qdrant)
Apri PowerShell ed esegui:
```powershell
docker run -d `
  --name qdrant-vector-db `
  --restart unless-stopped `
  -p 6333:6333 `
  -p 6334:6334 `
  -v qdrant_storage:/qdrant/storage `
  qdrant/qdrant
```

## Installazione di Ripgrep su Windows (Opzionale ma consigliata)
Apri PowerShell ed esegui:
```powershell
winget install BurntSushi.ripgrep
```

### Il Problema
A seguito di alcuni aggiornamenti di VS Code, l'eseguibile di **ripgrep** (`rg.exe`) viene spostato in percorsi diversi o rimosso dalle posizioni standard. Estensioni come **Roo Code** continuano a cercarlo nel vecchio percorso hardcoded (`...@vscode\ripgrep\bin\rg.exe`), causando il fallimento della ricerca nel codebase e dell'indicizzazione.

### La Soluzione
La soluzione consiste nel rintracciare dove si trova attualmente `rg.exe` all'interno dell'installazione locale di VS Code e copiarlo nel percorso esatto che l'estensione si aspetta, ricreando la cartella mancante.

Nelle versioni recenti, la vera applicazione si trova spesso in una sottocartella che riporta l'hash della build (es. `a5b5009513`). 

### Risoluzione rapida via PowerShell

Lo script seguente cerca automaticamente l'hash della build attiva, trova il file `rg.exe` (ad esempio tra i pacchetti di Copilot o ripgrep-universal) e lo copia nella directory corretta.

1. Apri **PowerShell** (non servono permessi di amministratore se VS Code è installato a livello utente).
2. Incolla ed esegui il seguente blocco di codice:

```powershell
# 1. Trova l'hash della build attiva (la cartella con risorse complete)
$baseVsCodeDir = "$env:LOCALAPPDATA\Programs\Microsoft VS Code"
$appDirs = Get-ChildItem -Path$baseVsCodeDir -Directory | Where-Object { Test-Path "$($_.FullName)\resources\app" }
$activeBuildDir =$appDirs | Select-Object -First 1

if (-not $activeBuildDir) {
    Write-Host "Impossibile trovare la cartella della build di VS Code." -ForegroundColor Red
    exit
}

# 2. Definisci il percorso in cui Roo Code cerca rg.exe
$targetDir = "$($activeBuildDir.FullName)\resources\app\node_modules.asar.unpacked\@vscode\ripgrep\bin"

# 3. Cerca un rg.exe esistente nell'installazione
$sourceRg = (Get-ChildItem -Path$baseVsCodeDir -Filter "rg.exe" -Recurse -ErrorAction SilentlyContinue | Select-Object -First 1).FullName

if ($sourceRg) {
    # 4. Crea la cartella se non esiste
    if (-not (Test-Path $targetDir)) {
        New-Item -ItemType Directory -Force -Path $targetDir | Out-Null
    }
    
    # 5. Copia il file
    Copy-Item -Path $sourceRg -Destination "$targetDir\rg.exe" -Force
    Write-Host "Fix applicato con successo!" -ForegroundColor Green
    Write-Host "Copiato da: $sourceRg" -ForegroundColor DarkGray
    Write-Host "Destinazione: $targetDir\rg.exe" -ForegroundColor Cyan
} else {
    Write-Host "Impossibile trovare rg.exe nell'installazione di VS Code." -ForegroundColor Red
}
```