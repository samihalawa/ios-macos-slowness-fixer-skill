---
name: ios-macos-slowness-fixer-skill
description: "Diagnose and fix slowness, freezes, crashes, overheating, and battery drain on any Apple device: iPhone, iPad, or Mac. Use when someone says a phone, tablet, or Mac is slow, laggy, buggy, crashy, freezing, restarting, hot, or draining. By default it detects where the agent is running (local macOS shell, sandbox, or cloud), which tools are available (Bash, Desktop Commander MCP, computer-use, libimobiledevice, pymobiledevice3, devicectl), lists connected iOS/iPadOS devices, collects evidence from crash logs, Jetsam memory kills, hang tracer analytics, sysdiagnose power logs, and live system logs, finds the real cause, applies safe fixes, and writes an owner-facing fix plan in the owner's language with menu paths in the device's UI language."
---

# iOS / iPadOS / macOS Slowness Fixer

Find the real reason an Apple device is slow or crashy from the device's own records, fix what an agent can safely fix, and give the owner a short, convincing plan for the rest.

Evidence beats intuition. Every claim in the final report must trace to a log, counter, or table read in this run.

## 0. Where am I running, and what can I reach?

Run this first, every time. The answers decide the route.

```bash
uname -s; sw_vers 2>/dev/null | tr '\n' ' '; echo
echo "user=$(id -un) home=$HOME"
# Sandbox / cloud signals
[ -n "$CLAUDE_CODE_REMOTE" ] && echo "claude-remote"; [ -f /.dockerenv ] && echo "container"
[ -S /var/run/usbmuxd ] && echo "usbmuxd socket present" || echo "no usbmuxd socket"
ls ~/Library/Logs/DiagnosticReports >/dev/null 2>&1 && echo "can read user diagnostics" || echo "diagnostics blocked"
# Device tools
for t in idevice_id ideviceinfo idevicediagnostics idevicecrashreport idevicesyslog ideviceinstaller idevicepair idevicedevmodectl pymobiledevice3 xcrun sqlite3 python3; do printf '%s=%s\n' "$t" "$(command -v $t || echo MISSING)"; done
[ -x ~/.local/bin/pymobiledevice3 ] && echo "pymobiledevice3 at ~/.local/bin"
```

Decide:

| Situation | Route |
|---|---|
| Local macOS shell, usbmuxd socket present | Run everything below directly with Bash. |
| macOS but the sandbox blocks USB, `/var/run/usbmuxd`, `~/Library/Logs`, or `log show` | Rerun the blocked command with the sandbox disabled (for example `dangerouslyDisableSandbox: true` in Claude Code), or run it through Desktop Commander MCP `start_process`, which executes on the real host. |
| Linux container, cloud session, or no host access | Look for host‑reaching tools in the tool list: Desktop Commander MCP (`start_process`, `read_file`), MacOS‑MCP `Shell`, "Control your Mac" `osascript`, computer‑use. Load them with ToolSearch and run the same commands through them. If none exists, say exactly which host tool is missing; never pretend to have run a check. |
| Only screenshots / computer-use available | Use them for on-screen proof (Settings pages, Battery, Storage), not as the primary data source. |

Missing CLIs on a Mac are installable and that counts as agent‑doable work:

```bash
brew install libimobiledevice ideviceinstaller
pipx install pymobiledevice3   # or: uv tool install pymobiledevice3
```

## 1. Inventory devices

```bash
xcrun devicectl list devices 2>&1 | grep -v simulated   # name, model, UDID, state
idevice_id -l                                            # USB-reachable UDIDs
for u in $(idevice_id -l); do
  echo "== $u"; idevicepair -u $u validate
  for k in DeviceName ProductType ProductVersion BuildVersion; do echo "$k: $(ideviceinfo -u $u -k $k)"; done
  ideviceinfo -u $u -q com.apple.international 2>/dev/null | grep -E '^(Language|Locale):'   # UI language for menu paths
  idevicedevmodectl -u $u list 2>/dev/null | tail -1                                         # Developer Mode state
done
```

- Pick the device the user means. If several are connected, prefer the one named or the one physically on USB, and say which one was used.
- `unavailable` in devicectl or a failed `validate` means not trusted: ask the owner to unlock it, tap **Trust**, enter the passcode, then rerun. If the prompt never appears: Settings → General → Transfer or Reset → Reset → **Reset Location & Privacy**, then reconnect.
- Record the device's UI language. The owner report must use menu labels in that language.

Set `U=<udid>` and `S=<scratchpad>/<device>` for the rest. Use the session scratchpad, never the project tree.

## 2. iPhone / iPad: fast checks (no Developer Mode needed)

### 2.1 Battery, heat, charging

```bash
idevicediagnostics -u $U ioregentry AppleSmartBattery > $S/batt.plist
python3 -I - $S/batt.plist <<'E'
import plistlib,sys
d=plistlib.load(open(sys.argv[1],'rb')); d=d.get('IORegistry',d); b=d.get('BatteryData',{})
lt=b.get('LifetimeData',{})
print('cycles',d.get('CycleCount'),'design',d.get('DesignCapacity'),'nominal',d.get('NominalChargeCapacity'),
      'health%',round(100*d.get('NominalChargeCapacity',0)/max(1,d.get('DesignCapacity',1)),1))
print('tempC',d.get('Temperature',0)/100,'charging',d.get('IsCharging'),'adapter',d.get('AdapterDetails',{}).get('Description'),d.get('AdapterDetails',{}).get('Watts'),'W')
print('daily SoC min/max',b.get('DailyMinSoc'),b.get('DailyMaxSoc'),'WeightedRa',b.get('WeightedRa'))
print('Ra table',[b.get(f'Ra{i:02d}') for i in range(15)])
print('lifetime maxTemp',lt.get('MaximumTemperature',0)/10,'minPackV',lt.get('MinimumPackVoltage'))
E
ideviceinfo -u $U -q com.apple.mobile.battery
```

Read it as: health below 80% is a service candidate; the internal resistance (`Ra`) rising sharply at the end of the table means voltage sag at low charge; `DailyMinSoc`/`DailyMaxSoc` show how the owner actually uses it. **Never conclude "the battery is slowing the phone" from these alone.** Only the sysdiagnose power log (section 3) proves throttling.

### 2.2 Storage and apps

```bash
ideviceinfo -u $U -q com.apple.disk_usage | grep -E 'TotalDataCapacity|TotalDataAvailable|AmountDataAvailable'
ideviceinstaller -u $U list --user -a CFBundleDisplayName -a DynamicDiskUsage -a StaticDiskUsage > $S/apps.csv
python3 -I - $S/apps.csv <<'E'
import csv,sys
rows=list(csv.reader(open(sys.argv[1]),skipinitialspace=True))[1:]
out=sorted(((int(r[1])/1e9,int(r[2])/1e9,r[0]) for r in rows if len(r)>2 and r[1].isdigit()),reverse=True)
print('apps',len(out),'data GB',round(sum(o[0] for o in out),1))
for d,s,n in out[:15]: print(f'{n:24s} data {d:5.1f} GB  app {s:4.1f} GB')
E
```

Lots of free space means storage is not the problem; say so and move on.

### 2.3 Crash, memory-kill, and hang records

```bash
mkdir -p $S/crashes && idevicecrashreport -u $U -e -k $S/crashes >/dev/null 2>&1
cd $S/crashes
# Tally by report type and by name
for f in $(find . -name '*.ips'); do head -1 "$f" | grep -oE '"bug_type":"[0-9]+"'; done | sort | uniq -c | sort -rn
find . -type f | sed -E 's#.*/##; s/[-_][0-9]{4}-[0-9]{2}-[0-9]{2}-[0-9]{6}.*//' | sort | uniq -c | sort -rn | head -30
find . -name 'JetsamEvent*' | sed -E 's/.*JetsamEvent-([0-9-]{10}).*/\1/' | sort | uniq -c    # memory kills per day
```

Jetsam (memory-kill) events: who was biggest, how big, what got killed.

```bash
for f in $(find . -name 'JetsamEvent*'); do python3 -I - "$f" <<'E'
import json,sys
d=json.loads(open(sys.argv[1]).read().split('\n',1)[1]); pg=d.get('memoryStatus',{}).get('pageSize',16384)
L=d.get('largestProcess'); sz=[round(p.get('rpages',0)*pg/1e6) for p in d.get('processes',[]) if p.get('name')==L]
print(L, sz[0] if sz else '?', 'MB')
E
done | awk '{c[$1]++; s[$1]=s[$1]" "$2} END{for(k in c) print c[k], k":"s[k]}' | sort -rn
```

Note: a process name with spaces (for example "Notion Calendar") breaks naive `awk` splitting. Print names last or quote them.

App crash reports (`bug_type` 309): print the exception, the termination code, and the top frames.

```bash
for f in $(grep -l '"bug_type":"309"' $(find . -name '*.ips')); do echo "## $f"; python3 -I - "$f" <<'E'
import json,sys
d=json.loads(open(sys.argv[1]).read().split('\n',1)[1])
print(' ex:',d.get('exception')); print(' term:',str(d.get('termination'))[:300])
ft=d.get('faultingThread'); th=d.get('threads',[]); im=d.get('usedImages',[])
if ft is not None and ft<len(th):
  print(' frames:',[(im[x['imageIndex']].get('name') if x.get('imageIndex') is not None and x['imageIndex']<len(im) else '?')+':'+str(x.get('symbol','')) for x in th[ft].get('frames',[])[:6]])
E
done
```

Termination codes worth naming in the report:

| Code | Meaning | Typical takeaway |
|---|---|---|
| `0x8BADF00D` | Watchdog: app took too long to launch, resume, or quit | The app is too heavy for this device right now (huge local data, memory pressure). |
| `0xBAADCA11` | App received a VoIP push and never reported a call to CallKit | The app abuses fake "incoming call" pushes to wake itself in the background. |
| `0xDEAD10CC` | App held a file or database lock while being suspended | Background misbehavior; usually the app's bug, not the owner's. |
| `0xC00010FF` | Thermal kill | Real overheating. |
| Jetsam reasons `vm-compressor-space-shortage`, `vm-pageshortage`, `proc-thrashing`, `per-process-limit` | Memory exhaustion | The device is out of RAM. |

Other report types to check:

- `SystemMemoryReset-*.ips` (bug_type 301): the whole system restarted because memory ran out. Read `eventReason` and `memoryStatus.largestProcess`. The file is JSON after the first line; if `json.loads` fails, grep `"eventReason"` and `"largestProcess"`.
- `*.diskwrites_resource-*.ips`: an app wrote more than the limit (for example 1 GB in N seconds). Grep `Writes:` and `Duration:`.
- `*.cpu_resource-*.ips`: CPU abuse.
- `409` hang stackshots, `ForceResetTailspins/` (the owner hard-reset a frozen device), and `Panics/` or `panic-full` files.

### 2.4 Hang tracer: how often the UI freezes, per app

The daily `Analytics-*.ips.ca` files (including those in `Retired/`) hold Apple's own freeze counts.

```bash
python3 -I - $(find . -name 'Analytics*' | sort) <<'E'
import json,sys,collections
for fn in sys.argv[1:]:
  H=collections.Counter(); T=collections.Counter(); U=collections.Counter()
  for line in open(fn,errors='ignore'):
    if not line.startswith('{'): continue
    try: d=json.loads(line)
    except: continue
    n=d.get('name',''); m=d.get('message',{})
    if n.startswith('HangTracerHangsByApp'):
      c=m.get('Count',1); H[m.get('bundleid')]+=c; T[m.get('hangtype')]+=c
    if n=='embeddedOSAppUsageSessions' and m.get('position')=='foreground': U[m.get('bundleID')]+=m.get('sum_of_duration',0)
  print('=====',fn.split('/')[-1][:32],'hangs',sum(H.values())); print(' types',dict(T)); print(' by app',H.most_common(12))
  if U: print(' foreground seconds',U.most_common(8))
E
```

A rising total across days is the strongest proof the problem is getting worse. `com.apple.springboard` freezing means the whole home screen and system UI stall. The `bucketed_duration` units are undocumented; report counts, not seconds.

### 2.5 Live system log (30–60 s)

```bash
idevicesyslog -u $U --no-colors > $S/live.log 2>&1 & P=$!; perl -e 'sleep 45'; kill $P
awk '{print $4}' $S/live.log | sed -E 's/\[[0-9]+\].*//;s/\(.*//' | sort | uniq -c | sort -rn | head -15
grep "killed by jetsam" $S/live.log | sed -E 's/.*Process ([^ ]+) \[[0-9]+\] killed by jetsam reason ([a-z-]+).*/\1 \2/' | sort | uniq -c | sort -rn
grep -E "installcoordinationd|appstored" $S/live.log | grep -oE "com\.[A-Za-z0-9-]+\.[A-Za-z0-9.-]+" | grep -v '^com.apple' | sort -u   # background App Store installs
grep -oE "thermalLevel[\":]+[0-9]+|New state : [0-9]+" $S/live.log | sort | uniq -c
```

**Never capture the live log while `idevicecrashreport` or any other file transfer runs.** `afcd` floods the log, and the extra load inflates kills and thermal readings. Capture a clean log separately before drawing conclusions from it.

### 2.6 Process snapshot

```bash
PYMOBILEDEVICE3_UDID=$U pymobiledevice3 processes ps > $S/ps.json   # works with Developer Mode off
python3 -I -c 'import json,sys;d=json.load(open(sys.argv[1]));v=list(d.values());print(len(v),"processes");print(sorted({x.get("ProcessName") for x in v if isinstance(x,dict) and x.get("ProcessName","")[:1].isupper()}))' $S/ps.json
PYMOBILEDEVICE3_UDID=$U pymobiledevice3 diagnostics battery single | head -40
```

`pymobiledevice3` takes the device through `--udid` or `PYMOBILEDEVICE3_UDID`. If an option is rejected, use the environment variable.

### 2.7 More USB sources (no Developer Mode)

```bash
idevicediagnostics -u $U diagnostics GasGauge    # battery gauge summary
idevicediagnostics -u $U diagnostics NAND        # storage controller info (when exposed)
idevicediagnostics -u $U diagnostics WiFi
ideviceinfo -u $U -k ProductVersion; ideviceinfo -u $U -q com.apple.mobile.software_behavior 2>/dev/null
```

MobileGestalt queries (`pymobiledevice3 diagnostics mg`, `idevicediagnostics mobilegestalt`) are deprecated on iOS 17.4 and later. Use ioreg and battery entries instead.

**Unified log straight from the device.** This holds more history than a sysdiagnose. It needs `sudo`, a trusted device and the passcode, so check `sudo -n true` first. If there's no passwordless sudo, ask the user to run the one command, or use the logarchive inside the sysdiagnose (section 3.4).

```bash
sudo log collect --device-udid $U --last 6h --output $S/device.logarchive
log show --archive $S/device.logarchive --info --predicate 'eventMessage CONTAINS "killed by jetsam"' --style compact | sed -E 's/.*Process ([^ ]+) \[[0-9]+\] killed by jetsam reason ([a-z-]+).*/\1 \2/' | sort | uniq -c | sort -rn | head -20
```

`log show` hides info and debug entries unless you pass `--info --debug`, so an empty result without them proves nothing.

### 2.8 Was there a recent OS update? (rule this out before blaming anything)

After a major iOS/iPadOS/macOS update, Spotlight re‑indexes, Photos re‑analyzes, Apple Intelligence downloads models, and the App Store updates apps. That can make a device slow and hot for **2–3 days, up to a week**, and Apple documents it as expected.

Signs:
- Settings shows "Optimizing Search and Siri", or Battery shows "Ongoing iOS Update / Device Setup".
- The live log is dominated by `corespotlightd`/`mediaanalysisd`/`photoanalysisd`/`installcoordinationd`.
- The OS build is new.

If the update is less than 72 hours old, the first fix is: leave the device locked, on Wi‑Fi and charging overnight, then re‑measure. Point releases often fix known storage or System Data bugs, so check whether a newer point update exists before deeper surgery.

## 3. iPhone / iPad: deep evidence from sysdiagnose

A sysdiagnose contains the power log (days of per-app energy, screen time, memory, battery throttling, and thermal data), `vm_stat`, process lists, spindumps, and microstackshots.

**Route A (Developer Mode on, device unlocked):**

```bash
xcrun devicectl device sysdiagnose --device $U --destination $S/sysdiag --timeout 1500
```

It fails with `device was still locked` (ask the owner to unlock) or `Developer mode is not enabled` (use Route B).

**Route B (no Developer Mode; works everywhere):** start the collector, then have the owner press **both volume buttons + the side button for about 1 second** until the device vibrates. They should not hold it long enough for the power-off or SOS screen.

```bash
PYMOBILEDEVICE3_UDID=$U pymobiledevice3 crash sysdiagnose --timeout 1200 $S/sysdiag/sysdiagnose.tar.gz
# progress check: an IN_PROGRESS_sysdiagnose_*.tmp should appear
PYMOBILEDEVICE3_UDID=$U pymobiledevice3 crash ls /DiagnosticLogs/sysdiagnose
```

Run the collector in the background. It takes about 2–10 minutes to build a 300–500 MB archive.

**Developer Mode is a security setting. Never enable it on the owner's device yourself**, even when asked. You may run `idevicedevmodectl -u $U reveal` so the toggle appears; the owner turns it on under Settings → Privacy & Security → Developer Mode, then restarts and confirms. If the toggle still doesn't show, don't loop: use Route B.

Extract into its own directory and open the power log by its exact filename. A glob passed to `sqlite3` creates an empty file literally named `*.PLSQL`.

```bash
mkdir -p $S/sd && tar -xzf $S/sysdiag/sysdiagnose.tar.gz -C $S/sd
R=$(ls -d $S/sd/*/ | head -1); cp "$R"logs/powerlogs/powerlog_*.PLSQL $S/pl.db; DB=$S/pl.db
sqlite3 $DB "select datetime(min(timestamp),'unixepoch','localtime'),datetime(max(timestamp),'unixepoch','localtime') from PLBatteryAgent_EventBackward_Battery"
```

### 3.1 Is the battery actually slowing the device?

```bash
sqlite3 -header $DB "select datetime(timestamp,'unixepoch','localtime') t, MitigationDisabled, MitigationState from PLBatteryAgent_EventForward_MitigationState order by timestamp desc limit 3"
sqlite3 $DB "select round(sum(PPPNotif_0*SampleTime)/sum(SampleTime)*100,2) lvl0, round(sum(PPPNotif_1*SampleTime)/sum(SampleTime)*100,2) lvl1, round(sum(PPPNotif_2*SampleTime)/sum(SampleTime)*100,2) lvl2, round(sum(PPPNotif_3*SampleTime)/sum(SampleTime)*100,2) lvl3, round(sum(SampleTime)/3600,1) hours from PLIOReportAgent_EventBackward_PPMStatsPeakPowerPressure"
sqlite3 $DB "select round(100.0*sum(Level<20)/count(*),1) pct_below20, round(100.0*sum(Level<40)/count(*),1) pct_below40, max(Temperature) maxT, round(avg(Temperature),1) avgT, min(Level) minLevel from PLBatteryAgent_EventBackward_Battery"
```

`MitigationState = 0` plus about 100% of time at peak-power-pressure level 0 means **the battery is not throttling performance**. Say so plainly, and correct any earlier claim that blamed the battery. The CPU power-reduction histogram is in `PLIOReportAgent_EventBackward_PPMStatsCPMSPowerReduction` (sum the `Sys100ms_*` columns, skipping `_Transitions`).

### 3.2 Who uses the screen, the background, the battery, and the memory

```bash
# Screen and background time per day (the aggregate table spans ~30 days, so always group by date)
sqlite3 -column $DB "select date(timestamp,'unixepoch','localtime') d, BundleID, round(sum(ScreenOnTime)/3600,1) screen_h, round(sum(BackgroundTime)/3600,1) bg_h, round(sum(BackgroundAudioPlayingTime)/60) bg_audio_min from PLAppTimeService_Aggregate_AppRunTime where timestamp > strftime('%s','now')-4*86400 group by d,BundleID order by d desc, screen_h desc limit 40"
# Energy share per app (what Settings > Battery shows), last 3 days
sqlite3 -column $DB "with e as (select n.Name name, sum(r.Energy) en from PLAccountingOperator_Aggregate_RootNodeEnergy r join PLAccountingOperator_EventNone_Nodes n on n.ID=r.NodeID where r.timestamp > strftime('%s','now')-3*86400 and n.Name like '%.%' group by n.Name) select name, round(100.0*en/(select sum(en) from e),1) pct from e order by en desc limit 15"
# Energy by hardware component (CPU, cellular radio BB*, display, Wi-Fi)
sqlite3 -column $DB "with e as (select n.Name name, sum(r.Energy) en from PLAccountingOperator_Aggregate_RootNodeEnergy r join PLAccountingOperator_EventNone_Nodes n on n.ID=r.RootNodeID where r.timestamp > strftime('%s','now')-3*86400 group by n.Name) select name, round(100.0*en/(select sum(en) from e),1) from e order by en desc limit 10"
# App memory: peak and while suspended in the background
sqlite3 -column -header $DB "select AppBundleId, round(max(PeakMemory)/1e6) peakMB, round(max(SuspendedMemory)/1e6) suspMB, round(avg(SuspendedMemory)/1e6) avgSuspMB from PLApplicationAgent_EventBackward_ApplicationMemory group by AppBundleId order by peakMB desc limit 15"
# CPU minutes, GB written, wakeups per app/daemon
sqlite3 -column -header $DB "select coalesce(nullif(BundleId,''),LaunchdName) who, round(sum(cpu_time)/60,1) cpu_min, round(sum(byteswritten)/1e9,2) GBw, round(sum(interrupt_wakeups)/1000) kwakes from PLCoalitionAgent_EventInterval_CoalitionInterval group by who order by cpu_min desc limit 15"
```

Signals to name:

- **High `BackgroundAudioPlayingTime` for an app that doesn't play audio** means it is faking audio to stay alive in the background.
- **High average `SuspendedMemory`** means the app hogs RAM even when not in use.
- **Many GB written by one app** heats the device and wears storage.
- **A large share of energy on `BB*` nodes** means the cellular radio; usually weak signal.
- **Huge `kernel_task` writes** usually mean swap.

### 3.3 Memory pressure right now

```bash
head -6 "$R/vm_stat.txt"
```

Page size is 16 KB. `free` × 16 KB is free memory. `cmprssor` is RAM spent holding compressed memory. `cmprssed` is how much data is stored there. Cumulative `swapins`/`swapouts` × 16 KB is how much was swapped since boot. Memory almost full, plus heavy swap, plus many Jetsam events: the root cause is memory, not the battery.

### 3.4 Query the sysdiagnose's own unified log (no sudo needed)

```bash
LA="$R/system_logs.logarchive"
log show --archive "$LA" --info --predicate 'eventMessage CONTAINS "killed by jetsam"' --style compact | sed -E 's/.*Process ([^ ]+) \[[0-9]+\] killed by jetsam reason ([a-z-]+).*/\1 \2/' | sort | uniq -c | sort -rn | head -20
log show --archive "$LA" --info --predicate 'process == "SpringBoard" AND messageType == fault' --style compact | tail -20
log show --archive "$LA" --info --predicate 'eventMessage CONTAINS[c] "thermal" AND eventMessage CONTAINS[c] "level"' --style compact | tail -20
log show --archive "$LA" --predicate 'process == "watchdogd" OR eventMessage CONTAINS "watchdog"' --style compact | tail -20
```

Other sysdiagnose files worth a glance: `ps.txt`, `ps_thread.txt`, `taskSummary.csv`, `jetsam_priority.txt` (what is resident and at what priority), `spindump-nosymbols.txt`, `microstackshots/`, `logs/Accessibility/`, `WiFi/`, and `apfs_stats.txt`.

Optional parsers when the user wants a full report: iLEAPP (`python ileapp.py -t tar -i sysdiagnose.tar.gz -o out/`, which also reads a logarchive converted with `log show --style json --archive X.logarchive > logarchive.json`), the EC‑DIGIT‑CSIRC `sysdiagnose` toolkit, and APOLLO powerlog modules. The SQL in 3.1–3.2 is usually enough.

### 3.5 With Developer Mode (optional extra)

iOS 17+ needs a tunnel for developer services (`sudo pymobiledevice3 remote tunneld`, then `--tunnel ''`). Then you can read live per-process CPU, memory and energy:

```bash
pymobiledevice3 developer dvt sysmon process monitor threshold 5 --tunnel ''
pymobiledevice3 developer dvt sysmon process single --tunnel ''
pymobiledevice3 developer dvt energy <PID...> --tunnel ''
```

## 4. Mac checks

Read-only first. On macOS, run on the host (see section 0).

```bash
sw_vers | tr '\n' ' '; sysctl -n machdep.cpu.brand_string hw.memsize hw.ncpu; uptime
memory_pressure | tail -1; sysctl vm.swapusage
vm_stat | grep -E 'free|stored in compressor|occupied by compressor|Swapins|Swapouts|Pageouts'
df -h /System/Volumes/Data | tail -1; ps -A | wc -l
ps -Ao pid,%cpu,%mem,rss,etime,comm -r | head -16
pmset -g therm; pmset -g batt | tail -1
system_profiler SPPowerDataType | grep -E 'Cycle Count|Condition|Maximum Capacity'
```

Read it as:
- Load average far above the core count means overloaded.
- `stored in compressor` × 16 KB vs `occupied by compressor` × 16 KB is the compression load.
- `Swapouts` × 16 KB since boot gives GB written to the SSD as swap.
- Swap close to its total means no headroom left.

Group memory and CPU by app family, since one app often has hundreds of helper processes:

```bash
ps -Ao rss=,%cpu=,command= > $S/mac_ps.txt
python3 -I - $S/mac_ps.txt <<'E'
import sys,re,collections
M=collections.Counter();C=collections.Counter();N=collections.Counter()
for l in open(sys.argv[1]):
  p=l.split(None,2)
  if len(p)<3: continue
  rss,cpu,cmd=int(p[0]),float(p[1]),p[2]; exe=cmd.split()[0]
  m=re.search(r"/Applications/([^/]+?)\.app",cmd)
  if m: k=m.group(1)
  elif "/.codex/" in cmd or exe.endswith("/codex") or exe=="codex": k="codex CLI/agents"
  elif re.search(r"(^|/)(node|npm|npx|bun|deno)$",exe): k="node/bun"
  elif "python" in exe.lower() or "Python.app" in cmd: k="python"
  elif re.search(r"chrom|playwright",cmd,re.I): k="chrome/automation (non-app)"
  elif re.search(r"claude",exe,re.I): k="claude CLI"
  else: k=exe.split("/")[-1][:30]
  M[k]+=rss;C[k]+=cpu;N[k]+=1
print("%-32s %6s %6s %5s"%("group","GB","CPU%","procs"))
for k,v in M.most_common(24): print("%-32s %6.1f %6.0f %5d"%(k,v/1048576,C[k],N[k]))
E
```

Crash, memory-kill, panic, and hitch records:

```bash
find ~/Library/Logs/DiagnosticReports /Library/Logs/DiagnosticReports -type f -mtime -7 2>/dev/null | sed -E 's#.*/##; s/[-_][0-9]{4}-[0-9]{2}-[0-9]{2}[-_][0-9]{6}.*//' | sort | uniq -c | sort -rn | head -25
# Kernel panic reason (the .contents.panic index carries panic_string even when the panic file is root-only)
LC_ALL=C grep -oE 'panic\(cpu[^"]{0,300}' /Library/Logs/DiagnosticReports/.contents.panic 2>/dev/null | head -2
# Mac Jetsam: largest processes
for f in $(find ~/Library/Logs/DiagnosticReports /Library/Logs/DiagnosticReports -type f -mtime -7 -name 'JetsamEvent*' 2>/dev/null); do python3 -I -c '
import json,sys;t=open(sys.argv[1]).read().split("\n",1);h=json.loads(t[0]);d=json.loads(t[1]);pg=d.get("memoryStatus",{}).get("pageSize",16384)
print(h.get("timestamp"),[(p.get("name"),round(p.get("rpages",0)*pg/1e9,1)) for p in sorted(d.get("processes",[]),key=lambda p:-p.get("rpages",0))[:5]])' "$f"; done
```

The panic string `userspace watchdog timeout: no successful checkins from WindowServer` means the screen system froze for about 2 minutes and macOS force-restarted. `skylight_hitch` reports are WindowServer stutters.

Leftover and duplicate background work (common on machines running AI agents):

```bash
# MCP servers / automation browsers: count, age, and who launched them
ps -Ao etime,command | grep '[c]hrome-devtools-mcp' | grep 'node ' | awk '{e=$1; print (e~/-/)?">1day":(length(e)>5?"hours":"<1h")}' | sort | uniq -c
for p in $(pgrep -f 'chrome-devtools-mcp|playwright|agent-browser' | head -100); do ps -o comm= -p $(ps -o ppid= -p $p | tr -d ' ') 2>/dev/null; done | sed -E 's#.*/##' | sort | uniq -c | sort -rn
ps -Ao pid,rss,etime,command | grep -iE '[c]odex|[c]laude' | sort -k2 -nr | head -5     # biggest agent processes
xcrun simctl list devices booted                                                        # forgotten simulators
for p in $(pgrep -f 'Virtualization.VirtualMachine'); do footprint -p $p 2>/dev/null | grep -m1 Footprint; done   # VMs
pmset -g assertions | grep -E 'PreventUserIdleSystemSleep|PreventSystemSleep|NoIdleSleep' | grep 'pid'           # who blocks sleep
ps -Ao %cpu,comm | grep -E 'mds|mdworker|suggestd|photolibraryd|contactsd' | sort -rn | head                      # indexing churn
```

### 4.1 Deeper Mac checks

No sudo needed:

```bash
# Why did it last shut down or restart? Positive codes are software, negative are hardware.
log show --last 3d --predicate 'eventMessage CONTAINS "Previous shutdown cause"' --style compact | tail -5
# SSD wear and lifetime writes (smartmontools 7.5+ reads Apple silicon NVMe without sudo)
command -v smartctl >/dev/null || brew install smartmontools
smartctl -a disk0 | grep -E 'Percentage Used|Data Units Written|Available Spare|Media and Data Integrity|Unsafe Shutdowns|Power On Hours'
ioreg -rc IOBlockStorageDriver -w0 | grep -oE '"Bytes \(Write\)"=[0-9]+' | sort -t= -k2 -nr | head -1   # bytes written since boot (busiest disk; the first entry can be 0)
# Swap's share of this boot's writes: (vm_stat Swapouts x 16384) / Bytes (Write); >10% means swap meaningfully adds wear
iostat -d -w 2 -c 5 disk0                         # live throughput; an idle Mac should sit near zero
footprint -p <PID>                                 # one process's real dirty memory (system-wide footprint -a needs sudo)
top -l 2 -o cpu -n 15 -stats pid,command,cpu,mem,power | tail -16
mdutil -s /                                       # Spotlight indexing state
tmutil listlocalsnapshots / | wc -l               # local snapshots holding purgeable space
launchctl list | grep -vc '^-' ; ls ~/Library/LaunchAgents /Library/LaunchAgents /Library/LaunchDaemons 2>/dev/null | wc -l
pmset -g log | grep -E 'Wake from|DarkWake|Entering Sleep|Wake reason' | tail -20       # wake storms
```

Shutdown cause codes:

| Code | Meaning |
|---|---|
| 5 | Clean shutdown |
| 3 | Forced (power‑button) shutdown |
| 0 | Power loss |
| -60 | Battery fully drained |
| -61 / -62 | Watchdog: an unresponsive app or WindowServer forced a shutdown or restart |
| -64 | Kernel panic |
| -71 / -74 / -86 / -95 / -100 / -101 | Memory, battery, proximity, CPU, power‑supply or display overheating |
| -75 / -78 | Charger communication or current problem |
| -103 / -104 | Battery voltage or battery fault |
| -128 | Unknown, often memory or after a panic |

With sudo (check `sudo -n true`; otherwise give the user the exact command or skip):

```bash
sudo powermetrics -n 1 -i 5000 --samplers tasks,thermal,cpu_power --show-process-energy | head -60   # per-process energy, thermal pressure
sudo footprint -a | tail -8                                 # system-wide sys_footprint
sudo spindump -notarget 10 10 -o $S/spindump.txt          # what everything is doing during a stall
sudo fs_usage -w -f filesys mds_stores | head -40         # who is writing (swap one process name for another)
sudo sfltool dumpbtm | grep -E 'Name:|Developer Name:|Disposition:' | head -80   # Login Items and Background Task Management entries
```

WindowServer CPU above about 15% usually means one of these:
- a browser repainting many background tabs (Chrome Memory Saver or tab discarding helps);
- many windows, Spaces or external displays with mismatched refresh rates;
- transparency effects (System Settings › Accessibility › Display › Reduce transparency);
- screen‑recording or capture apps.

If `mds`/`mdworker`/`backupd`/`kernel_task` lead in `top` instead, it's indexing, Time Machine, or thermal management rather than the browser.

### 4.2 Background services audit (launch agents and daemons)

Always-on helpers are a frequent hidden cause: one service that is set up badly costs CPU, disk writes and memory around the clock.

```bash
python3 -I - <<'P'
import plistlib,glob,os,collections
rows=[]
for f in glob.glob(os.path.expanduser('~/Library/LaunchAgents/*.plist'))+glob.glob('/Library/LaunchAgents/*.plist')+glob.glob('/Library/LaunchDaemons/*.plist'):
  try: d=plistlib.load(open(f,'rb'))
  except Exception: continue
  prog=d.get('Program') or (d.get('ProgramArguments') or [''])[0]
  logs=[(d.get(k), os.path.getsize(d.get(k))) for k in ('StandardOutPath','StandardErrorPath') if d.get(k) and d.get(k)!='/dev/null' and os.path.exists(d.get(k))]
  rows.append((d.get('Label'),prog,d.get('KeepAlive'),d.get('ThrottleInterval'),logs,f))
print('logs >20MB:',[(r[0],p,round(s/1e6)) for r in rows for p,s in r[4] if s>20e6])
c=collections.Counter(r[1] for r in rows); print('same program in several plists:',[(p,n) for p,n in c.items() if n>1 and p])
print('program missing:',[(r[0],r[1]) for r in rows if r[1].startswith('/') and not os.path.exists(r[1])])
print('KeepAlive:',[(r[0],r[3]) for r in rows if r[2]])
P
launchctl list | awk 'NR>1 && $2!=0 && $2!="-"' | grep -v 'com\.apple\.'                     # failing agents
for l in $(launchctl list | awk 'NR>1 && $2!=0 && $2!="-"{print $3}' | grep -v '^com\.apple\.'); do launchctl print gui/$(id -u)/$l 2>/dev/null | grep -E 'runs =|last exit|last terminating'; done
```

Patterns and fixes:

| Pattern | Evidence | Fix |
|---|---|---|
| Log that never rotates | A StandardOut/ErrorPath file of hundreds of MB, often duplicating the app's own daily log | Send the agent's output to `/dev/null` when the app already logs, or cap it in place (`tail -c 5M f > t && cat t > f`), which keeps the inode for O_APPEND writers |
| KeepAlive crash loop | `runs =` climbing fast plus a non-zero `last exit`, e.g. a startup step hitting a rate limit and the relaunch hitting it again | Raise `ThrottleInterval` (e.g. 300) and reload; fix the startup failure |
| Duplicate autostart | The same program in two plists, e.g. the app's own login item plus a custom KeepAlive agent | Keep the one with the wanted behavior and turn off the other at its source (app setting), so it isn't recreated |
| Cleanup job that kills itself | A reaper whose own command line matches its kill pattern; its log never appears, and `last terminating signal = 15` | Exclude its own PID and script name; then run it with `launchctl kickstart -k` and read the log back |
| Device or network retry loop | One warning dominating the log, e.g. a pinned Bluetooth device that is absent | Follow the system default device instead of a pinned one |
| Dead entries | The program path no longer exists | Harmless; remove only with the user's OK |
| Background AI workflows inside an app | Scheduled agents or summaries (screenpipe pipes, "activities", semantic context; Foresight's 4‑hourly refresh at 100% CPU for hours) running on top of the app's core job | Keep the core feature (recording, sync), and turn off the scheduled AI jobs in the app's settings or pipe files, or remove the job's launch agent. Read back after restart: some apps force settings back on from a built‑in policy |
| Tight polling scripts | A KeepAlive AppleScript or shell loop with `delay 0.5` driving System Events, which then sits at 10–50% CPU | Lengthen the loop delay (e.g. 2 s) rather than removing the automation; measure System Events before and after |

Never disable deliberate business automations (Codex automations, Claude scheduled tasks, publish jobs) as "background load". List them with their schedule and leave them to the user.

Back up every plist and script before editing, reload with `launchctl bootout`/`bootstrap`, and read the state back.

Orphan reapers only catch processes whose parent has exited (ppid 1). Many copies of an MCP server under a **live** long-running agent session are not orphans: report the count and parents instead of killing them silently.

## 5. Interpretation rules (lessons that prevent wrong diagnoses)

- Rule out a recent OS update (2.8) before diagnosing anything long‑term.
- `kernel_task` writing huge amounts is accounting (usually swap), not a culprit in itself. Attribute writes with `fs_usage` or the power-log coalition table.
- A large lifetime SSD write total alone isn't a failure signal. Media errors, critical warnings and falling Available Spare are.

- Verify a cause before blaming it. Battery: check MitigationState and peak power pressure. Heat: check average and maximum temperature. Storage: check free GB. Report these as "not the problem" or leave them out.
- Separate your own collection load from the device's normal behavior. Crash-log transfers flood the logs with `afcd` lines; re-measure on a clean capture.
- Aggregate tables span weeks. Always filter by date before saying "per day".
- One app with hundreds of helper processes is one cause. Group by family before ranking.
- Rank by impact, in this order: system-wide resets and panics, memory kills, freeze counts and their trend, then per-app memory, energy, writes, and background abuse.
- Distinguish **apps the owner uses** (fix their settings: background refresh, autoplay, cache) from **apps that misbehave and aren't essential** (offer deletion, saying their data is safe in the cloud when true).
- Correct earlier statements explicitly when deeper evidence contradicts them.

## 6. Fixes

### Agent-doable (Mac, or with host access)

Do these when they are in scope and the targets are clearly leftovers. Confirm first for anything that might be another agent's or the user's live work.

```bash
xcrun simctl shutdown all                          # idle simulators
pkill -f chrome-devtools-mcp                       # only after confirming they are orphaned (hours old, no live parent session)
brew install libimobiledevice ideviceinstaller     # missing tooling
```

- Never kill processes you didn't start without the user's OK when other agents or sessions may own them. Leftovers older than the current sessions, whose parent is a dead or detached session, are the usual safe set.
- Restarting the Mac clears swap. Recommend it; don't do it without asking.
- Settings changes on the user's own Mac (Spotlight exclusions, login items) need the user's go-ahead.

### Owner-doable (on the device; always explain, never do silently)

Typical high-impact iOS/iPadOS fixes, ordered by impact for the evidence found:

1. Turn off **Background App Refresh** globally or per app.
2. Turn off video autoplay and clear the cache in the worst social or video app.
3. Delete misbehaving non-essential apps (fake audio or calls, heavy background writes), stating that cloud data is kept.
4. Clear caches in heavy chat apps. Clear "cache" only; never chat history.
5. Turn off automatic **App Updates** and update manually while charging.
6. Restart the device.
7. Battery service if health is ≤ 80%, framed as battery life, not speed, unless throttling was proven.
8. On older devices running a newer OS: turn on **Reduce Motion** and **Reduce Transparency**, which cut GPU blur/animation cost. If Apple Intelligence is on and unused, turning it off frees on‑device model work and about 7 GB.
9. Storage or System Data bloat: install the latest point update; clear Safari website data; empty Recently Deleted (Photos, Messages); set Messages › Keep Messages to 1 year; **Offload** (keeps data) or delete and reinstall cache‑heavy apps; restart. Erase‑and‑restore is the last resort.
10. Review apps with **Always** location access. They can run in the background regardless of Background App Refresh.

Menu paths by UI language. Use the device's language from section 1:

| Setting | English | Español | 中文 |
|---|---|---|---|
| Background refresh | Settings › General › Background App Refresh | Ajustes › General › Actualización en segundo plano | 设置 › 通用 › 后台App刷新 |
| Auto app updates | Settings › App Store › App Updates | Ajustes › App Store › Actualizaciones de apps | 设置 › App Store › App更新 |
| Delete app | Long-press icon › Remove App › Delete App | Mantener pulsado › Eliminar app | 长按图标 › 移除App › 删除App |
| Storage | Settings › General › iPhone Storage | Ajustes › General › Almacenamiento del iPhone | 设置 › 通用 › iPhone储存空间 |
| Shut down | Settings › General › Shut Down | Ajustes › General › Apagar | 设置 › 通用 › 关机 |
| Battery health | Settings › Battery › Battery Health & Charging | Ajustes › Batería › Salud y carga de la batería | 设置 › 电池 › 电池健康与充电 |
| Developer Mode | Settings › Privacy & Security › Developer Mode | Ajustes › Privacidad y seguridad › Modo de desarrollador | 设置 › 隐私与安全性 › 开发者模式 |
| Reduce Motion | Settings › Accessibility › Motion › Reduce Motion | Ajustes › Accesibilidad › Movimiento › Reducir movimiento | 设置 › 辅助功能 › 动态效果 › 减弱动态效果 |
| Reduce Transparency | Settings › Accessibility › Display & Text Size › Reduce Transparency | Ajustes › Accesibilidad › Pantalla y tamaño del texto › Reducir transparencia | 设置 › 辅助功能 › 显示与文字大小 › 降低透明度 |
| Offload app | Settings › General › iPhone Storage › app › Offload App | Ajustes › General › Almacenamiento del iPhone › app › Desinstalar app | 设置 › 通用 › iPhone储存空间 › App › 卸载App |
| Safari data | Settings › Apps › Safari › Clear History and Website Data | Ajustes › Apps › Safari › Borrar historial y datos de sitios web | 设置 › App › Safari › 清除历史记录与网站数据 |
| Location access | Settings › Privacy & Security › Location Services | Ajustes › Privacidad y seguridad › Localización | 设置 › 隐私与安全性 › 定位服务 |

Translated labels can shift between OS versions. If a label doesn't match, confirm it on a screenshot of the device or from the device's UI language before sending.

Third-party apps keep their own language. Give in-app paths in the app's language (for example 我 › 设置 › 通用设置 › 视频自动播放 in Weibo).

## 7. Owner-facing report

Write for the device owner, who may not be the person asking.

- Use the owner's language, with menu labels in the device UI language. Put them in visible "chip" style when rendered.
- Open critically and concretely: numbers from the device's own records (memory kills per week, freezes per day and the trend, system resets). Say "the phone's own diagnostic records show…".
- Name the 1–2 main culprits with the strongest evidence; don't list every minor finding.
- **Reassure before asking:** turning off Background App Refresh does **not** stop notifications. Messages, calls and calendar reminders are pushed by the server and keep arriving; apps just load fresh content a second later when opened. Clearing a cache doesn't delete accounts, chats, or favorites.
  - Push notifications go through Apple's push service (APNs), independent of Background App Refresh.
  - The only exception to mention when relevant: mail accounts set to *fetch* rather than push check less often until opened.
- Don't tell the owner to delete apps they use. Adjust settings or close them after use instead.
- Leave out things that aren't problems (healthy storage, a battery that isn't throttling) unless the owner is worried about them.
- Keep it short: 4–6 steps, "takes 5 minutes", ending with the benefit.
- Visual options:
  - an inline HTML/SVG widget;
  - a generated poster through OpenRouter (`OPENROUTER_API_KEY` from the user's global env; model `google/gemini-3-pro-image`, `modalities: ["image","text"]`, `image_config: {aspect_ratio: "9:16", image_size: "2K"}`). Give it every exact string to render, then read the image back and check every word and number. Fix small defects (a duplicated row, for example) by cropping and splicing with PIL at uniform background rows rather than regenerating, and save the deliverable to the user's report folder.
- Optional audio: read the plan aloud in the owner's language with TTS, speaking the content directly with no meta instructions.

## 8. Output contract

Finish with:

1. Device(s) checked, how they were reached (direct shell, sandbox override, Desktop Commander), and which evidence layers were read (crash logs, Jetsam, analytics, live log, sysdiagnose power log, `vm_stat`).
2. The root cause in one sentence, with its 3–5 strongest numbers.
3. Corrections to earlier assumptions, if any.
4. Fixes applied by the agent, each with read-back proof.
5. The owner's step list (localized), plus any blocked item with the exact missing access.

## Env vars

- `OPENROUTER_API_KEY`: only for optional poster generation. Use the user's existing global env (`~/.env`, shell profile, or the user's secret manager). Test it with `curl -s https://openrouter.ai/api/v1/key -H "Authorization: Bearer $OPENROUTER_API_KEY"` before use; a dead key returns `User not found`.
- `PYMOBILEDEVICE3_UDID`: optional; selects the device for `pymobiledevice3`.
