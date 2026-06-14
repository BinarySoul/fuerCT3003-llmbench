# fuerCT3003-llmbench
Ein beispiel für eine geführte Anwendung um die Qualität von Coding LLM´s zu testen.


# llmBench – Benchmark für skalierbare Coding-LLMs (Rust)

## Ziel

llmBench ist eine Referenzanwendung zur Bewertung von Coding-LLMs unter realistischen Bedingungen für datenintensive Verarbeitung.

Im Fokus stehen nicht nur syntaktisch korrekte Programme, sondern:

* Systemarchitektur
* Parallelisierung
* Speicher- und Ressourcenmanagement
* deterministische Datenverarbeitung
* skalierbare Algorithmen (auch über RAM-Grenzen hinaus)

Die Anwendung ist in **Rust** zu implementieren.

---

# Grundprinzip

Die Anwendung verarbeitet große Mengen an Textdaten aus einem dynamischen Input-Verzeichnis:

```text
/testfiles/
```

Die Anzahl der Dateien ist **nicht fest vorgegeben** und kann stark variieren (z. B. 5 bis 5000+ Dateien).

---

# Allgemeine Regeln

## Datei-Erkennung

Die Anwendung muss automatisch alle `.txt` Dateien in `/testfiles/` erkennen.

Ausnahmen:

* `reference.txt` darf **niemals** in die Verarbeitung einbezogen werden

---

## Datenmodell

Jede Datei enthält strukturierte Datensätze im Format:

```text
Postleitzahl;Ort;Telefonnummer;Vorname;Name;Alter
```

---

# Betriebsmodi

Die Anwendung unterstützt drei Modi:

```bash
llmBench --live
llmBench --check
llmBench --read
llmBench --create
```

---

# Architekturvorgabe (zwingend)

Die Anwendung muss eine **geführte, hierarchische Pipeline-Architektur** implementieren.

---

## 1. Worker Layer (Parallelisierungsebene)

* Alle verfügbaren CPU-Threads werden ermittelt
* Maximal werden genutzt:

```text
CPU-Threads - 2
```

* Diese Threads sind reine **Worker**

### Aufgaben der Worker:

* Lesen von Datei-Chunks
* lokale Normalisierung der Datensätze
* lokale Sortierung
* lokale Duplikaterkennung innerhalb des Chunks
* Weiterleitung der Ergebnisse an Orchestratoren

---

## 2. Orchestrator Layer (2 Instanzen zwingend)

Es müssen **genau zwei Orchestrator-Threads** existieren.

### Aufgaben:

* Empfang von Worker-Daten
* Zwischensortierung / Merge von Chunks
* Aufbau konsistenter Zwischenzustände
* Weiterleitung an finalen Orchestrator

Die Orchestratoren arbeiten unabhängig voneinander auf getrennten Datenströmen.

---

## 3. Finaler Orchestrator (1 Instanz zwingend)

Der finale Orchestrator:

* empfängt Ergebnisse beider Orchestratoren
* führt globale Konsolidierung durch
* entfernt alle globalen Duplikate
* stellt deterministische Gesamtsortierung sicher
* schreibt das finale Ergebnis in `/export/`

---

## Architekturregel

Die Verarbeitung muss strikt hierarchisch erfolgen:

```text
Workers → Orchestrator A / B → Final Orchestrator → Export
```

Dabei gilt:

* kein globales Shared-State-HashSet
* keine unsynchronisierte Parallel-Write-Struktur
* keine Race-Condition-abhängigen Ergebnisse

---

# Betriebsmodus: --live

## Ziel

Vollständige Datenverarbeitung.

---

## Ablauf

1. Erkennung aller Dateien in `/testfiles/`
2. Ausschluss von `reference.txt`
3. Aufbau der Pipeline
4. Parallelverarbeitung der Daten
5. Sortierung + Duplikatentfernung
6. Export in `/export/`

---

## Ergebnis

Eine einzige konsolidierte Datei:

```text
/export/final_output.txt
```

---

# Betriebsmodus: --check

## Ziel

Systemanalyse ohne Datenverarbeitung

Ausgabe:

* CPU-Threads gesamt
* verwendete Worker-Threads
* Anzahl Orchestratoren (fix: 2)
* RAM gesamt
* verfügbares RAM
* RAM-Arbeitsbudget (RAM - 4 GB)
* geplante Pipeline-Konfiguration

---

Beispiel:

```text
CPU-Threads: 16
Worker: 14
Orchestratoren: 2
Finaler Orchestrator: 1

RAM: 64 GB
Arbeitsbudget: 60 GB

Pipeline: aktiv (hierarchisch)
```

---

# Betriebsmodus: --read

## Ziel

Validierung der Ergebnisse

---

## Vergleich

Die Anwendung vergleicht:

* `/testfiles/reference.txt`
* `/export/final_output.txt`

---

## Auswertung

Mindestens folgende Metriken:

* Zeilenanzahl
* fehlende Datensätze
* zusätzliche Datensätze
* verbleibende Duplikate
* Sortierfehler
* Inhaltsfehler

---

## Beispielausgabe

```text
Referenz: 1.550.646 Datensätze
Export:   1.549.900 Datensätze

Fehlend: 746
Zusätzlich: 0
Duplikate: 3
Sortierfehler: 0
Inhaltsfehler: 12
```

---

# Betriebsmodus: --create

## Ziel

Generierung eines vollständigen Testdatensatzes inklusive Referenz und kontrollierter Duplikatinjektion.

---

## Ablauf

### 1. Benutzerinteraktion

Das Programm fragt interaktiv:

* Anzahl der Dateien
* Größe pro Datei (Anzahl Datensätze oder Dateigröße)

---

### 2. Datengenerierung

Für jede Datei werden synthetische Datensätze erzeugt:

```text
Postleitzahl
Ort
Telefonnummer
Vorname
Name
Alter
```

Die Verteilung soll realistisch wirken, jedoch vollständig fiktiv sein.

---

### 3. Schreibprozess

* Dateien werden in `/testfiles/` erstellt
* jede Datei enthält exakt die definierte Datenmenge

---

### 4. Referenzgenerierung

Nach Erstellung aller Dateien wird:

* eine vollständige `reference.txt` erzeugt
* diese enthält den **korrekten Merge aller Dateien**
* sortiert und ohne Duplikate

---

### 5. Duplikatinjektion

Anschließend werden zufällig ausgewählte Dateien manipuliert:

* 2–5 % aller Datensätze im Gesamtdatensatz werden dupliziert
* Duplikate werden gezielt in einzelne Dateien eingefügt
* `reference.txt` bleibt unverändert

---

## Wichtige Regel

* `reference.txt` ist immer der **Ground Truth**
* alle anderen Dateien sind potenziell fehlerhaft
* Fehler sollen reproduzierbar, aber zufällig verteilt sein

---

# Bewertungsziele des Benchmarks

Die Aufgabe ist darauf ausgelegt, echte Unterschiede in der Qualität von Coding-LLMs sichtbar zu machen.

---

## 1. Architekturqualität

* klare Trennung der Pipeline-Stufen
* saubere Verantwortlichkeiten
* keine globalen Race-Condition-Strukturen

---

## 2. Parallelisierung

* effektive Nutzung mehrerer CPU-Kerne
* stabile Worker-Orchestrator-Kommunikation
* Vermeidung von Bottlenecks

---

## 3. Skalierbarkeit

* Verarbeitung unabhängig von Dateianzahl
* Chunking bei großen Datenmengen
* kein vollständiges RAM-Laden zwingend erforderlich

---

## 4. Algorithmik

* externe oder interne Sortierung
* effiziente Duplikatentfernung
* deterministische Merge-Strategien

---

## 5. Datenkonsistenz

* korrekte Behandlung von Duplikaten
* exakte Rekonstruktion der Referenzstruktur
* valide Fehlererkennung im --read Modus

---

## Erwartetes Ergebnis

Ein starkes Coding-LLM erzeugt eine Lösung, die:

* hochparallel arbeitet
* Speicher bewusst begrenzt nutzt
* deterministisch bleibt
* skalierbar auf große Datenmengen ist
* eine klare Pipeline-Architektur implementiert

---

# Kernidee des Benchmarks

Der Benchmark ist so gestaltet, dass einfache Lösungen (z. B. „alles in HashSet laden“) nicht mehr funktionieren.

Stattdessen müssen LLMs:

* echte Systemarchitektur entwerfen
* Pipeline-Design verstehen
* Parallelisierung korrekt koordinieren
* globale Konsistenz trotz lokaler Verarbeitung sicherstellen

Damit wird nicht nur Codequalität, sondern echte Software-Engineering-Kompetenz messbar.



# llmBench – Referenzarchitektur (Rust)

## Ziel der Referenz

Diese Referenz beschreibt eine **idealtypische Implementierung** von llmBench.

Sie dient als Vergleichsmaßstab für LLM-generierten Code hinsichtlich:

* Architekturqualität
* Parallelisierungsmodell
* Datenfluss-Korrektheit
* Skalierbarkeit
* Determinismus

---

# Gesamtsystemübersicht

## Pipeline-Struktur

```text id="a1p7kx"
                +----------------------+
                |   File Discovery     |
                +----------+----------+
                           |
                           v
                +----------------------+
                |     Worker Pool     |
                | (CPU-Threads - 2)   |
                +----------+----------+
                           |
            +--------------+--------------+
            |                             |
            v                             v
   +------------------+        +------------------+
   | Orchestrator A   |        | Orchestrator B   |
   | (Merge Layer 1)  |        | (Merge Layer 1)  |
   +---------+--------+        +---------+--------+
             \                      /
              \                    /
               v                  v
              +----------------------+
              | Final Orchestrator  |
              | (Global Reduce)     |
              +----------+----------+
                         |
                         v
                +----------------------+
                |     Export File      |
                +----------------------+
```

---

# Modulstruktur (Rust-Design)

## Empfohlene Projektstruktur

```text id="mod1"
src/
 ├── main.rs
 ├── cli/
 │    ├── mod.rs
 │    ├── args.rs
 │    └── mode.rs
 │
 ├── discovery/
 │    ├── mod.rs
 │    └── scanner.rs
 │
 ├── model/
 │    ├── record.rs
 │    └── file_chunk.rs
 │
 ├── worker/
 │    ├── mod.rs
 │    ├── pool.rs
 │    └── processor.rs
 │
 ├── orchestrator/
 │    ├── mod.rs
 │    ├── layer1.rs
 │    ├── final.rs
 │
 ├── pipeline/
 │    ├── mod.rs
 │    └── coordinator.rs
 │
 ├── generator/
 │    ├── mod.rs
 │    └── create.rs
 │
 ├── validation/
 │    ├── mod.rs
 │    └── diff.rs
 │
 └── util/
      ├── io.rs
      ├── sort.rs
      └── dedupe.rs
```

---

# Datenmodell

## Record-Struktur

```text id="rec1"
Record {
    postal_code: String,
    city: String,
    phone: String,
    first_name: String,
    last_name: String,
    age: u8
}
```

Alle Operationen arbeiten auf diesem Modell.

---

# Kernpipeline

## 1. File Discovery Layer

### Verantwortlichkeiten:

* Scannen von `/testfiles/`
* Filtern von `reference.txt`
* Erzeugen einer Liste von Input-Dateien

### Output:

```text id="disc1"
Vec<PathBuf>
```

---

## 2. Worker Layer (Map Phase)

### Eigenschaften:

* parallel (ThreadPool)
* begrenzt auf:

```text id="cpu1"
CPU - 2 Threads
```

### Aufgaben pro Worker:

* Datei lesen oder chunk streamen
* Parsing → `Record`
* lokale Sortierung
* lokale Duplikatentfernung
* Ausgabe als sortierter Chunk

### Output:

```text id="worker1"
SortedChunk
```

---

## 3. Orchestrator Layer (Level 1)

Zwei Instanzen:

* Orchestrator A
* Orchestrator B

### Aufgaben:

* Empfang von Worker-Chunks
* k-way merge
* Zwischen-Deduplikation
* Aufbau eines stabilen sortierten Streams

### Output:

```text id="orch1"
IntermediateSortedStream
```

---

## 4. Final Orchestrator (Reduce Phase)

### Aufgaben:

* Merge von A + B Streams
* globale Sortierung (falls nötig)
* globale Duplikatentfernung
* finaler Konsolidierungscheck

### Output:

```text id="final1"
Vec<Record> (final sorted dataset)
```

---

## 5. Export Layer

* schreibt `/export/final_output.txt`
* deterministische Reihenfolge
* Streaming write bevorzugt

---

# Memory-Strategie

## Regel

Das System darf NICHT davon ausgehen, dass alles in RAM passt.

---

## Strategie

### Klein genug:

```text id="mem1"
alles im RAM
```

### Groß:

```text id="mem2"
chunked processing + external merge
```

---

## Prinzip

> Always fallback to streaming.

---

# Parallelisierungsmodell

## Worker Pool

* dynamisch basierend auf CPU
* keine globalen Locks für Datenstrukturen
* Kommunikation ausschließlich über Channels

---

## Orchestrator Kommunikation

Empfohlen:

* bounded channels
* backpressure support

```text id="chan1"
Worker → Orchestrator (channel)
Orchestrator → Final (channel)
```

---

## Wichtige Regel

Kein Shared Mutable Global State für Datenverarbeitung.

---

# Sortier- & Merge-Strategie

## Lokale Sortierung

* innerhalb Worker
* O(n log n)

---

## Global Merge

* k-way merge
* streaming merge

---

## Ziel

Deterministisches Ergebnis unabhängig von:

* Thread timing
* OS scheduling
* CPU load

---

# Duplikatstrategie

## Definition

Duplikate sind identische Records über alle Felder.

---

## Regeln

* lokale Duplikate: Worker Phase
* globale Duplikate: Final Orchestrator

---

## Technische Umsetzung

* HashSet nur innerhalb begrenzter Scope-Lebensdauer erlaubt
* finale Konsistenz über sortierten Stream + linear scan

---

# --create Modul

## Zweck

Generierung von Testdaten + Referenz + Fehlerinjektion

---

## Ablauf

### 1. User Input

* Anzahl Dateien
* Größe pro Datei

---

### 2. Generator

Erzeugt synthetische Records:

* PLZ
* Ort
* Telefonnummer
* Vorname
* Name
* Alter

---

### 3. Datei-Erstellung

* Schreibvorgang in `/testfiles/`
* gleichmäßige Verteilung

---

### 4. Reference Build

* vollständiger Merge aller generierten Daten
* sortiert
* dedupliziert
* gespeichert als `reference.txt`

---

### 5. Noise Injection

* 2–5% Duplikate global
* gezielt in zufällige Dateien verteilt
* reference.txt bleibt unverändert

---

# --check Modul

* Systemanalyse
* keine Verarbeitung
* Ausgabe von:

  * CPU / RAM
  * Worker Limit
  * Pipeline Topologie

---

# --read Modul

## Vergleich

* reference.txt vs final_output.txt

---

## Metriken

* missing records
* extra records
* duplicates
* sort errors
* field mismatch errors

---

# Bewertungsdimensionen

## 1. Architektur

* klare Layer
* saubere Trennung
* keine God-Structs

---

## 2. Parallelität

* echte Work Distribution
* kein global lock bottleneck
* stabile Pipeline

---

## 3. Skalierbarkeit

* funktioniert mit 5 bis 50.000 Dateien
* RAM-unabhängig skalierbar

---

## 4. Determinismus

* identisches Ergebnis unabhängig von Thread timing

---

## 5. Systemdesign

* Backpressure vorhanden
* keine Overcommitment-Strategien
* kontrollierte Ressourcenverwendung

---

# Kernaussage des Benchmarks

Dieser Benchmark misst nicht nur „ob Code funktioniert“, sondern:

> ob ein Modell ein echtes skalierbares Datenverarbeitungssystem entwerfen kann.

Er trennt:

* einfache Implementierung (In-Memory + HashSet)
* mittlere Architektur (Threadpool + Merge)
* professionelle Systeme (Pipeline + External Merge + Backpressure)

---

# Ergebnis

Die Referenzarchitektur definiert ein System, das:

* parallel
* skalierbar
* speicherkontrolliert
* deterministisch
* fehlertolerant

arbeitet und somit als objektiver Maßstab für Coding-LLMs dient.
