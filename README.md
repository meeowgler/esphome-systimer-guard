# systimer_guard for ESPHome

An ESPHome external component that works around the ESP32-C6 SYSTIMER fault described in
[espressif/esp-idf#19036](https://github.com/espressif/esp-idf/issues/19036): on some chips,
the counter behind `esp_timer` (SYSTIMER counter 0) suddenly loses minutes to hours, and its
snapshot handshake can wedge.

**Status: experimental.** It has corrected 16 real events on 3 boards so far (see [Results](#results)).
Use it only if your C6 boards show the symptoms below, and please report how it behaves.

## Related work

[danjurgens/esp32c6-systimer-backstep](https://github.com/danjurgens/esp32c6-systimer-backstep)
(GPL-3.0) is where this fault was pinned down: a bare ESP-IDF reproducer, the register-level
analysis showing the bit-clear signature, a ledger of verified events, and its own ESPHome
`clock_guard` and `loop_watchdog` components, which have absorbed real events in production
since September 2026. **If you want the established option, use that one.**

This component was written independently, and was called `clock_guard` until 25 October 2026;
it was renamed to avoid confusion with danjurgens' component of that name. The main
differences: it compares the two SYSTIMER hardware counters directly, every 1 ms, with
interrupts off, rather than `esp_timer` against the FreeRTOS tick count; its counters reset at
boot instead of persisting in NVS; and it has no correction cap, since a backward step can
never be larger than the counter's own value.

## Results

- **25 October 2026, first real event:** on the board that had stalled most (20 times in 17
  days), `esp_timer` lost 2667.577 s (44.5 min). That is exactly bits 28, 29, 30, 31, 32 and 35
  of the 16 MHz counter cleared at once. The guard put the time back within its 1 ms check, and
  the board's 60-second uptime reports carried on without a gap; before the guard, the same
  fault silenced the board for the time lost. Counter 1 did not move backward.
- **26 October 2026:** the same board lost 1149.239 s (19.2 min), bits 27, 30 and 34.
  Corrected with no gap.
- **27 October 2026:** a board that had never shown a stall lost 4.079 s, bits 18 and 21 to 25.
  Events this small were invisible before: a few seconds' silence in a 60-second report never
  looked like a stall.
- **27 October 2026, Wi-Fi power saving off:** to see whether modem sleep sets the fault off, the
  worst board was switched to `power_save_mode: NONE`. Seven and a half hours later it lost
  1166.016 s (19.4 min), bits 27, 28, 30 and 34, corrected with no gap. A diagnostic sensor
  reading `esp_wifi_get_ps()` confirmed on 28 October that the driver keeps NONE with Bluetooth
  running (ESPHome logs a refused setting only at verbose level, so this needed checking). So on
  this board modem sleep is not the trigger; turning it off only avoids the Wi-Fi hang described
  under [Limits](#limits).
- **30 October 2026:** a third board, which had stalled once before the guard, lost 205.521 s,
  bits 26, 30 and 31. The worst board lost 71.303 s (bits 26 and 30), and the 27 October board
  0.491 s (bits 19 to 22).
- **31 October 2026, the largest so far:** the worst board lost 244813.13 s (68.0 hours), bits
  36, 39, 40 and 41. Without the guard it would have gone quiet for almost three days. Going by
  its uptime at the time (about 253,600 s), bit 37 was also set and survived, so this was not a
  whole bit group cleared. That is worked out from the loss, not from a raw capture.
- **2 November 2026:** the worst board had five events, of about 4.20 s, 2998.91 s, 2.92 s,
  2.09 s and 1858.06 s. Their bits can't be named: these losses come from the `time_recovered`
  total, a 32-bit float that only resolves about 0.02 s once it passes 245,000 s. The
  `last_event` sensor records exact raw values instead.
- **Below bit 26:** the 4.079 s and 0.491 s events lie entirely below bit 26, outside the two
  bit groups (26 to 35 and 36 to 51) seen in the raw captures on the upstream issue.
- **Since 3 November 2026** the worst board runs with `load_canary` and `last_event` (see
  [Options](#options)), to record raw values for the upstream issue.
- **4 November 2026, first raw captures, all on the worst board.** Counter 0 lost time three
  times. The load registers held the canary each time, and none of its bits turned up:
  - 67.109 s: only bit 30 cleared; bits 27, 33, 34 and 35 in the same group stayed set.
  - 1547.698 s: bits 26 to 35 all cleared (26, 30, 31, 32 and 34 were set).
  - 4261.413 s: bits 26 to 35 all cleared (29 to 35 were set).

  Had the 26-to-35 group been loaded from the load registers, it would have come back with the
  canary's bits 26, 29, 31, 32 and 34 set. It came back zero, so these were clears, not
  spurious loads, and the first was not a whole group.
- **4 November 2026, counter 1 loses time too.** Two captures first logged as `ahead` (35 ms
  and 94 ms) were counter 1 losing time, not counter 0 jumping forward. When counter 1 loses
  time, the FreeRTOS tick waits for it to count back up to its next alarm, so between two checks
  counter 1 still moves 1 ms while counter 0 moves the real time; counter 1 never looks as if it
  went backward. The first lost exactly bits 15 and 19 (557,056 ticks, to within 80 ticks); the
  second about 94 ms, which fits bits 0 to 20 cleared. Since 4 November the guard reads the RTC
  timer as a third clock to tell the two cases apart, and counts these as tick counter jumps.
- **To 4 November 2026:** 16 real events on counter 0 on 3 of 12 boards since the guard went on
  all of them on 25 October, all corrected; 13 were on the worst board. Counter 1 lost time
  twice, both on the worst board. No stalls and no snapshot recoveries.

  The raw captures (counter values in 16 MHz ticks):

  ```
  lost 67.109 s;   u0 0x0001e49ebf020 -> 0x0001e09ec2ea0; u1 0x0001e49eb6b95 -> 0x0001e49ebaa15; load 0x5a5a5a5a5a5a5
  lost 1547.698 s; u0 0x00055c4acf705 -> 0x0005000ad356a; u1 0x00055c4ac7329 -> 0x00055c4acb196; load 0x5a5a5a5a5a5a5
  lost 4261.413 s; u0 0x0008fe3bc4452 -> 0x0008003bc826f; u1 0x0008fe3bbc1bc -> 0x0008fe3bc0016; load 0x5a5a5a5a5a5a5
  ahead 0.035 s;   u0 0x000ef9fe908a1 -> 0x000ef9ff1c789; u1 0x000ef9fe88795 -> 0x000ef9fe8c62d; load 0x5a5a5a5a5a5a5
  ahead 0.094 s;   u0 0x000fa551fb884 -> 0x000fa5536da36; u1 0x000fa5516b796 -> 0x000fa5516f68b; load 0x5a5a5a5a5a5a5
  ```
- Simulated 600 s jumps (see [Testing it](#testing-it)) are corrected to the millisecond.
- A simulated wedged handshake (`simulate_wedge()`) ends in an abort about a second later, and
  the board restarts with an `Abort` crash record and reset reason "exception/panic".

## Symptoms

`esp_timer`, and with it ESPHome's `millis()`, uptime and scheduler, steps backward. ESPHome's
next scheduled run is then due at a time the clock has to count back up to, so:

- every `update_interval` sensor (uptime, debug, Wi-Fi signal) goes quiet for exactly the time
  that was lost, often 15 minutes to several hours;
- uptime then resumes from where it stopped, so it ends up behind the wall clock;
- event-driven work keeps running: a Bluetooth proxy keeps forwarding adverts through it.

Some affected chips also crash: an interrupt or task watchdog with the backtrace in
`esp_timer_impl_get_time` / `systimer_hal_get_counter_value`, where the counter's snapshot
"value valid" flag never sets.

## How it works

The C6 has a second SYSTIMER counter (counter 1, which drives the FreeRTOS tick), running from
the same clock. Their difference is constant, so a task at the highest priority wakes every
1 ms and compares the two. If `esp_timer` has fallen behind by more than the threshold, it
steps `esp_timer` forward by exactly the lost time (`esp_timer_private_advance()`) and re-arms
the `esp_timer` alarm, so timers that fell due in the meantime run at once.

- Both counters are read with interrupts off. A slow read can only make `esp_timer` look
  ahead, never behind, so delays cannot cause a false correction. A clock that is ahead is
  never touched.
- A counter that stops, rather than jumps back, is stepped forward until it runs again;
  corrections less than a second apart count as one event.
- If counter 1 itself loses time, the guard re-learns the offset and counts it as a "tick
  counter jump" instead of acting. A counter 1 loss usually looks like `esp_timer` jumping
  ahead, because the tick waits for counter 1 to catch up, so the guard checks the RTC timer to
  see which counter moved.

### A wedged snapshot handshake

ESP-IDF reads a counter by requesting a snapshot and spinning, without a limit, until its
"value valid" flag sets. If the flag never sets, every `esp_timer` reader hangs until a
watchdog resets the board. The guard's own reads are bounded and re-request the snapshot up
to four times; a re-request can also release the other readers, which wait on the same flag,
and each one is counted as a "snapshot recovery". If the reads still fail for about a second,
the guard calls `esp_system_abort()` (without logging, since the logger reads the clock), so
the board restarts promptly with a crash record naming the cause instead of waiting for a
watchdog.

Each check takes a few microseconds; on a busy Bluetooth proxy the main loop time went from
19 to 20 ms.

## Limits

- **ESP32-C6, ESP-IDF framework only.**
- **No automatic light sleep.** Power management (`CONFIG_PM_ENABLE`) and tickless idle step
  both counters during sleep, so the build stops with an error if either is on.
- **Wi-Fi modem sleep** (`power_save_mode: LIGHT`, the ESP32 default) works with the guard,
  but danjurgens reports that with modem sleep on, the fault can also hang the Wi-Fi PHY path
  faster than any poll can act (an interrupt watchdog reset). `power_save_mode: NONE` avoids
  that form (the driver does apply it alongside Bluetooth), but not the fault itself: a board with
  it set still lost 1166 s (see [Results](#results)).
- It works around the symptom; it does not fix the cause. A wedged handshake still costs a
  restart if re-requesting the snapshot does not clear it.
- `esp_timer_private_advance()` is an ESP-IDF private API. Check that the component still
  builds and behaves after ESP-IDF updates.
- Tested on ESPHome 2026.10.0 (ESP-IDF 5.5) with Seeed XIAO ESP32-C6 boards, chip revisions
  0.1 and 0.2.

## Usage

```yaml
external_components:
  - source: github://meeowgler/esphome-systimer-guard
    components: [systimer_guard]

systimer_guard:
  corrections:
    name: "Clock Corrections"
  time_recovered:
    name: "Clock Time Recovered"
  tick_counter_jumps:
    name: "Clock Tick Counter Jumps"
  snapshot_recoveries:
    name: "Clock Snapshot Recoveries"
```

### Options

- **threshold** (*Optional*, time, default `2ms`, minimum `500us`): how far `esp_timer` may
  fall behind before it is corrected.
- **update_interval** (*Optional*, time, default `60s`): how often the sensors publish. They
  also publish at once after any event.
- **corrections** (*Optional*, sensor): number of correction events since boot.
- **time_recovered** (*Optional*, sensor, seconds): total time put back since boot.
- **tick_counter_jumps** (*Optional*, sensor): how often counter 1 lost time. The fault hits
  counter 1 too (see [Results](#results)).
- **snapshot_recoveries** (*Optional*, sensor): reads that only succeeded after the snapshot
  was re-requested.
- **last_event** (*Optional*, text sensor): the raw values around the latest event: what
  happened (`lost`, `ahead` or `tick counter lost`), counter 0 and counter 1 at the check before
  and the check that caught it, counter 0's load registers, which bits of counter 0 differ from
  where it should be (the XOR; its lowest few bits also carry the read delay), and the time
  between the two checks by the RTC timer. Each event is logged as a warning too, as
  `Raw capture: ...`.
- **load_canary** (*Optional*, boolean, default `false`): keep the pattern `0x5A5A5A5A5A5A5`
  in counter 0's load registers, as Espressif asked in
  [esp-idf#19036](https://github.com/espressif/esp-idf/issues/19036). If an event loads part of
  the counter from those registers rather than clearing it, the pattern shows up in the counter.
  The registers are written without being applied, so the counter itself is not touched; the
  guard rewrites the pattern after each correction, since `esp_timer_private_advance()` uses the
  same registers. A spurious load can move the clock forward, which the guard records but does
  not undo.

Each event is also logged as a warning:

```
[W][systimer_guard]: esp_timer lost 600.000 s and was stepped forward (1 corrections, 600.0 s in total)
```

### Testing it

`simulate_jump(seconds)` steps `esp_timer` back the way the fault does (never further than
the uptime allows). `simulate_wedge()` makes the guard's own reads fail, so it should abort
and the board restart about a second later. Keep test buttons disabled by default:

```yaml
systimer_guard:
  id: systimer_guard_1

button:
  - platform: template
    name: "Clock Guard Test"
    entity_category: diagnostic
    disabled_by_default: true
    on_press:
      - lambda: id(systimer_guard_1).simulate_jump(600);
  - platform: template
    name: "Clock Guard Wedge Test"
    entity_category: diagnostic
    disabled_by_default: true
    on_press:
      - lambda: id(systimer_guard_1).simulate_wedge();
```

## License

MIT
