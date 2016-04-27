# Vorlesung Betriebssysteme 1

## Organisatorisches, Vorlesung vom 16.03.2016
1. Praktikum
    1. Start vsl. 04.04.2016
    1. Moodle-Passwort: DOS1.0
1. Themen
    1. Übersicht
    1. Grundlagen Rechnerarchitektur (RA)
    1. Prozesse, Threads, Interrupts
    1. Scheduling
    1. Synchronisation
    1. Memory Management (File System)
    1. (I/O)
    1. (Security)
    1. (Design & Implementation)
    1. (Virtualisierung)
    1. (Multiprocessors)
1. Klausur gemeinsam mit Vogt
1. Literatur
    1. Tanenbaum: Modern OS
    1. Silberschatz: OS Concepts
    1. Bovet: Understanding the Linux Kernel
    1. Russinovich: Windows Internals
1. Themen der heutigen Vorlesung
    1. Geschichte von Unix und Windows
    1. Was ist bzw. macht ein OS?
    1. User Mode und Kernel Mode

## Grundlagen Rechnerarchitektur (RA), 23.03.2016

### Wozu braucht das OS Hardware-Unterstützung?
Wiederholung: Wozu braucht man ein OS?
1. Schnittstelle für Systemprogramme
1. Schnittstelle für Anwenderprogramme
1. UI
1. Antwort: **Verwendung von Ressourcen, Rechten, Synchronisation und Kommunikation**

### Rechte
Die CPU hat einen Kernel-/Privileged-Mode und einen User Mode:

Kernel-/Privileged-Mode | User Mode 
---- | ---- 
alle Rechte | eingeschränkte Rechte |
x86 Real Mode: ohne Rechteverwaltung, Protected Mode: 4 priviledged Level | Tendenz: möglichst wenig im Kernel Mode machen (micro Kernel), User Mode Driver (Windows), User Mode File System (Unix) 
MMIX: negative Adressen | MMIX: positive Adressen 
Beschränkte Einsprungpunkte inkl. Rechtekontrolle | Wie kommt man in den Kernel Mode? a) Syscalls, interne Spezial-Instruktionen b) Interrupts; DTRAP/FTRAP, synchr. Interrupts Einschränkungen
 | Einschränkungen des Registerzugriffs; MMIX: rK, x86: PSW (program status word enthält die Mode Bits)
 | Einschränkungen bei den Instruktionen; x86 in/out (Zugriff auf Ports) `OR AX, 0xFF` und `OUT #21, AX`. `#21` ist das Interrupt Mask Register des PIC (Program Interrupt Controller)

### Ressourcen
1. CPU-Zeit: Userprozess darf CPU nur für eine bestimmte Zeit nutzen. Das OS hat und holt sich die Kontrolle zurück:
    1. syscall ==> Userprogramm muss warten
    1. timer interrupt; time slice läuft ab ==> Prozesswechsel
1. Memory
1. (Segmentation)
1. page tables (PT) ==> Im User Mode immer nur so: virtuelle Adressen (Programm/CPU), physikalische Adressen (Bus/RAM)
    1. Die PT ist eine Tabelle, die für jede *page number* eine *frame number* und außerdem Zusatzinformationen enthält.
    1. *page number*: `20 Bit + 12 Bit` offset
    1. 4 GB RAM und 32 Bit Adressraum ==> ca. 1 Millionen pages `2^20` und ca. 4 MB PT pro Prozess. Für die meisten Programme sind 4 MB PT viel zu viel.
        1. *Large tables* beim x86: 2 MB und 4 MB pages
        1. zweistufige PT
    1. Spezielle Caches TLB (Translation Lookaside Buffer) machen die PT-Übersetzung in Hardware mit 8 bis 256 Einträgen. 
    
## Prozesse Threads Interrupts, Vorlesung vom 30.03.2016

### Prozesse
* früher: 
    - Prozess = Job = Task
    - Speicher für Code und Daten
    - Rechte für Device-Zugriff
    - CPU für Ausführung von Code
* heute: 
    - komplexer durch Multiprogramming/-threading/-tasking
    - mehrere CPUs führen den Code gleichezeitig aus
    
### Threads
1. Abstraktion über die Anzahl der verfügbaren CPUs
1. Ein Thread verkörpert eine "virtuelle" CPU, die den Code ausführt
1. sind die ausführbaren Einheiten
1. **Exkurs** heute; Prozess = 
    - Adressraum ==> Zuordnung von Speicher nach Bedarf
    - Rechte ==> Zugriff auf "virtuelle Devices" über Handles (z.B. Filesystem)
    - Threads ==> Zuordnung von CPUs nach Bedarf = *Scheduling*
1. Ein Prozess hat mindestens einen - oft aber mehrere - Threads
1. alle Threads eines Prozesses teilen sich den gemeinsamen Adressraum (Speicher) und die gleichen Zugriffsrechte (Handles)
1. **Jeder Thread**
    1. hat seinen eigenen Registersatz, insbesondere Program Counter (PC) und Stack Pointer (SP),
    1. hat seinen eigenen Stack für lokale Variablen und Rücksprungadressen von Funktionen,
    1. und wird nach Bedarf und Verfügbarkeit eine CPU zugeordnet.

### Interrupts (Unterbrechnungen)
1. asynchrone/externe/HW-Interrupts
1. synchrone/interne/Exceptions/SW-Interrupts (z.B. Division durch 0, page fault)

#### HW-Interrupts

```
                      PIC (Programmable Interrupt Controller)   |
+---------+         +---------+                                 | MMIX
|         |---------|   IRQ   |                                 | +----+
|   CPU   |         |   IRM   |                                 | | rQ |-
|         |         +---------+                                 | | rK |-
+---------+          |||||||||      PIC                         | +----+
                           |_____+---------+                    |
                                 |   IRQ   |                    |
                                 |   IRM   | Interrupt Mask Register
                                 +---------+
                                  ||||||||| 
```

### Was passiert, wenn ein Interrupt auftritt?
1. externer Interrupt wird einer CPU zugewiesen. Interne Interrupts haben bereits eine CPU. 
1. CPU unterbricht Ausführung des laufenden Threads. 
1. CPU wechselt von User Mode in Kernel Mode.
1. Der Thread Kontext (PC, SP, ...) wird (teilweise) gesichert (context switch light). 
1. Der Interrupt Handler wird ausgewählt. Geschieht meist über eine Sprungtabelle (Interrupt Vektor)
1. Der Handler wird ausgeführt. 
1. Der Thread-Kontext wird wieder hergestellt.
1. CPU wechselt in User Mode.
1. Der Thread wird an der ursprünglichen Stelle weiter ausgeführt.

### Interrupt Priorities
1. Jeder Interrupt hat eine zugeordnete Priority.
1. Prio. des laufenden Threads in einem Register (CPU/PIC)
1. Interrupt mit höherer Prio. unterbrechen Interrupts mit niedrigerer Prio (wenn enabled).
1. (Bemerkung: Man kann Interrupt Priorities im Kernel auch für Synchronisation verwenden)

### Vorteile von Threads
1. Vermeiden von Wartezeiten auf IO
1. effizientere Nutzung der CPU
1. Echte Parallelität mit mehreren CPUs
1. Threadwechsel sind billiger als Prozesswechsel
1. Bessere Strukturierung von Programmen
1. Kommunikation zwischen Threads ist effizienter als zwischen Prozessen (über gemeinsame globale Datenstrukturen)

### Nachteile von Threads
1. Overhead Threadwechsel
1. Synchronisation ist komplex (data race)

### Zustände von Threads
Zustände von Threads (s. Abbildung)

1. running current: Thread hat CPU und läuft
1. blocked/waiting: Thread hat nichts zu tun und braucht keine CPU
1. ready: Thread könnte laufen, hat aber keine CPU
![Zustände von Threads](images/2016-03-30_thread-zustaende.jpg
)

Hinweise:

1. Wenn dem OS die CPU entzogen wird, stehen **alle** User Level Threads
1. Wenn ein User Level Thread warten muss, müssen alle warten. 

## Prozesse, Threads, Interrupts am Beispiel Windows, Vorlesung vom 06.04.2016

### Prozesse

Threads

1. teilen sich den Adressraum
1. teilen sich die Ressourcen (Handles wie Datei, Fenster, Font, Thread, Prozess, ...)
1. getrennter, eigener Kontext (Register, Stack)
1. eigener user mode stack
1. eigener kernel mode stack

### Adressraum bei Win32

```
                +--------+
    FF FF FF FF |        |    - gemeinsam für alle Prozesse
                | Kernel |    - ca. 2 GB
                | Space  |
    80 00 00 00 |        |    
                +--------+
    7F FF FF FF |        |    - für jeden einzelnen Prozess
                | User   |    - ca. 2 GB
                | Space  |
    00 00 00 00 |        |
                +--------+
```

### Adressraum sonst

1. Bei Win64
    - 0,5 TB Kernel
    - 8 TB User Space
1. Ab Win2000
    1. Jobs: Mehrere Prozesse mit
        - gemeinsamer Priorität
        - gemeinsamen Ressourcenlimits (Ruhezeit, Disk Quota)
    1. Fibers: Mehrere User-Level-Threads innerhalb eines Kernel-Threads

### Schnittstellen (API) Win32

1. Win32 API
1. Posix API
    1. Subset funktioniert unter Windows
    1. hilft teilweise beim Portieren
1. (OS2 API, IBM)

### Prozess-Erzeugung (Win32 API)

```
BOOL CreateProcess(
    ApplicationName (string, *optional*)  // eines von beiden muss!
    CommandLine (string, *optional*)      // eines von beiden muss!
    ProcessAttributes (*optional*)
    ThreadAttributes (*optional*)
    InheritHandles (bool)
    CreationFlags {console, windows, unicode, debug, detached, ...}
    EnvironmentPointer (*optional*)
    CurrentDirectoryPointer (*optional*)
    StartupInfoPointer
    ProcessInfoPointer (out, *optional*) {ProcessHandle, ThreadHandle, ProcessID, ThreadID}
)
```

erzeugt den Prozess und den primären Thread. Der primäre Thread kann dann weitere Threads erzeugen.

### Thread-Erzeugung (Win32 API)

```
HANDLE CreateThread(
    ThreadAttributes (*optional*)
    StackSize
    StartAddress
    ParameterPointer (*optional*)
    CreationFlags
    ThreadIDPointer (out, *optional*)
)
```

**StackSize**:

1. Größe des User Space
1. wird auf ganze Seiten gerundet
1. wenn `StackSize == 0`, dann wir die `default` StackSize (1 MB) verwendet. 
1. Der Stack (Adressraum) wird **"reserved"** (invalid) und **"commited"** (reserved SWAP). Mehr darüber im Kapitel Memory Management.
1. Man bekommt nicht gleich 1 MB RAM, sondern nur 1 Page (ein Eintrrag in der page table)
1. Die restlichen Einträge der page table werden nur *reserviert* und bei Bedarf angelegt, wenn der Stack wächst. 
1. der so reservierte Adressraum kann für nichts anderes mehr verwendet werden, als für diesen Stack!

### Was heit commited?

1. Das Betriebssystem verpflichtet sich dazu, das nötige RAM bei Bedarf auch bereit zu stellen. 
1. Das Betriebssystem hat ein Commit-Limit, welches größer sein muss, als der gemeldete Stack. 
1. Die gemeldete Stackgröße wird dann vom Commit-Limit abgezogen. 

## Prozesse und Threads in Unix (Unix/Linux/Posix), Vorlesung vom 13.04.2016

### Wie erzeugt man in Unix einen Prozess?
1. `int fork();` keine Parameter
1. Rückgabewert
    * `< 0` Fehler
    * `= 0` Kind Prozess
    * `> 0` ProzessID des Kind-Prozesses im Eltern Prozess.
1. erzeugt ein Duplikat des laufenden Prozesses
    * ==> Sehr einfaches Konzept
    * ==> **Aber:**
        - Mutexe?
        - Handles?
        - Offene Dateien?
        - Threads? Nur der Thread, der fork() aufruft wird gestartet.
    * Es wird (wurde) **alles** kopiert!
    * Semantik ist nicht so ganz klar.
    * Speicher?
        1. heutzutage mit PageTables und "Copy on Write"
        1. Kopie der PageTables
        1. Sämtliche Einträge der PageTable werden *read only* gesetzt.
        1. "Leben im gleichen Speicher, solange nicht in diesen geschrieben wird."
        1. Beim ersten Schreibzugriff auf eine Seite bekommt man einen PageFault. Daraufhin bekommt jeder der Prozesse seine eigene Kopie.
1. Bootvorgang:
    * ProzessID 0 wird zum swap Prozess
        * mit fork() wird ProzessID 1 (init) erzeugt
            * init erzeugt alle weiteren Prozesse
                * getty
                    * bash
                        * gcc
                    * exit
                * weiterer Prozess
                * ...

#### Im Interrupt Handler
Beispiel: Eine einfache Shell

```
int main() {

char buffer[200];
int pid, status;

while(1) {
    fputs(">", stdout);
    fgets(buffer, 200, stdin); // lese input zeile in Konsole
    pid = fork(); // Prozess teilt sich auf: zwei bash

    if(pid < 0) {
        fputs("Error\n",stdout);
        return 1;

    } else if(pid == 0) { // "hier geht der Kind-Prozesses rein"
        buffer[strlen(buffer)-1] = 0; // Entfern \n am Zeilenende
        /*
         * Startet ein neues Programm, indem der gesamte Adressraum überschrieben wird.
         * Genauer: Die PageTables
         */
        status = execlp(buffer, "", 0);
        return status;
    } else { // "hier geht der Eltern-Prozesses rein"
        wait(&status); // wartet auf das Ende des Kind-Prozesses und erhält dessen Exit-Status.
    }
}
```

## Terminierung

1. `exit(status) // system call, geht an den Eltern-Prozess` ==> Files werden geschlossen Handles freigegeben.
1. Was passiert, wenn der Elternprozess nicht auf den Kindprozess wartet?
    * ==> Die Datenstruktur für den Prozess wird aufgehoben solange bis der Eltern-Prozess den Exitstatus bekommt.
    * ==> Der Kind-Prozess ist ein Zombie
    * Ausnahmen:
        - Der Eltern-Prozess kann das SIGCHLD Signal auch ignorieren oder abfangen.
        - Wenn der Eltern-Prozess vor dem Kind-Prozess temriniert, dann wird der Kind-Prozess (orphan, Waisenkind) vom init-Prozess adoptiert.

## Prozess-Zustände

![Prozess-Zustände in Unix](images/2016-04-13_prozesszustaende.png)
![Prozess-Zustände in Unix](images/2016-04-13_prozesszustaende.jpg)


## Scheduling, Vorlesung vom 20.04.2016

### Was ist Scheduling? 

Die Zuordnung von CPUs zu Threads. 

> Auf Servern eine wesentliche Aufgabe. Auf PCs ist die CPU hauptsächlich mit Warten beschäftigt. Auf Mobilgeräten ist es aufgrund der Akkulaufzeit relevant. Deswegen gibt es bereits viele Algorithmen für das Scheduling. 

### Was muss man beachten? Ziele? 
1. **Effizienz**: Jede CPU sollte soweit wie möglich genutzt werden.
1. **Fairness**: Jeder Prozess/Thread bekommt einen fairen Anteil an der CPU-Zeit. Insbesondere: kein Thread "verhungert" (starvation). 
1. **Responsiveness**: Die Wartezeit auf die CPU sollte nicht zu groß sein. 
1. **Realtime**: Die Antwortzeit auf Events hat eine obere Schranke. 

```
                                                   |
-----|--------------------|----------------|-------|----> Zeit
   Event      Warten   Scheduled        Antwort    |
                         CPU                    Schranke
```
### Methoden des Scheduling
1. **preemptive**: Threads werden nach einer vorher vorgegebenen Zeit (time slice, Quantum) abgebrochen und neu ge-scheduled.
1. **non-preemptive**: Jeder Thread darf so lange laufen, wie er etwas zu tun hat. 

### Verfahren 1: Round Robin Scheduling
Das Round Robin Scheduling (~ "im Kreis herum") ist ein wichtiges Basisverfahren. Dabei sind die Threads sind in einer zirkulären Liste (Queue) und kommen der Reihe nach dran. 

1. Der vorderste (laut Zeiger) bekommt die CPU für eine vorgegebene Zeit. 
1. Wenn die Zeit aufgebraucht ist, wird er wieder hinten eingereiht. 
1. Threads die Blockieren (warten müssen) werden aus der Queue entfernt. 
1. Threads, die aus dem Wartezustand wieder in den Ready-Zustand kommen, werden wieder in die Queue eingestellt. 
    - hinten in der Schlange ==> Muss noch eine Zeit lang warten.
    - vorne in der Schlange ==> Kommt gleich dran. 

Die entscheidende Frage bei diesem Verfahren ist, wie man das Quantum (Time Slice) wählt. 

1. nicht zu groß sonst leidet die Responsiveness)
1. nicht zu klein wegen des Kontext Switch Overheads

### Verfahren 2: Priority Based Scheduling
1. Jeder Prozess bzw. Thread hat eine Priorität. 
1. Der Thread mit der höchsten Priorität bekommt die CPU. 
1. Bei Threads mit gleicher Priorität wird meist das Round Robin Verfahren verwendet. 
1. Wenn ein Thread mit höherer Priorität nach *ready* wechselt, wird der Thread mit niedrigerer Priorität unterbrochen (preemption). 

Hier stellt sich die Frage, wie man die Prioritäten wählt?

1. Das Programm selbst legt die Priorität fest. 
1. Der Benutzer/Administrator legt die Priorität fest. 
1. Automatisch abhängig von der Prozessart (Foreground, Background, Interaktiv, Batchprocess)
1. Dynamische Anpassung der Priorität
    * normale Priorität/Base Priority
    * aktuelle/current Priority 
    * Scheduling erolgt aufgrund der current priority
    
### Beispiel
```
     CPU        CPU        CPU         CPU-Bound
----#####----->#####----->#####--->

   CPU    CPU    CPU    CPU    CPU     I/O-Bound
----#----->#----->#----->#----->#->   
```

Ziel: I/O-Bound-Prozesse bekommen eine höhere Priorität als CPU-Bound, weil die I/O nicht wesentlich von der CPU abhängig ist, sondern von der Lese- bzw. Schreibgeschwindigkeit der Disk. 

1. Interaktive Prozesse (z.B. Editor, Input, ...) bekommen eine höhere Aktivität. 
1. Foreground Prozesse bekommen eine höhere Priorität.
1. Prozesse, die schon sehr lange ready sind bekommen eine höhere Priorität. 

### Wann passiert das Scheduling?
1. Wenn ein Thread blockiert. 
1. Wenn das Time Slice aufgebraucht ist. 
1. Wenn die Blockierung eines Threads endet. 
1. Wenn ein neuer Thread startet.
1. Wenn ein Thread terminiert. 
1. Wenn sich die dynamischen Prioritäten ändern. 

### Wie passiert das Scheduling?
1. Scheduler läuft als Interrupthandler mit niedriger Priorität (bei Windows Level 2). 
1. Den Scheduler "aufrufen" heißt den entsprechenden Interrupt auslösen. 
1. Mehrfache "Aufrufe" ergeben nur einen Interrupt für den Scheduler.
1. Der Scheduler läuft nicht sofort, sondern erst, wenn die höheren Interrupts fertig sind. 
1. Kernel Threads können die Unterbrechnung durch den Scheduler verhindern, indem sie auf einem höheren Interrupt-Level laufen. 

### Wer ruft den Scheduler auf?
1. in einem system call (blockiert)
1. in einem I/O Interrupt (unblock)
1. im Clock Interrupt (preempt)

## Scheduling in Windows, Vorlesung vom 27.04.2016

1. Dispatcher läuft im Interrupt-Request-Level 2 (IRQ-Level2) 
    1. mit Prioritäten, 32 Level
    1. 32 Ready-Queues (Threads werden mittels round robin aktiviert. 
1. Wann läuft der Dispatcher?
        1. Wenn ein wartender Thread nach I/O completion ready wird. Windows ist [full preemptive](https://de.wikipedia.org/wiki/Multitasking#Pr.C3.A4emptives_Multitasking)
        1. Wenn ein Thread einen Event signalisiert. 
        1. Wenn das Quantum aufgebraucht ist. (clock interrupt ==> jetzt Zeitscheibe (preemption) zu Ende)
        1. Thread wartet auf IO (blockiert)
        1. Timed Wait wird fertig
1. Bit mask 32 Bit, 1 Bit pro Queue, "ready summary" zeigt an, welche Queues leer/voll sind. 
1. Bit mask 32 Bit, 1 Bit pro CPU, "idle summary" zeigt an, welche CPU idle/busy ist. 
1. Ablauf: Lock setzen, Zugriff bekommen, auf Queues arbeiten, freigeben
1. Modifikationen
    1. Prioritäten
        1. Klasse (pro Prozess) `SetPriorityClass`
            1. Idle -> 4
            1. BelowNormal --> 6
            1. Normale --> 8
            1. AboveNormal --> 10
            1. High--> 13
            1. Realtime --> 24
            1. 0 ist der Idle Thread
            1. 1 - 15 werden für User-Programme verwendet
            1. 16 - 31 sind Realtime-Threads
        1. Level (Thread) `SetThreadPriority`
            1. Idle --> 1 oder 16  (minimum)
            1. Lowest --> -2
            1. BelowNormal --> -1
            1. Normal -->  +/- 0
            1. AboveNormal --> +1
            1. Highest -->  +2
            1. Critical --> 15 oder 31 (maximal)
            1. ==>  Jeder Thread hat dadurch eine Base Priority
            1. ==> Die Base Priority wird dynamisch modifiziert --> **CurrentPriority; Wird für das Scheduling verwendet.**
            1. **keine** dynmaische Anpassung für Realtime-Threads
        1. Affinity
            1. Zuordnung von Threads zu CPU
            1. Jeder Thread hat einen idealen Prozessor ==> Thread läuft meist auf dem gleichen Prozessor (Vorteil: Daten bereits im Cache)
            1. hard affinity
                1. Jedem Thread werden fest CPUs zugeordnet
                1. Thread darf nur auf diesen CPUs laufen
            1. soft affinity
                1. Jedem Thread werden bevorzugte CPUs zugeordnet
                1. Thread sollte, wenn möglich, auf diesen CPUs laufen 
    2. Quantum
        1. 180 ms für Windows Server => längere Zeit ==> weniger context switches
        1. 30 ms für Work Station 
        1. 90 ms für den foreground Prozess bei Work Station
1. Dynamische Prioritätsanpassung
    1. `base priority <= current priority <= 15`
    1. Threads, die aufwachen, nachdem sie auf **I/O** gewartet haben, bekommen einen *priority boost* für ein Quantum. Der *boost* hängt vom Device ab:
        1. Disk, CD-Rom, ... --> +1
        1. Serielle IO, Netzwerk --> +2
        1. Keyboard, Mouse --> +6
        1. Sound --> +8
    1. Threads, die auf eine Windows-Message (User input) warten und ready werden, bekommen boost von +2 für ein Quantum.
    1. foreground Prozesse die ready werden, nachdem sie auf Mutex/Semaphore gewartet haben, bekommen auch +2 für ein Quantum.  
1. Anti Starvation boost (Starvation = verhungern; wenn ein Thread ready ist, aber nie die CPU bekommt)
    1. Einmal pro Sekunde wird geprüft, ob es einen Thread gibt, der schon mindestens 6 Sekunden ready ist. 
    1. Wenn ja, bekommt er Priorität 15 und dreifaches Quantum
    1. Der boost bleibt nur bis zur ersten Unterbrechung erhalten
1. [Priority Inversion](https://de.wikipedia.org/wiki/Priorit%C3%A4tsinversion) (???)
1. Multimedia Class Scheduler Service
    1. Applikation kann sich mit dem Service registrieren und wird dann speziell gescheduled
        1. high --> 23 - 26 (professionelle Multimedia-Software)
        1. Audio --> 16 - 22
        1. Normal --> 8 - 15
        1. Exhausted --> 1 - 7
1. Wie wählt Windows den nächsten Thread aus, wenn eine CPU frei wird?
    1. primary candidate: der erste Thread in der ready queue mit der höchsten Priorität. Er bekommt die CPU, wenn 
        1. er zuletzt auf dieser CPU lief, oder
        1. diese CPU seine bevorzugte CPU ist, oder
        1. er schon ca. 45 ms ready ist, oder
        1. die Priorität >= 24 ist.
    1. sonst: Suche nach anderem Thread, der diese Kriterien erfüllt. 
    
