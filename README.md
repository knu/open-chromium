# open-chromium

Command line tools for opening URLs in Chromium-based browsers on macOS — picking a profile, an Edge workspace, or a specific window, and reusing an existing tab when one is already open.

`open-chromium` drives Microsoft Edge, Google Chrome, Chromium, and other Chromium variants via JXA, so it can do things `open -a` cannot: select a profile through the Profiles menu, target a window by title or by Edge workspace, and activate an already-open tab matching a URL.  `open-browser` is a thin wrapper that picks the first installed candidate browser (Edge → Chrome → Chromium by default) and forwards to `open-chromium`.  `chromium-profile` is a helper that resolves profile display names to on-disk directories and looks up Edge workspaces by name or UUID.

## Installation

Drop the scripts somewhere on your `PATH`:

```console
% git clone https://github.com/knu/open-chromium.git
% cp open-chromium/bin/* ~/bin/
```

Keep the three scripts (`open-browser`, `open-chromium`, `chromium-profile`) in the same directory so they can find each other.

### Requirements

- macOS with at least one Chromium-based browser installed (Microsoft Edge, Google Chrome, Chromium, ...).
- `jq` on `PATH` — used by `open-browser` to read each browser's `Local State`.  Install via `brew install jq`.
- Ruby — used by `chromium-profile`.  The system Ruby works.
- `leveldb` — only needed when resolving Microsoft Edge workspaces by name.  Install via `brew install leveldb`.
- [`macwin-cli`](https://github.com/knu/macwin) — optional.  When present, `open-chromium` uses it to raise a single window by title without disturbing the rest of the browser's window stack.  Without it, the script falls back to raising the whole application.

## Usage

```console
% open-browser [-a APPS] [-p PROFILE] [-d PROFILE_DIR] [-w WINDOW] [-i] [-I] URL
% open-chromium [-a APP] [-p PROFILE] [-d PROFILE_DIR] [-w WINDOW] [-i] [-I] URL
% chromium-profile [-a APP] dir PROFILE_NAME
% chromium-profile [-a APP] [-p PROFILE_DIR] workspace (uuid NAME | name UUID | list)
```

Open a URL in the first installed browser (Edge, Chrome, then Chromium):

```console
% open-browser https://example.com/
```

Pick a profile by its display name (as it appears in the browser's Profiles menu):

```console
% open-browser -p Personal https://example.com/
```

Pick a profile by its on-disk directory (e.g. `Default`, `Profile 2`).  `-d` wins over `-p`:

```console
% open-browser -d "Profile 2" https://example.com/
```

Pick the browser explicitly.  Multiple candidates can be given with `:`, tried in order:

```console
% open-browser -a "Google Chrome:Microsoft Edge" https://example.com/
```

Activate (or open in) a window with a given title.  Multiple candidates can be given with `:`:

```console
% open-browser -w "Work" https://example.com/
```

For Microsoft Edge, `-w` also accepts a workspace UUID or workspace name.  An already-open window matching the title is preferred; only if no such window is open is a workspace launch attempted:

```console
% open-browser -a "Microsoft Edge" -p Work -w "Project Alpha" https://example.com/
```

A leading `!` negates the match: `-w '!Name'` selects any window whose title is *not* `Name` (preferring the front window), creating a new one if every open window is titled `Name`.  This is handy for keeping a dedicated window — say one named `Workspace` — untouched while routing everything else elsewhere:

```console
% open-browser -w '!Workspace' https://example.com/
```

Open in incognito mode:

```console
% open-browser -i https://example.com/
```

Reuse an existing tab when the URL matches ignoring the query string:

```console
% open-browser -I https://example.com/search?q=foo
```

If a tab for the URL is already open in any window of the chosen profile, that tab is activated instead of opening a new one.  If a `chrome://newtab/` or `edge://newtab/` tab is open, it is reused.  When the URL is omitted, `open-chromium` just raises the matching window.

## chromium-profile

`chromium-profile` looks up Chromium browser metadata directly from `Local State` and (for Edge workspaces) the profile's `Sync Data/LevelDB`:

```console
% chromium-profile -a "Microsoft Edge" dir Personal
Default
% chromium-profile -a "Microsoft Edge" -p Default workspace list
01234567-89ab-cdef-0123-456789abcdef	Project Alpha
...
% chromium-profile -a "Microsoft Edge" -p Default workspace uuid "Project Alpha"
01234567-89ab-cdef-0123-456789abcdef
% chromium-profile -a "Microsoft Edge" -p Default workspace name 01234567-89ab-cdef-0123-456789abcdef
Project Alpha
```

Exit status is `0` on a unique match, `1` on no match, `2` on ambiguous match, and `3` on a usage or runtime error.

## Options

### `open-browser`

- `-a APPS` — `:`-separated list of preferred browsers, tried in order; falls back to `Microsoft Edge:Google Chrome:Chromium`.
- `-p PROFILE` — `:`-separated list of profile display names (as shown in the Profiles menu).
- `-d PROFILE_DIR` — On-disk profile directory (e.g. `Default`, `Profile 2`).  Wins over `-p`.
- `-w WINDOW` — `:`-separated list of window titles, or — for Microsoft Edge — workspace names or UUIDs.  A leading `!` (e.g. `!Name`) selects any window other than the named one, creating a new window if none exists.
- `-i` — Open in incognito mode.
- `-I` — Match tabs by URL ignoring the query string.

### `open-chromium`

Same flags as `open-browser`, except `-a` takes a single application name.

### `chromium-profile`

- `-a APP` — Browser app name (default: `Microsoft Edge`).
- `-p PROFILE_DIR` — Profile directory name (default: `Default`).

## How it works

- Profile selection drives the browser's Profiles menu via System Events, so it works even when a browser does not expose `--profile-directory` on relaunch.
- When `macwin-cli` is available, the target window is raised by title without disturbing the rest of the application's window stack.  Without it, `open-chromium` falls back to app-level activation.
- Edge workspaces are resolved from the profile's `Sync Data/LevelDB`.  If Edge is running and holds the database lock, `chromium-profile` snapshots the LevelDB files to a temporary directory and opens that copy.

## Author

Copyright (c) 2026 Akinori Musha.

Licensed under the MIT license.  See `LICENSE` for details.

Visit the [GitHub Repository](https://github.com/knu/open-chromium) for the latest information.
