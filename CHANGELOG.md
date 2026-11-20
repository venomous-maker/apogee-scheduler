# Changelog

Every release of apogee-scheduler, newest first. Each is a `vX.Y.Z` tag with a GitHub
release carrying its section; the `grav` line in `grav.toml` says which
compiler versions it builds with.

## Unreleased

## 0.2.1 (2026-11-20)

### Requirements

- Uses the 0.2.1 releases of apogee-queue, apogee-console and grav-cache.

## 0.2.0 (2026-11-20)

### Requirements

- Needs grav 1.9.0 or later, and uses the 0.2.0 releases of the packages it
  depends on.

## 0.1.1 (2026-11-19)

- Uses the newer releases of apogee-queue, grav-db, apogee-console, grav-cache, grav-orbit (faster row loading and query building in grav-orbit and grav-db).

## 0.1.0 (2026-11-19)

The first tagged release. Needs grav 1.7.0 or newer.

Scheduled tasks for [Apogee](https://github.com/venomous-maker), as Laravel's scheduler. A task can be a function, a queued job, a console command or a shell command. Each task has:
