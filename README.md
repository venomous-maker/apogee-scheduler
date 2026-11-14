# apogee-scheduler

Scheduled tasks for [Apogee](https://github.com/venomous-maker), as Laravel's
scheduler. A task can be a function, a queued job, a console command or a
shell command. Each task has:

- a frequency, written with cron expressions or Laravel's methods
- a time zone and conditions
- hooks and pings
- overlap and one-server locks
- an output file

Tasks can run as often as every second. The `schedule:*` console commands
come with it.

```grav
import "std"
import "apogee-scheduler"

let schedule = new Schedule(cache)                  // locks live in the cache: share it across servers
schedule.timezone = "Africa/Nairobi"
schedule.queue = queue                               // for job()
schedule.commands = fn(line: string) -> int { return console.call(line) }   // for command()

schedule.command("emails:digest --to=admins").dailyAt("07:00").weekdays()
schedule.job(new PruneSessions(30), "maintenance").hourly().onOneServer()
schedule.call(fn() { metrics.flush() }).everyTenSeconds()
schedule.exec("pg_dump app > /backups/app.sql").daily().appendOutputTo("/var/log/backup.log")
    .withoutOverlapping().onFailure(fn(t: ScheduledTask) { pager.alert(t.label()) })
```

Cron calls `schedule:run` every minute:

```
* * * * * cd /srv/app && ./app schedule:run >> /dev/null 2>&1
```

In development, `./app schedule:work` runs the schedule in the foreground
instead.

## Frequencies

| | |
|---|---|
| Cron | `cron("*/15 9-17 * * 1-5")` |
| Seconds | `everySecond`, `everyTwoSeconds`, `everyFiveSeconds`, `everyTenSeconds`, `everyFifteenSeconds`, `everyTwentySeconds`, `everyThirtySeconds`, `everyNSeconds(n)` |
| Minutes | `everyMinute`, `everyTwoMinutes` … `everyFifteenMinutes`, `everyThirtyMinutes` |
| Hours | `hourly`, `hourlyAt(17)`, `everyOddHour`, `everyTwoHours(15)`, `everyThreeHours`, `everyFourHours`, `everySixHours` |
| Days | `daily`, `dailyAt("13:00")`, `at("9:30")`, `twiceDaily(1, 13)`, `twiceDailyAt(1, 13, 15)` |
| Weeks | `weekly`, `weeklyOn(1, "8:00")`, `weekdays`, `weekends`, `mondays` … `sundays`, `days([0, 3])` |
| Months | `monthly`, `monthlyOn(4, "15:00")`, `twiceMonthly(1, 16, "13:00")`, `lastDayOfMonth("17:00")` |
| Quarters, years | `quarterly`, `quarterlyOn(1, "0:00")`, `yearly`, `yearlyOn(6, 1, "17:00")` |
| Windows | `between("8:00", "17:00")`, `unlessBetween("23:00", "4:00")` (a window can span midnight) |

These methods combine as in Laravel, so `weekdays().at("9:30")` gives
`30 9 * * 1-5`.

**Time zones:** each task reads the time in its own zone (`timezone("America/New_York")`),
else the schedule's, else the system's.

**Seconds tasks:** these make `schedule:run` stay for the rest of the minute,
running them on their seconds until `schedule:interrupt`.

## Cron expressions

- **Fields:** `CronExpression` takes five fields, or six with seconds first.
- **Values:** `*`, `?`, numbers, names (`JAN`, `MON`), lists, ranges and steps.
- **Day of month:** `L` (the last day) and `LW` (the last weekday).
- **Day of week:** `5L` (the last Friday), `1#2` (the second Monday), and 0 or
  7 for Sunday.
- **Both day fields restricted:** a day matches if either field does, as in
  Vixie cron.
- **Shortcuts:** `@yearly`, `@monthly`, `@weekly`, `@daily`, `@hourly`.
- **Methods:** `isDue(moment)`, `nextRunDate(from)`, `previousRunDate(from)`
  and `nextRunDates(from, n)`. They jump whole months, days and hours that
  can't match, so even `0 0 29 2 *` takes a few hundred steps.

## Conditions, locks, hooks

**Conditions:**

- `when(fn)`, `skip(fn)`
- `environments(["production"])`
- `evenInMaintenanceMode()`. Otherwise the task waits while `schedule.maintenance()`
  says the app is down.

**Locks:**

- `withoutOverlapping(minutes)` doesn't start while the last run still goes.
- `onOneServer()` makes one server per run take the task, through a lock in
  the shared cache.

**Running it:**

- `runInBackground()` runs the task in a thread, so the next ones start at
  once. The threads are joined before `schedule:run` exits.

**Hooks:**

- `before`, `after`/`then`, `onSuccess`, `onFailure`
- `pingBefore(url)`, `thenPing(url)`, `pingOnSuccess(url)`, `pingOnFailure(url)`

**Output:** `sendOutputTo(path)` and `appendOutputTo(path)` keep a command's
output.

**Labels:** `name("…")` and `description("…")` show in `schedule:list`.

**After a run:** a task keeps `lastOutput`, `exitCode` and `lastError`.

**Events:** `ScheduledTaskStarting`, `ScheduledTaskFinished`,
`ScheduledTaskFailed` and `ScheduledTaskSkipped`, when `schedule.events` is
set.

## Commands

`registerScheduleCommands(console)` adds these; the `Schedule` comes from the
container:

| Command | What it does |
|---|---|
| `schedule:run` | the due tasks, then the seconds tasks for the minute |
| `schedule:work` | every minute, in the foreground |
| `schedule:list [--timezone=] [--next]` | each task's cron, label, and when it's next due |
| `schedule:test [--name=]` | a task now, chosen by name or from a list |
| `schedule:interrupt` | stops a running `schedule:run`'s seconds tasks |
| `schedule:clear-cache` | releases overlap locks |

## Tests

```
grav test tests/scheduler.grav
```
