# Apache Kafka 4.x Native Windows 11 Setup & Patch Guide

This guide documents the mandatory script modifications, configuration changes, and CLI commands required to run modern Apache Kafka (Kafka 4.x in KRaft mode) natively on Windows 11 without WSL or Docker.

---

## 1. Prerequisites & Environment Setup

* **Java:** OpenJDK 17 or 21 above installed. Verify with `java -version`.
* **Installation Path:** Place Kafka in a short path without spaces (e.g., `C:\kafka_2.13-4.4.0`).
* **Shell:** Standard Command Prompt (`cmd.exe`) or PowerShell running as Administrator.

---

## 2. File Patches

### Patch 1: Fix WMIC Deprecation in `bin\windows\kafka-server-start.bat`
**Problem:** Windows 11 removes `wmic.exe` by default, triggering `'wmic' is not recognized as an internal or external command`.

**Fix:** Open `bin\windows\kafka-server-start.bat` and locate lines ~22–32:
```bat
IF ["%KAFKA_HEAP_OPTS%"] EQU [""] (
    rem detect OS architecture
    wmic os get osarchitecture | find /i "32-bit" >nul 2>&1
    IF NOT ERRORLEVEL 1 (
        rem 32-bit OS
        set KAFKA_HEAP_OPTS=-Xmx512M -Xms512M
    ) ELSE (
        rem 64-bit OS
        set KAFKA_HEAP_OPTS=-Xmx1G -Xms1G
    )
)
```

Replace the entire block with a direct 64-bit heap setting:
```bat
IF ["%KAFKA_HEAP_OPTS%"] EQU [""] (
    set KAFKA_HEAP_OPTS=-Xmx1G -Xms1G
)
```

### Patch 2: Fix WMIC Deprecation in `bin\windows\kafka-server-stop.bat`

**Problem:** kafka-server-stop.bat uses wmic to locate and terminate the JVM process, which fails silently or crashes on Windows 11.

**Fix:** Open bin\windows\kafka-server-stop.bat, find the wmic process where... line:
```bat
wmic process where (commandline like "%%kafka.Kafka%%" and not name="wmic.exe") delete
```

Replace it with the native PowerShell CIM command:
```bat
powershell -NoProfile -ExecutionPolicy Bypass -Command "Get-CimInstance Win32_Process | Where-Object { $_.CommandLine -like '*kafka.Kafka*' } \vert{} ForEach-Object { Stop-Process -Id$_.ProcessId -Force }"
```

### Patch 3: Silence Log4j 2 StatusLogger Error in `bin\windows\kafka-run-class.bat`

**Problem:** The command outputs ERROR Reconfiguration failed: No configuration found for '\<hash>' at 'null' in 'null' or throws an IllegalArgumentException caused by Windows quote escaping and uninitialized URI parsing in CLI utilities.

**Fix:** Open bin\windows\kafka-run-class.bat, locate KAFKA_LOG4J_OPTS (~lines 150–165), and set it to turn off the internal Log4j status diagnostic layer:
```bat
set KAFKA_LOG4J_OPTS=-Dlog4j2.StatusLogger.level=OFF
```

***(Note: Keep only the single flag -Dlog4j2.StatusLogger.level=OFF to prevent command-line parsing splits in the batch file).***

### Patch 4: Avoid Command-Line Overflow in `bin\windows\kafka-run-class.bat`

**Problem:** Startup fails with The input line is too long due to individual JAR concatenation exceeding Windows 8,191-character command limit.

**Fix:** In bin\windows\kafka-run-class.bat, locate the JAR loop (~lines 85–95):
```bat
for %%i in ("%BASE_DIR%\libs\*") do (
    call :concat "%%i"
)
```

Replace it with a single wildcard reference:

```bat
set CLASSPATH=%CLASSPATH%;%BASE_DIR%\libs\*
```

### Patch 5: Adjust config\server.properties for Windows File Locking

Problem: Windows file locks cause crashes (FileAlreadyExistsException / AccessDeniedException) during background segment rolls and deletions.

**Fix:** Open config\server.properties and add/modify these lines:

```bat
# Use forward slashes to prevent escape character parsing errors
log.dirs=C:/kafka_2.13-4.4.0/kraft-combined-logs

# Prevent Windows file-locking crashes during local testing
log.cleaner.enable=false
log.segment.delete.delay.ms=60000
```
---

## 3. Storage Formatting & Startup Sequence

### Navigate to your Kafka root directory in Command Prompt (cmd.exe):

```bat
cd C:\kafka_2.13-4.4.0
```

**Step 1:** Generate KRaft Cluster UUID
```bat
.\bin\windows\kafka-storage.bat random-uuid
```

***Output will return a clean 22-character string, e.g., Atg8UBTEQiSGnTj94RsBVQ.***

**Step 2:** Format Storage Directory (Single-Node KRaft)

In Kafka 4.x, KRaft requires --standalone when controller.quorum.voters is not set:

```bat
.\bin\windows\kafka-storage.bat format --standalone -t <YOUR_UUID> -c .\config\server.properties
```

**Step 3:** Launch the Kafka Server

```bat
.\bin\windows\kafka-server-start.bat .\config\server.properties
```

***Startup verification: Look for [KafkaServer id=1] started or [KafkaRaftServer] Kafka Server started.***

---

## 4. Daily Verification & Topic Management

### Open a separate Command Prompt window `(C:\kafka_2.13-4.4.0)`:

**Create a Topic**
```bat
.\bin\windows\kafka-topics.bat --create --topic win11-topic --bootstrap-server localhost:9092
```

**List Topics**
```bat
.\bin\windows\kafka-topics.bat --bootstrap-server localhost:9092 --list
```

**Send Messages (Producer)**
```bat
.\bin\windows\kafka-console-producer.bat --topic win11-topic --bootstrap-server localhost:9092
```

**Consume Messages (Consumer)**
```bat
.\bin\windows\kafka-console-consumer.bat --topic win11-topic --from-beginning --bootstrap-server localhost:9092
```

**Delete a Topic**
```bat
.\bin\windows\kafka-topics.bat --bootstrap-server localhost:9092 --delete --topic win11-topic
```

**Stop the Kafka Server**
```bat
.\bin\windows\kafka-server-stop.bat
```

---

**Created by:** Samarth Gautam   
*© 2026 Samarth Gautam. All rights reserved.*