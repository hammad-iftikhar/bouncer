# Bouncer — Product Spec

## Overview

Bouncer is an open-source macOS menu bar app that keeps the Finder's "Open With" menu limited to the apps you actually install. macOS registers every app bundle it finds anywhere on disk, so Xcode debug builds, archives, and stray copies in Downloads pile into the menu. Bouncer lets you declare which folders count as real app locations and continuously unregisters everything else.

**Who it is for**: macOS developers who build apps locally (Xcode, Electron, Tauri, Flutter) and anyone whose Open With menu has grown unusable. v1 ships as a free download and source code on GitHub.

**Working name**: Bouncer. Tagline: _Only apps on the list get in._

## Goals and non-goals

**Goals**

- The Open With menu shows only apps under user-approved roots, within seconds of a new build appearing.
- Zero maintenance after setup: watch, detect, unregister, repeat, with a periodic sweep as a backstop.
- Nothing destructive. Bouncer never deletes files and never resets the Launch Services database.
- Every action is reversible from the app (re-register with one click).
- A clear enforcement toggle so the user chooses between automatic, ask-first, and preview-only.
- A small default-app manager so users can also set which app opens each file type.

**Non-goals**

- Mac App Store distribution. Bouncer is released on GitHub instead.
- Hiding individual apps inside an allowed folder, other than through a block rule.
- Managing the Dock, Spotlight, or the Finder sidebar.
- Supporting macOS older than 13 Ventura.

## Constraints

macOS keeps a system list of every app it has seen, and the Open With menu is built from that list. There is no way to tell macOS to only look in certain folders, so Bouncer works by removing unwanted entries continuously rather than by changing a setting once.

- An app removed from the list stops responding to its links and file types until it is allowed again. This matters for developers testing those features in a debug build.
- macOS can re-add an app it discovers again, so Bouncer keeps watching rather than cleaning once.
- Bouncer never deletes files and never resets the system list, since a full reset can break System Settings on recent macOS versions.
- Bouncer cannot be distributed through the Mac App Store.

## Core features

### Policy: allowed roots and rules

An app is **allowed** if its path is under an allowed root, or matches an allow rule. Everything else is an **offender**. Rules are evaluated in order: explicit block rules, then explicit allow rules, then allowed roots.

- Default allowed roots: `/Applications`, `/System/Applications`, `~/Applications`, plus other app locations Bouncer detects on the machine, such as Homebrew's Caskroom and Application Support helper apps. Detected locations are proposed during onboarding and, when new ones appear, as a suggestion in the Rules view; they are never added silently. Users can add and remove roots.
- Allow rules: a single app ("Always allow this build") or a pattern, e.g. every Release build of one project.
- Block rules: an app or pattern that is removed even inside an allowed root, e.g. an "Old Versions" folder.
- Bundle identifier rules: allow or block an app by its identifier, useful when its path changes on every build.

### Watching and sweeping

- Bouncer watches a list of folders for new or changed apps. Defaults: Xcode's DerivedData and Archives. Users can add any folder, such as Electron, Tauri, or Flutter build output.
- A new build is handled a few seconds after it finishes (default 5 s, adjustable), so Bouncer never races the build tool.
- A full sweep of everything runs on a schedule (default every 15 min), on wake from sleep, on launch, and on demand from the menu bar.
- Failures are retried on the next sweep and shown in History.

### Enforcement modes (user setting, Settings > General)

| Mode      | Behaviour                                                                            | Notification                        |
| --------- | ------------------------------------------------------------------------------------ | ----------------------------------- |
| Automatic | Unregister offenders as soon as they are found                                       | Optional summary ("Removed 3 apps") |
| Ask first | Found offenders are queued; a notification offers Remove / Allow once / Always allow | Always, with actions                |
| Preview   | Nothing is changed. Offenders show in the menu with a count badge                    | None                                |

Default for a fresh install is **Preview**, so the user sees what Bouncer would do before it does anything. Onboarding ends with a one-click switch to Automatic.

### Offender list and actions

Each offender row shows icon, name, version, bundle ID, path (shortened), and first-seen date. Actions: Unregister, Allow once (this path), Always allow (adds an allow rule), Block (adds a block rule), Reveal in Finder, Copy path.

### Undo and history

Every removal is logged with the app, its path, when it happened, and what triggered it (watch, sweep, or manual). The History view lists the last 500 entries with a Re-register button that puts the app back exactly as it was. "Undo last sweep" restores everything from the most recent batch. History is kept for 30 days.

### Launch at login

A single toggle in Settings. Bouncer appears in System Settings > General > Login Items like any other app.

## Default-app manager

A second tab in the main window lists file types and URL schemes with the app that currently opens each, and lets the user change it without digging through Finder's Get Info panel.

- **Rows**: a curated starter list of about 60 common file types grouped by category (Documents, Code, Images, Video, Audio, Archives, Web links) plus a search field that accepts an extension (`.md`) or a link type (`mailto`).
- **Picker**: the choice of apps for a type is filtered by the Bouncer policy, so debug builds never appear as options. A "Show all" toggle lifts the filter.
- **Bulk set**: select several rows and assign one app, e.g. every code type to one editor.
- **Presets**: a small bundled set ("Developer: all code files to editor X") plus the user's own saved presets. Presets export with everything else.
- Changes made here are logged to History and can be reverted.

## Rules export and import

All configuration lives in one JSON file so it can be versioned, shared, or synced through a dotfiles repo.

- **Export**: File > Export Rules saves one file containing allowed roots, watched folders, allow and block rules, enforcement settings, default-app choices, and presets.
- **Import**: merges by default (new entries added, duplicates skipped) with a "Replace all" option.
- **Portable**: home-folder paths are stored relative to the user, so a rules file works on another Mac. The importer warns about folders that do not exist on this machine instead of dropping them silently.
- **Versioned**: files from older Bouncer versions import cleanly into newer ones.
- **Dotfiles-friendly**: the rules file can be pointed at a location of the user's choosing, such as a dotfiles repo.

## User interface

Native macOS look and feel. Bouncer lives in the menu bar with no Dock icon by default, with an option to show one.

### Menu bar

A door icon. A badge shows the offender count in Preview and Ask-first modes.

- Status line: "3 apps outside allowed folders" or "All clear".
- Offender list (up to 10, then "Show all...") with inline Unregister / Allow buttons.
- Buttons: Sweep now, Pause for 1 hour, Open Bouncer, Settings, Quit.

### Main window

A sidebar window with four sections.

1. **Offenders**: the table described under Core features, with multi-select and a toolbar for bulk Unregister / Allow.
2. **Default Apps**: the manager described above.
3. **History**: log of actions with Re-register and Undo last sweep.
4. **Rules**: allowed roots, watched folders, allow/block rules, each editable in place with drag-and-drop of folders from Finder.

### Settings

- General: enforcement mode (with a one-line description of each), launch at login, show Dock icon, check for updates.
- Timing: delay after a build, sweep interval, sweep on wake.
- Notifications: per-event toggles (offender found, batch removed, error) and whether notifications carry action buttons.
- Advanced: rules file location, open log folder, reset onboarding.

### Onboarding (first launch)

1. One screen explaining what Bouncer does and that it uses `lsregister`, with a link to the GitHub README.
2. Auto-detect: Xcode folders found, proposed watched folders and roots, current offender count.
3. Choose enforcement mode. Preview is preselected.
4. Spotlight tip: if a watched folder is not in Spotlight's privacy list, show a one-click button that opens System Settings > Spotlight and copies the folder path.

### Notifications

Standard macOS notifications. Ask-first notifications are persistent: they stay on screen until the user acts on them, and carry Remove, Allow once, and Always allow as action buttons so the common case needs no window. Onboarding asks the user to set Bouncer's notification style to Alerts, since macOS Banners dismiss on their own.

## Distribution

- **Requires**: macOS 13 Ventura or later.
- **Where**: GitHub Releases (ZIP) and the source repository, with build-from-source instructions in the README.
- **Trust**: no Apple Developer account, so the app is not notarized. The first launch needs a right-click > Open (or "Open Anyway" in System Settings > Privacy & Security); the README and release notes walk through this with a screenshot. Users who prefer can build it themselves from source in Xcode. No extra permissions are requested; if a user adds a protected folder such as Desktop, macOS asks for access to that folder once.
- **Updates**: v1 checks GitHub for a newer release and links to it. Automatic in-app updates are a later addition.
- **License**: MIT, source on GitHub with a README, screenshots, and contribution guide.

## Edge cases and risks

| Case                                                 | Behaviour                                                                                        |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Xcode re-adds a build right after Bouncer removes it | Bouncer retries once, then waits for the next sweep and logs a warning                           |
| User is testing a link or file type in a debug build | "Allow once" and "Pause for 1 hour" in the menu bar; Ask-first mode makes this the default       |
| macOS rediscovers a removed app later                | Known limitation; onboarding and Rules prompt the user to exclude watched folders from Spotlight |
| A future macOS removes the ability to unregister     | Enforcement switches off with a clear banner; Preview and the default-app manager keep working   |
| App deleted between detection and removal            | Skipped silently; stale entries are cleaned on the next sweep                                    |
| Apps installed via Homebrew or symlinked             | Treated as allowed; the Homebrew folder is a default root                                        |
| Apps on external or network drives                   | Not watched by default; shown with a volume badge, and the drive can be added as a root          |
| Same app registered from several paths               | Each path is listed separately; identifier rules apply to all of them                            |
| User removes `/Applications` from allowed roots      | Allowed after a confirmation sheet                                                               |
| Very large DerivedData folders                       | First sweep may take a few seconds and shows progress                                            |

## Milestones

Build order, each milestone usable on its own. No dates set yet.

1. **M1 Core loop**: rules, detection, and manual sweep from a bare menu bar item. Proves the approach on a real Xcode project.
2. **M2 Watch and enforce**: folder watching, the three enforcement modes, history, and re-register.
3. **M3 Full UI**: main window (Offenders, History, Rules), Settings, onboarding, notifications with actions, launch at login.
4. **M4 Default-app manager** and rules export/import.
5. **M5 Release**: release build, ZIP on GitHub Releases, README with install and first-launch steps, screenshots, 1.0.
