# Acceptance Criteria: debugview

**Tool**: Sysinternals DebugView CLI (DbgViewCli)
**Repository**: https://github.com/nicksysinern/dbgview (internal)
**Purpose**: Validate correct usage of DbgView CLI skill for capturing Windows debug output

---

## 1. Bounded Execution Patterns

### 1.1 Duration Limit

#### ✅ CORRECT: Bounded capture with duration

```bash
dbgviewcli --no-banner --duration 30
```

#### ❌ INCORRECT: Unbounded capture (will run forever)

```bash
dbgviewcli
```

### 1.2 Line Limit

#### ✅ CORRECT: Stop after N lines

```bash
dbgviewcli --no-banner --max-lines 1000
```

#### ❌ INCORRECT: No limit for automated use

```bash
dbgviewcli --no-banner
# Missing --duration, --max-lines, or --wait-for
```

### 1.3 Pattern-Based Exit

#### ✅ CORRECT: Exit when pattern matches

```bash
dbgviewcli --no-banner --wait-for "*FATAL*" --duration 120
```

#### ❌ INCORRECT: wait-for without a safety timeout

```bash
dbgviewcli --no-banner --wait-for "*FATAL*"
# If pattern never matches, runs forever
```

---

## 2. Privilege Awareness

### 2.1 Kernel Capture

#### ✅ CORRECT: Acknowledge admin requirement

```bash
# Run as Administrator for kernel capture
dbgviewcli --kernel --no-banner --duration 30
```

#### ❌ INCORRECT: Kernel capture without elevation note

```bash
dbgviewcli --kernel --duration 30
# Will fail silently or error without admin
```

### 2.2 Boot Logging

#### ✅ CORRECT: Admin-only boot operations

```bash
# Must be run as Administrator
dbgviewcli --boot-enable
dbgviewcli --boot-status
dbgviewcli --boot-disable
```

#### ❌ INCORRECT: Boot logging without elevation

```bash
dbgviewcli --boot-enable
# Will fail — registry access denied
```

---

## 3. Output Format Patterns

### 3.1 Machine-Readable Output

#### ✅ CORRECT: Clean output for parsing

```bash
dbgviewcli --no-banner --format csv --duration 10
dbgviewcli --no-banner --format xml --duration 10
```

#### ❌ INCORRECT: Text format with banner for programmatic use

```bash
dbgviewcli --format csv
# Banner text on stderr will confuse naive parsers
```

### 3.2 Status Check

#### ✅ CORRECT: Machine-readable status

```bash
dbgviewcli --status
# Output: running=true\npaused=false\nelevated=true
```

#### ❌ INCORRECT: Parsing process list to detect DbgView

```bash
tasklist | findstr dbgview
# Fragile — use --status instead
```

---

## 4. Filtering Patterns

### 4.1 Process-Specific Capture

#### ✅ CORRECT: PID or process name filter

```bash
dbgviewcli --no-banner --pid-filter 4567 --duration 30
dbgviewcli --no-banner --process-filter "myservice.exe" --duration 30
```

#### ❌ INCORRECT: Capture all then grep

```bash
dbgviewcli --no-banner --duration 30 | findstr "myservice"
# Inefficient — use built-in filters instead
```

### 4.2 Wildcard Include/Exclude

#### ✅ CORRECT: Semicolon-separated patterns

```bash
dbgviewcli --no-banner --filter "MyDriver*;Network*" --exclude "verbose*;trace*"
```

#### ❌ INCORRECT: Regex syntax (not supported)

```bash
dbgviewcli --filter "MyDriver|Network"
# Filters use wildcard glob, not regex
```

---

## 5. Resource Cleanup Patterns

### 5.1 Graceful Shutdown

#### ✅ CORRECT: Bounded execution ensures cleanup

```bash
dbgviewcli --no-banner --duration 30
# Ctrl+C also triggers graceful shutdown (driver unload, file close)
```

#### ❌ INCORRECT: Kill process without cleanup

```bash
taskkill /F /IM dbgviewcli.exe
# May leave kernel driver loaded or log file unclosed
```

### 5.2 Boot Logging Cleanup

#### ✅ CORRECT: Always disable boot logging when done

```bash
dbgviewcli --boot-enable
# ... reboot and capture ...
dbgviewcli --boot-disable
```

#### ❌ INCORRECT: Leave boot logging enabled indefinitely

```bash
dbgviewcli --boot-enable
# Forgot to disable — driver loads on every boot
```

---

## 6. Remote Monitoring Patterns

### 6.1 Remote Connection

#### ✅ CORRECT: Remote with bounded execution

```bash
dbgviewcli --connect SERVER01 --no-banner --duration 60
```

#### ❌ INCORRECT: Remote without bounds

```bash
dbgviewcli --connect SERVER01
# Unbounded remote capture
```

---

## 7. Logging Patterns

### 7.1 File Logging with Limits

#### ✅ CORRECT: Bounded log file

```bash
dbgviewcli --no-banner --log debug.log --log-limit 100 --log-wrap --duration 3600
```

#### ❌ INCORRECT: Unbounded log file

```bash
dbgviewcli --log debug.log
# No size limit — can fill disk
```

---

## 8. Combined Safety Patterns

### 8.1 Defense-in-Depth Bounds

#### ✅ CORRECT: Multiple bounds for safety

```bash
dbgviewcli --no-banner --duration 60 --max-lines 10000 --wait-for "*COMPLETE*"
# Whichever triggers first wins
```

#### ❌ INCORRECT: Single point of failure

```bash
dbgviewcli --no-banner --wait-for "*COMPLETE*"
# If pattern never matches, runs forever
```
