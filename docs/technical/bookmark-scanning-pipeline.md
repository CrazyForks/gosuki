# Bookmark Scanning Pipeline

Internal developer reference for gosuki's bookmark scanning, parsing,
synchronization, and persistence architecture.

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Module Lifecycle & Browser Registration](#2-module-lifecycle--browser-registration)
3. [File Watching System](#3-file-watching-system)
4. [Per-Browser Bookmark Parsing](#4-per-browser-bookmark-parsing)
5. [Tree Building & URL Index](#5-tree-building--url-index)
6. [Hooks System](#6-hooks-system)
7. [Database Architecture](#7-database-architecture)
8. [Sync Pipeline](#8-sync-pipeline)
9. [xhsum & Lamport Clock](#9-xhsum--lamport-clock)
10. [Tag Handling Algorithm](#10-tag-handling-algorithm)
11. [P2P Sync Foundations](#11-p2p-sync-foundations)
12. [Known Issues & Architectural Gaps](#12-known-issues--architectural-gaps)

---

## 1. System Architecture Overview

### High-Level Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    BOOTSTRAP (one-time)                      │
│                                                             │
│  Module Registration → Profile Detection → Browser Setup    │
│         ↓                                                    │
│  InitCache(ctx) → SyncFromDisk → SetupBrowser(PreLoad)      │
│         ↓                                                    │
│  All PreLoads complete → Watchers activated                 │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│              RUNTIME LOOP (per browser)                      │
│                                                             │
│  fsnotify event → Run() / ReduceEvents → Run()             │
│         ↓                                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              BROWSER-SPECIFIC PARSING               │    │
│  │                                                     │    │
│  │ Chrome:  Full JSON rebuild → Tree + URLIndex        │    │
│  │ Firefox: Incremental CTE scan → update Tree+Index   │    │
│  │ Qute:    Line-by-line text parse → Bookmark list    │    │
│  └──────────────────┬──────────────────────────────────┘    │
│                     ↓                                        │
│  ┌─────────────────────────────────────────────────────┐    │
│  │               HOOKS (per bookmark)                  │    │
│  │                                                     │    │
│  │ ParseNodeTags / ParseBkTags → extract #tag, @action │    │
│  └──────────────────┬──────────────────────────────────┘    │
│                     ↓                                        │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           BUFFER → CACHE SYNC (per browser)         │    │
│  │                                                     │    │
│  │ SyncURLIndexToBuffer / SyncTreeToBuffer → buffer    │    │
│  │ buffer.SyncToCache() → Cache(UpsertBookmark)        │    │
│  └──────────────────┬──────────────────────────────────┘    │
│                     ↓                                        │
│  ┌─────────────────────────────────────────────────────┐    │
│  │       SCHEDULED BACKUP (debounced, ~4s)             │    │
│  │                                                     │    │
│  │ Cache.SyncTo(L2Cache) → L2.BackupToDisk(gosuki.db) │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### Key Design Invariants

1. **Buffers are ephemeral**: Each browser has its own in-memory SQLite buffer.
   Buffers are created fresh per `SetupBrowser()` and live for the process
   lifetime. They accumulate entries across all `Run()` cycles.

2. **Cache is additive**: The global Cache (L1) merges data from all browser
   buffers using union semantics on tags. A tag, once in cache, persists
   until explicitly deleted through the admin interface.

3. **URLIndex is the source of truth per browser**: For Firefox, the URLIndex
   (in-memory RB-Tree) — not the tree — drives buffer synchronization. The
   tree exists only for representing bookmark hierarchy.

4. **L2 Cache is a disk mirror**: L2Cache keeps an in-memory replica of
   `gosuki.db`. It enables xhsum-based change detection to avoid redundant
   disk I/O.

5. **xhsum gates all persistence**: A bookmark is only propagated to disk
   (and triggers P2P sync) when its xhsum changes. xhsum =
   `xxhash(url + metadata + tags + desc)`.

---

## 2. Module Lifecycle & Browser Registration

### Module Types

Gosuki has two module categories:

| Type | Interface | Description |
|------|-----------|-------------|
| **Browser** | `BrowserModule` = `Browser` + `Module` | Watches local browser bookmark sources |
| **Generic** | `Module` | Fetches bookmarks from arbitrary sources (pollers, loaders) |

### Registration Flow

All modules self-register in their package-level `init()` function:

```go
// browsers/chrome/chrome.go
func init() {
    modules.RegisterBrowser(Chrome{ChromeConfig: ChromeCfg})
}

// browsers/firefox/firefox.go
func init() {
    modules.RegisterBrowser(Firefox{FirefoxConfig: FFConfig})
}
```

Registration stores modules in `registeredBrowsers` and `registeredModules`
global slices. The `init()` functions are triggered when the packages are
imported (via blank imports in the build).

### Browser Setup Sequence

`SetupBrowser()` in `pkg/modules/browser.go` orchestrates initialization:

```
1. [Initializer|ProfileInitializer].Init(c, p)
   → Browser-specific setup: detect profiles, configure paths, set up watchers

2. Create per-browser BufferDB (in-memory SQLite with unique name)
   → database.NewBuffer(bConf.Name)
   → Schema: gskbookmarks table (same as disk DB)

3. Create in-memory URLIndex (RB-Tree)
   → bConf.URLIndex = index.NewIndex()

4. [PreLoader].PreLoad(c)
   → Full initial scan of all bookmarks
   → Populates BufferDB, URLIndex, NodeTree
```

### Interface Implementation Matrix

| Interface | Chrome | Firefox | Qutebrowser |
|-----------|--------|---------|-------------|
| `BrowserModule` | ✅ | ✅ | ✅ |
| `ProfileInitializer` | ✅ | ✅ | ❌ |
| `Initializer` | ❌ | ❌ | ✅ |
| `PreLoader` | ✅ | ✅ | ✅ |
| `WatchRunner` | ✅ | ✅ | ✅ |
| `Shutdowner` | ✅ | ✅ | ❌ |
| `Detector` | ✅ (config.go) | ❌ | ✅ |
| `ResetWatcher` | ✅ | ❌ | ❌ |
| `HookRunner` | ✅ (via BrowserConfig) | ✅ (via BrowserConfig) | ✅ |

### Lifecycle Diagram

```
init() package
   │
   ├─ RegisterBrowser(Firefox{FFConfig})
   │      │
   │      └─ registeredBrowsers = [..., Firefox]
   │
   ▼
cmd run (main.go)
   │
   ├─ database.Init(ctx)
   │     ├─ RegisterSqliteHooks()
   │     ├─ initCache(ctx) → Cache + L2Cache created
   │     ├─ startSchedulers() → cacheSyncScheduler goroutine
   │     └─ SyncFromDisk() if gosuki.db exists
   │
   ├─ For each browser in registeredBrowsers:
   │     SetupBrowser(browser, ctx, profile)
   │         ├─ Init(ctx, profile)     → setup watchers
   │         ├─ NewBuffer(name)        → per-browser buffer
   │         ├─ URLIndex.NewIndex()    → per-browser index
   │         └─ PreLoad(ctx)           → full initial scan
   │
   └─ For each WatchWork unit:
         manager spawns goroutine → WatchLoop() → wait for events
```

---

## 3. File Watching System

### Components

The watching system lives in `pkg/watch/` and has three layers:

| Component | Location | Role |
|-----------|----------|------|
| `fsnotify` | OS-level | Raw filesystem event notifications |
| `WatchDescriptor` | `watcher.go` | Wrapper around `*fsnotify.Watcher` with per-module watch rules |
| `Watch` | `watcher.go` | Single watched path: events types + event name filters |
| `ReduceEvents` | `reducer.go` | Debounce/merge rapid-fire events |

### Watch Descriptor Creation

Each browser module defines its watches during `Init()`:

**Chrome:**
```go
w := &watch.Watch{
    Path:       bookmarkDir,            // ~/.config/google-chrome/Default/
    EventTypes: []fsnotify.Op{fsnotify.Create},
    EventNames: []string{bookmarkPath}, // .../Bookmarks
    ResetWatch: false,
}
```

Chrome watches for `Create` events on the `Bookmarks` file because Chrome
writes its bookmark data atomically — each change creates a new file via
rename.

**Firefox:**
```go
w := &watch.Watch{
    Path:       watchedPath,            // ~/.mozilla/firefox/XXXX.default/
    EventTypes: []fsnotify.Op{fsnotify.Write},
    EventNames: []string{.../places.sqlite-wal},
    ResetWatch: false,
}
```

Firefox watches the WAL (Write-Ahead Log) file because Firefox uses SQLite's
WAL journal mode. All writes go through the WAL before being checkpointed
into `places.sqlite`. Watching the WAL catches all modifications.

**Qutebrowser:**
```go
// Two separate watches:
w := &watch.Watch{ Path: bookmarkDir, EventTypes: [...Create], EventNames: [urls] }
wQuickmarks := &watch.Watch{ Path: baseDir, EventTypes: [...Create, ...Write], EventNames: [quickmarks] }
```

Qutebrowser has two separate files (`urls` and `quickmarks`) so it registers
two watch descriptors.

### Event Dispatch: WatchLoop

```
fsnotify event arrives
    │
    ├─ Match against watched.EventTypes?
    └─ Match against watched.EventNames?
           │
           ├─ If reducer enabled (eventsChan != nil):
           │     → Forward event to eventsChan
           │     → ReduceEvents goroutine handles debounce
           │
           └─ If no reducer:
                 → Reset parsing counter
                 → Call module.Run() immediately in new goroutine
```

### Event Reducer (`ReduceEvents`)

Used by Firefox only. The reducer debounces rapid events:

```
Event 1 arrives → timer reset to interval (1500ms)
Event 2 arrives → timer reset again (events accumulated)
...
timer expires   → Run() called once with all events consumed
```

This is necessary because Firefox may generate a burst of WAL write events
during a single bookmark operation. Without debouncing, `Run()` would fire
multiple times unnecessarily, each time spawning a new `places.sqlite` copy
and executing the expensive CTE queries.

### Chrome ResetWatcher

Chrome implements `ResetWatcher`. If `watch.ResetWatch == true`, after
processing an event:

```go
r.ResetWatcher()      // Close old fsnotify.Watcher, create new one
break watchloop       // Exit WatchLoop so it restarts with the new watcher
```

This is needed because on some systems, `fsnotify` loses track of a file
after rename. Chrome's `ResetWatcher` re-watches the directory from scratch.
Currently `ResetWatch: false` in production (the issue appears resolved).

---

## 4. Per-Browser Bookmark Parsing

### 4.1 Chrome: Full JSON Tree Rebuild

**Source**: `browsers/chrome/chrome.go`

Chrome bookmarks are stored in a JSON file (`Bookmarks`). Chrome writes this
file atomically for every change (rename-to-new), which is why the watcher
looks for `Create` events.

#### Parsing Algorithm

```
1. Read entire Bookmarks file into memory
2. jsonparser.Get(f, "roots") → extract root containers
3. Walk root objects:
     For each child node in children array:
       ├─ Parse {type, name, url, children}
       ├─ Create tree.Node
       │     └─ module = "chrome_{flavour}_{profile}"
       ├─ If folder (type="folder"):
       │     push to parent stack
       │     recurse into children array
       │     pop from parent stack
       │
       └─ If URL (type="url"):
             ├─ Check URLIndex for existing entry
             ├─ If new: Insert into URLIndex, CallHooks()
             ├─ If existing: Compare NameHash(title)
             │     └─ If hash changed: CallHooks()
             │
             └─ If parent is folder:
                   currentNode.Tags = append(..., Parent.Title)
```

#### Key Characteristics

| Property | Behavior |
|----------|----------|
| **Parse scope** | Full rebuild every `Run()` |
| **URLIndex lifecycle** | Replaced via `RebuildIndex()` after tree walk |
| **Tag sources** | Parent folder names (tree hierarchy) + hooks (`#tag`, `@action`) |
| **Change detection** | `NameHash = xxhash(title)` — hooks only re-run if title changed |
| **Buffer sync** | `SyncTreeToBuffer(tree, buffer)` walks tree, upserts each URL node |

#### Tags in Chrome

Chrome does not have a native "tag" concept. Tags are derived from:

1. **Folder membership**: If a bookmark's immediate parent is a folder, the
   folder's name is added as a tag. A bookmark can belong to multiple folders
   (via Chrome's "organized by star" behavior), yielding multiple tags.

2. **Hook-derived**: The `node_tags_from_name` hook scans the title for
   `#word` patterns and extracts them as tags, stripping them from the title
   simultaneously.

3. **Action tags**: `@actionname` patterns are extracted similarly but kept
   with the `@` prefix in the tag list.

#### Important Detail: URLIndex Replacement

After parsing, Chrome replaces its entire URLIndex:

```go
// After tree walk:
ch.RebuildIndex()
```

Which does:
```go
func (b BrowserConfig) RebuildIndex() {
    b.URLIndex = index.NewIndex()  // ← Fresh index
    tree.WalkBuildIndex(b.NodeTree, b.URLIndex)
}
```

This means the URLIndex always reflects the current state of the bookmark
file. Bookmarks deleted from Chrome's JSON file will not appear in the new
URLIndex. However, they persist in the buffer (see §12).

---

### 4.2 Firefox: Incremental SQLite CTE Scan

**Source**: `browsers/firefox/firefox.go`

Firefox bookmarks are stored in a SQLite database (`places.sqlite`) with two
key tables:

| Table | Content |
|-------|---------|
| `moz_places` | URLs, titles, descriptions, visit counts |
| `moz_bookmarks` | Bookmark entries: type (1=bookmark, 2=tag, 3=folder),
parent-child relationships, `lastModified` timestamps |

Firefox uses SQLite's WAL journal mode. Tags are stored as special bookmark
entries (type=2) under the virtual "tags" folder (id=4).

#### Scanning Strategies

**PreLoad (full scan)** — `scanBookmarks()`:

```sql
-- recursive-all-bookmarks.sql
WITH RECURSIVE
  folder_marks(...) AS (...),     -- CTE: all folders and their hierarchy
  bk_in_folders(...) AS (...),    -- CTE: bookmarks inside folders
  tags(...) AS (...),             -- CTE: tag entries under folder id=4
  marks(...) AS (...),            -- CTE: join bookmarks to tags + folders
  ...
SELECT placeId, title, tags, parentFolderId, folders, url, plDesc, lastModified
FROM all_bookmarks
GROUP BY placeId
ORDER BY lastModified
```

This is a recursive CTE that walks the `moz_bookmarks` tree to resolve:
- Folder hierarchy (recursive)
- Bookmark-to-folder membership
- Bookmark-to-tag associations (via type=2 entries in folder 4)
- Multi-folder bookmarks (group_concat)

**Run() (incremental scan)** — `scanModifiedBookmarks(since timestamp)`:

Same CTE structure, but with a temporal filter:
```sql
WHERE lastModified > :change_since
```

The `since` timestamp is computed as:
```go
scanSince := ff.lastRunAt.Add(-1 * time.Second)
```

The 1-second rollback avoids missing changes due to clock skew between the
WAL event timestamp and gosuki's wall clock.

#### Firefox Tag Handling in Scan

```go
func (f *Firefox) loadBookmarksToTree(bookmarks []*MozBookmark, runTask bool) {
    for _, bkEntry := range bookmarks {
        created, urlNode := f.addURLNode(bkEntry.URL, bkEntry.Title, bkEntry.PlDesc)

        // For each tag on this bookmark:
        for _, tagName := range strings.Split(bkEntry.Tags, ",") {
            if tagName == "" { continue }
            seen, tagNode := f.addTagNode(tagName)

            // CRITICAL: Tags are accumulated, never replaced
            urlNode.Tags = utils.Extends(urlNode.Tags, tagNode.Title)
            tree.AddChild(f.tagMap[tagNode.Title], urlNode)
        }

        // Link to folder parent if applicable
        folderNode, fOk := f.folderMap[bkEntry.ParentID]
        if fOk { tree.AddChild(folderNode, urlNode) }

        // Run hooks
        err := f.CallHooks(urlNode)
    }
}
```

#### Key Characteristics

| Property | Behavior |
|----------|----------|
| **Parse scope** | Full on PreLoad, incremental on Run() |
| **URLIndex lifecycle** | Persistent across runs, extended incrementally |
| **Tag sources** | Explicit tags (moz_bookmarks type=2) + hooks |
| **Change detection** | `lastModified` timestamp in places.sqlite |
| **Buffer sync** | `SyncURLIndexToBuffer(URLIndexList, URLIndex, buffer)` — iterates all index entries |
| **Places access** | Copy of `places.sqlite` to tmp dir (avoid VFS lock) |

#### Places Copy Mechanism

Firefox's `places.sqlite` is locked by the browser process. To avoid SQLite
VFS lock errors, gosuki copies `places.sqlite*` (including WAL and SHM files)
to a temporary directory before opening it:

```go
func (f *Firefox) initPlacesCopy() (mozilla.PlaceCopyJob, error) {
    pc := mozilla.NewPlaceCopyJob()   // Creates tmp/XXXXX/
    utils.CopyFilesToTmpFolder(..., pc.Path())
    f.places = database.NewDB("places", path.Join(pc.Path(), "places.sqlite"), ...)
}
```

Old copy jobs are cleaned up after 1 hour.

#### Important Detail: URLIndex Persistence

Firefox's `URLIndex` is **never** cleared or rebuilt from scratch during
`Run()`. Entries are added on first encounter and updated (title/desc) on
subsequent scans. 

This incremental approach is a deliberate performance optimization:
- Avoids re-executing expensive CTE queries on every change event
- Avoids copying `places.sqlite` to tmp on every event (the copy happens once per Run() call, but the index avoids re-scanning unchanged entries)

However, it creates correctness gaps: deleted bookmarks and removed tags
persist in the URLIndex forever. See §12.2.

---

### 4.3 Qutebrowser: Line-by-Line Text Parse

**Source**: `browsers/qute/qutebrowser.go`

Qutebrowser stores bookmarks as plain text files:
- `urls`: One bookmark per line, format: `URL Title`
- `quickmarks`: Multiple space-separated fields, last field is URL, preceding
  fields are tags

#### Parsing Algorithm

```go
func (qu *Qute) loadBookmarks(runTask bool) {
    // File: urls — format: "URL Title words"
    for line in file:
        fields := strings.Fields(line)
        bk := Bookmark{
            URL: fields[0],
            Title: join(fields[1:], " "),
            Module: qu.Name,
        }
        qu.CallHooks(bk)         // Extract #tags from title
        qu.BufferDB.UpsertBookmark(bk)  // Direct upsert, no tree/index
}

func (qu *Qute) loadQuickMarks(runTask bool) {
    // File: quickmarks — format: "tag1 tag2 URL"
    for line in file:
        fields := strings.Fields(line)
        bk := Bookmark{
            Tags: fields[:len(fields)-1],  // All except last
            URL:  fields[len(fields)-1],   // Last field
            Module: qu.Name,
        }
        qu.CallHooks(bk)
        qu.BufferDB.UpsertBookmark(bk)
}
```

#### Key Characteristics

| Property | Behavior |
|----------|----------|
| **Parse scope** | Full reload every `Run()` / `PreLoad()` |
| **URLIndex** | Not used (direct buffer upsert) |
| **Tag sources** | Quickmark fields + hooks (`bk_tags_from_name`) |
| **Buffer sync** | Direct `UpsertBookmark` calls during load, then `SyncToCache()` |
| **File watcher** | Two watches: `urls` (Create), `quickmarks` (Create+Write) |

---

## 5. Tree Building & URL Index

### Node Types

```
NodeType:
  RootNode   — Top-level container (browser-specific name: "ROOT", "menu", etc.)
  FolderNode — Browser bookmark folder
  TagNode    — Bookmark tag (Firefox: moz_bookmarks type=2)
  URLNode    — Bookmark leaf (has URL, title, tags, desc)
```

### Tree Structure

**Chrome**: Tree mirrors the bookmark JSON hierarchy. Folders are parent nodes,
URLs are leaves. Tags come from folder parents and hooks.

```
ROOT
├── Bookmarks Bar/         ← FolderNode
│   ├── Google             ← URLNode, tags=["Bookmarks Bar"]
│   └── Work/              ← FolderNode
│       └── GitHub         ← URLNode, tags=["Work"]
└── Other Bookmarks/       ← FolderNode
    └── Example            ← URLNode, tags=["Other Bookmarks"]
```

**Firefox**: Tree has a separate "TAGS" branch for tag nodes:

```
ROOT
├── TAGS/                  ← FolderNode (special, id=4)
│   ├── work               ← TagNode
│   │   ├── Google         ← URLNode (added as child, but parent points to folder)
│   │   └── GitHub         ← URLNode
│   └── important          ← TagNode
│       └── Google         ← URLNode
├── Bookmarks Menu/        ← FolderNode
│   ├── Google             ← URLNode
│   └── Work/              ← FolderNode
│       └── GitHub         ← URLNode
└── Other Bookmarks/       ← FolderNode
    └── Example            ← URLNode
```

**Critical detail**: URL nodes point their `Parent` field to folder parents
only. Tag parent relationships are stored in the tagMap/children but not in
`URLNode.Parent`. This is because a URL can have multiple tags but only one
folder parent. Tags for a URL are resolved via `getTags()`:

```go
func (node *Node) getTags() []string {
    root := node.GetRoot()
    // Traverse tree to find all folder ancestors → add as tags
    parentFolders := FindParents(root, node, FolderNode)
    // Traverse tree to find all tag ancestors → add as tags
    parentTags := FindParents(root, node, TagNode)
    node.Tags = combine(parentFolders + parentTags)
    return node.Tags
}
```

### URL Index (RB-Tree)

**Type**: `hashmap.RBTree` (from `github.com/blob42/hashmap`) — an in-memory
Red-Black tree hashmap providing O(log n) lookups by URL string.

| Browser | URLIndex Usage |
|---------|---------------|
| Chrome | Rebuilt from scratch every `Run()`. Acts as deduplication check during tree walk (skip hooks if title unchanged). Not used for buffer sync — `SyncTreeToBuffer` walks the tree directly. |
| Firefox | Persistent across runs. Source of truth for buffer sync (`SyncURLIndexToBuffer`). Updated incrementally via `addURLNode`. Never pruned. |
| Qute | Not used. Bookmarks upserted directly into buffer during load. |

### URLIndex vs BufferDB

The URLIndex and BufferDB serve complementary roles:

| Property | URLIndex (RB-Tree) | BufferDB (SQLite) |
|----------|-------------------|-------------------|
| Storage | In-memory Go hashmap | In-memory SQLite |
| Primary key | URL string | URL column (UNIQUE constraint) |
| Content | `*tree.Node` pointers | `gskbookmarks` rows |
| Purpose | Fast dedup / lookup during parse | Persist browser state for cache sync |
| Lifecycle | Chrome: rebuilt each Run(). Firefox: persistent. | Accumulates across runs, never cleared |

---

## 6. Hooks System

### Overview

Hooks are pluggable functions executed during bookmark parsing to extract tags,
process commands, or transform data. They run on every bookmark as it enters
the browser's parse pipeline — before the bookmark reaches the buffer.

**Location**: `hooks/` package, `pkg/parsing/tags.go`

### Hook Registry

```go
// hooks/defined.go
var Defined = HookMap{
    "node_tags_from_name": Hook[*tree.Node]{
        name:     "node_tags_from_name",
        Func:     parsing.ParseNodeTags,
        priority: 2,
        kind:     BrowserHook,       // Runs during browser parse
    },
    "bk_tags_from_name": Hook[*gosuki.Bookmark]{
        name:     "bk_tags_from_name",
        Func:     parsing.ParseBkTags,
        priority: 2,
        kind:     BrowserHook,
    },
}
```

### Hook Kinds

| Kind | When Executed | Target Type |
|------|--------------|-------------|
| `BrowserHook` | During browser parse (PreLoad + Run) | `*tree.Node` or `*gosuki.Bookmark` |
| `GlobalInsertHook` | After bookmark inserted into Cache/L2 | `*gosuki.Bookmark` |
| `GlobalUpdateHook` | After bookmark updated in Cache/L2 | `*gosuki.Bookmark` |

### Execution Points

**Browser hooks**: Called by `BrowserConfig.CallHooks()` during parse:
- Chrome: Inside tree walk, after node creation, if URL is new or title changed
- Firefox: Inside `loadBookmarksToTree`, after tag extraction
- Qute: After constructing Bookmark struct from line

**Global hooks**: Dispatched via `hooksQueue` channel during
`SyncToClock()`. Processed by `HooksScheduler()` goroutine.

### Tag Extraction Hooks

`ParseNodeTags` and `ParseBkTags` use regex to extract tags from the title:

| Pattern | Example | Extracted As |
|---------|---------|-------------|
| `#tagname` | `"My #work Bookmark"` | `"work"` (hash stripped from title) |
| `@action` | `"@notify:ping Status"` | `"@notify:ping"` (kept with `@`) |

The title is mutated in-place: matched patterns are removed so the stored
title doesn't contain raw tag markers. This transformation happens before the
bookmark reaches the buffer, so the cached bookmark title never contains
`#tag` or `@action`.

### Hooks and Tag Attribution

Hook-derived tags flow through the same pipeline as folder-based or explicit
tags. They are appended to `node.Tags` or `bk.Tags`, then upserted into the
buffer, then merged in cache. There is no distinction between "hook tag" and
"folder tag" once inside the buffer — all tags are equal strings in the
comma-delimited `tags` column.

---

## 7. Database Architecture

### Schema (v3)

**File**: `internal/database/schema.go`, `CurrentSchemaVersion = 3`

```sql
CREATE TABLE gskbookmarks (
    id          INTEGER PRIMARY KEY,
    URL         TEXT NOT NULL UNIQUE,
    metadata    TEXT DEFAULT '',     -- Title
    tags        TEXT DEFAULT '',     -- Comma-delimited: ,tag1,tag2,
    desc        TEXT DEFAULT '',
    modified    INTEGER DEFAULT (strftime('%s')),
    flags       INTEGER DEFAULT 0,
    module      TEXT DEFAULT '',     -- Source: "chrome_default", "firefox_profile1", etc.
    xhsum       TEXT DEFAULT '',     -- xxhash(url + metadata + tags + desc)
    version     INTEGER DEFAULT 0,   -- Lamport clock value at last modification
    node_id     BLOB                  -- Originating P2P node UUID
);

CREATE TABLE sync_nodes (
    ordinal     INTEGER PRIMARY KEY,
    node_id     BLOB NOT NULL UNIQUE,
    version     INTEGER NOT NULL
);

-- Buku compatibility view + triggers
CREATE VIEW bookmarks AS SELECT id, URL, metadata, tags, desc, flags FROM gskbookmarks;
```

### Database Instances

| Name | Type | Location | Purpose |
|------|------|----------|---------|
| `buffer_{name}_{rand}` | In-memory SQLite | RAM | Per-browser staging area. Accumulates entries across all Run() cycles. Schema: gskbookmarks only. Created via `NewBuffer()`. |
| `Cache` (memcache) | In-memory SQLite | RAM | Global merge of all browser buffers. Source of truth for runtime state. Schema: gskbookmarks + sync_nodes + views. |
| `L2Cache` (memcache_l2) | In-memory SQLite | RAM | In-memory mirror of on-disk gosuki.db. Enables xhsum comparison to avoid redundant I/O. Schema: same as Cache. |
| `DiskDB` (gosuki_db) | File-based SQLite | `~/.local/share/gosuki/gosuki/gosuki.db` | Persistent storage. WAL journal mode. Only written via `BackupToDisk()` from L2Cache. |

### Creation Sequence

```
1. initCache(ctx):
     Cache.DB = NewDB("memcache", "", "file:memcache?mode=memory&cache=shared&_journal=MEMORY").Init()
     Cache.InitSchema(ctx)
     L2Cache.DB = NewDB("memcache_l2", "", same DSN).Init()
     L2Cache.InitSchema(ctx)

2. If gosuki.db exists on disk:
     Cache.SyncFromDisk(dbpath)    → SQLite backup API: disk → L1
     L2Cache.SyncFromDisk(dbpath)  → SQLite backup API: disk → L2

3. For each browser SetupBrowser():
     Buffer = NewBuffer(browserName) → "file:buffer_chrome_abc123?mode=memory..."
     Buffer.InitSchema(ctx)
```

### In-Memory DSN

All in-memory databases use `mode=memory&cache=shared&_journal=MEMORY`:
- `mode=memory`: Entire database lives in RAM, no WAL/shm files on disk
- `cache=shared`: Multiple connections to the same in-memory DB share data
  (required because Buffer uses SetMaxOpenConns(1) but sqlx may open more)
- `_journal=MEMORY`: Transaction journal also in memory

### L2 Cache Rationale

The L2 cache exists to avoid reading `gosuki.db` on every sync cycle. Without
L2, the scheduler would need to:
1. Read gosuki.db to get current xhsum for each bookmark
2. Compare with Cache's xhsum
3. If different, write back to disk

With L2:
1. Cache.SyncTo(L2Cache) — in-memory copy via SyncToClock
2. L2Cache already has the mirror, so comparing xhsum between Cache and L2Cache
   is a fast in-memory operation
3. When differences are detected, L2Cache.BackupToDisk() uses SQLite's native
   backup API (efficient page-level copy)

---

## 8. Sync Pipeline

### Buffer → Cache

Each browser syncs its buffer to the global cache at the end of `Run()` or
`PreLoad()`:

**Chrome / Qute:**
```go
database.SyncTreeToBuffer(ch.NodeTree, ch.BufferDB)  // or direct upserts
err = ch.BufferDB.SyncToCache()                       // buffer → Cache
```

**Firefox:**
```go
database.SyncURLIndexToBuffer(f.URLIndexList, f.URLIndex, f.BufferDB)
err = f.BufferDB.SyncToCache()    // PreLoad: SyncToCache
// or
ff.BufferDB.SyncTo(database.Cache.DB)  // Run(): direct SyncTo
```

#### `SyncToCache` Path

```go
func (src *DB) SyncToCache() error {
    empty, _ := Cache.IsEmpty()
    if empty {
        src.CopyTo(Cache.DB, "main", "main")  // SQLite full copy
    } else {
        src.SyncTo(Cache.DB)                   // Row-by-row merge
    }
}

func (src *DB) SyncTo(dst *DB) {
    src.SyncToClock(dst, Clock.Value)
}
```

### SyncToClock (Row-by-Row Merge)

**File**: `internal/database/sync.go`

This is the core merge function. It handles both local sync (Cache→L2) and
P2P sync (remote→local).

#### Algorithm — Two-Phase Transaction

**Phase 1: Insert new rows**

```
FOR each row in src.gskbookmarks:
    TRY INSERT into dst.gskbookmarks
    IF constraint violation (duplicate URL):
        → Record old xhsum, add to existingUrls map
    ELSE (success):
        → If dst == L2Cache: tick Lamport clock
```

**Phase 2: Update changed rows**

```
FOR each row in existingUrls:
    GET existing tags from dst (SELECT tags FROM gskbookmarks WHERE url=?)
    MERGE tags: newTags = union(srcTags, dstTags)
    newHash = xhsum(url, metadata, newTags, desc)
    IF oldHash == newHash:
        → SKIP (no change)
    ELSE:
        → UPDATE row with merged tags
        → If dst == L2Cache: tick Lamport clock
```

#### Merge Semantics (Critical)

Tags are **always merged additively**:

```go
srcTags := tagsFromString(scan.Tags, TagSep).Sort()
dstTags := tagsFromString(tags, TagSep).Sort()
tagMap := make(map[string]bool)
for _, v := range srcTags.tags { tagMap[v] = true }
for _, v := range dstTags.tags { tagMap[v] = true }
// result: union — never removes tags
```

This means:
- New tags are always added
- Existing tags are never removed through sync
- Title/desc updates use `CASE WHEN ? != '' THEN ? ELSE metadata END` (preserve empty)
- xhsum is recalculated with the merged tag set

### Cache → L2Cache → Disk (Scheduled Backup)

#### Scheduler

```go
func cacheSyncScheduler(input <-chan any) {
    queue := make(chan any, 100)
    timer := time.NewTimer(0)
    for {
        select {
        case <-input:                        // Trigger from ScheduleBackupToDisk()
            timer.Reset(Config.SyncInterval) // Default: 4 seconds
            select {
            case queue <- true:              // Enqueue (drop if full)
            default: log.Debug("queue full")
            }
        case <-timer.C:                      // Debounce expired
            Cache.SyncTo(L2Cache.DB)         // L1 → L2 merge
            L2Cache.BackupToDisk(dbpath)     // L2 → disk (SQLite backup API)
            SyncTrigger.Store(true)          // Signal to other components
            drain(queue)
        }
    }
}
```

#### BackupToDisk (SQLite Backup API)

Uses SQLite's native page-level copy API — not row-by-row INSERT:

```go
bkp, _ := conn1.Backup("main", conn0, "main")
bkp.Step(-1)  // Copy all pages at once
bkp.Finish()
```

Two dedicated connections are maintained in `_sql3BackupConns` for this purpose.

---

## 9. xhsum & Lamport Clock

### xhsum Calculation

```go
func xhsum(url, metadata, tags, desc string) string {
    input := fmt.Sprintf("%s+%s+%s+%s", url, metadata, tags, desc)
    return strconv.FormatUint(xxhash.ChecksumString64(input), 10)
}
```

xhsum is the change detection mechanism for bookmarks. Any change to URL,
title, tags, or description produces a different hash.

#### Where xhsum is set

| Point | Who Sets It | Value |
|-------|------------|-------|
| `UpsertBookmark` (new row) | Browser buffer → Cache | Empty string `""` (placeholder, recalculated later) |
| `UpsertBookmark` (existing row) | Buffer update path | Empty string `""` (placeholder) |
| `SyncToClock` (insert phase) | Cache → L2 sync | Computed: `xhsum(url, metadata, tags, desc)` |
| `SyncToClock` (update phase) | Cache → L2 sync, after tag merge | Computed: `xhsum(url, metadata, mergedTags, desc)` |

The placeholder `""` in buffer upserts is intentional: buffers are ephemeral
and don't need accurate xhsums. The definitive xhsum is set during the
Cache→L2 sync phase, where it's used to detect changes against the disk DB
mirror.

### Lamport Clock

**File**: `internal/database/clock.go`

The Lamport clock provides causal ordering for P2P synchronization:

```go
type LamportClock struct {
    Value uint64
    mu    sync.RWMutex
}

func (c *LamportClock) Tick(peerClock uint64) uint64 {
    c.Value = max(c.Value, peerClock) + 1
    return c.Value
}

func (c *LamportClock) LocalTick() uint64 {
    c.Value += 1
    return c.Value
}
```

#### Tick Points

| Event | Clock Action | Who Calls It |
|-------|-------------|--------------|
| Bookmark inserted into L2Cache | `Clock.Tick(remoteClock)` | SyncToClock insert phase |
| Bookmark updated in L2Cache (xhsum changed) | `Clock.Tick(remoteClock)` | SyncToClock update phase |
| P2P receive from remote node | `Clock.Tick(remoteNodeVersion)` | SyncToClock caller |
| Initialization | `L2Cache.GetDBClock()` → `MAX(version)` from disk DB | database.Init() |

#### Clock and Version Column

The `version` column in `gskbookmarks` stores the Lamport clock value at the
time of last modification. During P2P sync:

1. Remote node sends bookmarks with their version numbers
2. Local node calls `SyncToClock(remote, remoteClock)` where
   `remoteClock = max(versions from remote)`
3. Local clock ticks to `max(local, remote) + 1`
4. New/updated rows get the new clock value as their version
5. This ensures causal ordering across nodes

---

## 10. Tag Handling Algorithm

### Tag Sources

| Source | Chrome | Firefox | Qute |
|--------|--------|---------|------|
| Folder membership | ✅ Parent folder name → tag | ❌ (folders are structural, not tags) | ❌ |
| Explicit tags | ❌ (no native tags) | ✅ `moz_bookmarks` type=2 entries | ✅ quickmarks file fields |
| Hook-derived (`#tag`) | ✅ `node_tags_from_name` | ✅ `node_tags_from_name` | ❌ (uses `bk_tags_from_name`) |
| Hook-derived (`@action`) | ✅ Same hook, kept with prefix | ✅ Same hook | ❌ (uses different hook) |

### Tag Lifecycle (End-to-End)

```
1. BROWSER PARSE
   Folder/Explicit/Hook tags → node.Tags / bk.Tags

2. BUFFER UPSERT
   UpsertBookmark(bk):
     IF new URL: INSERT with raw tags
     IF existing URL: MERGE (union) → tags = cacheTags ∪ incomingTags

3. CACHE SYNC (Buffer → Cache)
   SyncToClock(buffer, Cache):
     IF new: INSERT
     IF existing: MERGE (union) again
     xhsum recalculated with merged tags

4. L2 SYNC (Cache → L2Cache, scheduled)
   SyncToClock(Cache, L2Cache):
     IF new: INSERT, tick clock
     IF existing: MERGE (union) again
     xhsum recalculated with merged tags
     Clock ticks ONLY if xhsum changed

5. DISK BACKUP (L2Cache → gosuki.db, scheduled)
   BackupToDisk: Full SQLite page copy
   (xhsum + version already set in L2)
```

### Merge Points (Three Total)

| Merge Point | Location | Effect |
|-------------|----------|--------|
| #1: `UpsertBookmark` (buffer) | `bookmark_tx.go` | Tags in buffer = cache_tags_in_buffer ∪ new_tags_from_browser |
| #2: `SyncToClock` (Cache) | `sync.go` update phase | Tags in Cache = l2_tags ∪ buffer_tags |
| #3: `SyncToClock` (L2) | `sync.go` update phase | Tags in L2 = cache_tags ∪ l2_tags (already merged) |

Each merge point applies union semantics. The practical effect: a tag from
any browser, once it reaches the buffer, propagates to Cache and L2, and
then to disk. Once there, no subsequent sync cycle can remove it because
the local side of every merge always retains existing tags.

### Tag String Format

Tags are stored as comma-delimited strings with delimiter wrapping:

```
,tag1,tag2,
```

The `StringWrap()` method ensures the string starts and ends with `,`:
- Empty tags → `,`
- Single tag → `,work,`
- Multiple tags → `,important,work,`

This format provides two conveniences:
1. Splitting by `,` produces empty strings at boundaries (easily filtered)
2. No special handling needed for leading/trailing delimiters

### `Tags` Helper Type

```go
type Tags struct {
    delim string  // Always ","
    tags  []string
}

// Construction
NewTags([]string{"a", "b"}, TagSep)           → Tags{tags: ["a", "b"]}
tagsFromString(",a,b,", TagSep)                → Tags{tags: ["a", "b"]}

// Operations
tags.Sort()                                    → alphabetically sorted
tags.PreSanitize()                             → Replace "," within tag names with "--"
tags.String(wrap=true)                         → ",sorted_a,sorted_b,"
tags.Extend(otherTags)                         → append (dedup via utils.Extends)
```

### Critical Gap: Tag Deletion

The merge algorithm is **additive-only**. There is no mechanism for a
browser to signal "this tag should be removed." The consequences:

1. User removes `tag1` from Chrome folder → Next scan: buffer has no `tag1`
   for this bookmark → Merge with cache: cache still has `tag1` → `tag1` stays

2. User removes all tags from Firefox bookmark → Next scan: buffer has empty
   tags → Merge with cache: cache still has old tags → tags stay

3. Same bookmark added in multiple browsers with same tag → Both browsers
   claim the tag → Removing from one browser keeps it via the other →
   Removing from both keeps it because merge always preserves existing


---

## 11. P2P Sync Foundations

### Current State

The database schema includes `version` and `node_id` columns for P2P
synchronization. The Lamport clock provides causal ordering. However, the
actual P2P transport layer is not yet implemented in the codebase — the
database infrastructure is prepared but not connected to a networking stack.

### Prepared Infrastructure

| Component | Status | Purpose |
|-----------|--------|---------|
| `version` column | ✅ Schema v3 | Lamport clock value per bookmark |
| `node_id` column | ✅ Schema v3 | UUID of originating node |
| `sync_nodes` table | ✅ Schema v3 | Track known peers |
| `SyncToClock(src, dst, remoteClock)` | ✅ Implemented | Generic merge with clock awareness. Works for both local (Cache→L2) and P2P (remote→local) sync. |
| Clock initialization from disk | ✅ `GetDBClock()` | Seed local clock from max version in DB |
| Clock tick on update/insert | ✅ In SyncToClock | Ensures monotonic version numbers |

### P2P Sync Semantics (When Implemented)

`SyncToClock` is designed to handle remote bookmarks:

```go
// When receiving from peer:
remote.SyncToClock(local, remoteNodeVersion)
```

For each incoming bookmark:
1. If URL not in local DB → Insert (new, always accept)
2. If URL exists + xhsum changed → Merge tags (union), recalculate xhsum
3. Clock ticks if any change propagated to L2Cache

The tag merge for P2P uses the same union semantics as local sync.

---

## 12. Known Issues & Architectural Gaps

### 12.1 Buffer Staleness (All Browsers)

**Problem**: Per-browser BufferDBs accumulate entries across all `Run()`
cycles and are never cleared. When a bookmark is deleted from the browser,
the old entry persists in the buffer.

**Impact**: `SyncToCache` reads ALL rows from the buffer, including stale
ones. The cache's unique URL constraint prevents duplicate rows, but the
stale entry keeps its old data alive in the cache (merged with existing).

**Chrome specifics**: Chrome rebuilds its NodeTree and URLIndex from scratch
each run. `SyncTreeToBuffer` only writes entries present in the current tree.
However, entries from previous runs that are no longer in the tree remain in
the buffer. The buffer is a superset of the current tree state.

**Firefox specifics**: Firefox's persistent URLIndex compounds this. Deleted
bookmarks stay in the URLIndex (see §12.2), so `SyncURLIndexToBuffer` keeps
syncing them to the buffer.

**Qute specifics**: Like Chrome, Qute loads fresh data each run but the
buffer retains stale entries from previous runs.

**Severity**: Medium. In current design, cache merge is additive and the
unique constraint prevents corruption. Stale entries are mostly invisible.


### 12.2 Firefox URLIndex Persistence (Major, Firefox-specific)

**Problem**: Firefox's URLIndex is never cleared. Deleted bookmarks and
removed tags persist forever.

**Root causes:**

1. **No deletion path**: `addURLNode` only inserts (new) or updates
   (title/desc). There is no code path that removes a URL from the index.

2. **Incremental scan**: `scanModifiedBookmarks` returns only modified
   bookmarks. Deleted bookmarks don't appear in results, so the index is
   never informed of their deletion.

3. **Tag accumulation**: `utils.Extends` appends tags to existing tag lists.
   Removed tags from the browser's tag list are never subtracted from
   `urlNode.Tags`.

4. **URLIndexList append-only**: `f.URLIndexList = append(f.URLIndexList, url)`
   only adds. When `SyncURLIndexToBuffer` iterates this list, it syncs every
   historically-known URL.

**Impact:**

| Scenario | Effect |
|----------|--------|
| User deletes bookmark in Firefox | Stays in URLIndex → stays in buffer → stays in cache forever |
| User removes tag from bookmark | `urlNode.Tags` keeps old tags → buffer has stale tags → cache merge keeps them |
| User renames folder (Chrome) | N/A — Chrome rebuilds tree each run |

**Severity**: High. This is an active correctness bug, not a latent one. It
means the gosuki database can contain bookmarks and tags that no longer exist
in Firefox.

**Relation to design intent**: The incremental approach was deliberately chosen
for performance. The trade-off between performance and correctness
was not explicitly evaluated for the deletion case.

### 12.3 Tags Are Additive-Only at All Merge Points

**Problem**: Three separate merge points (buffer upsert, Cache sync, L2 sync)
all use union semantics. A tag cannot be removed by any browser through the
sync pipeline.

**Impact**: Tags can never be deleted through normal browser activity. The
only way to remove a tag is via the CLI/admin interface.

**Relation to other issues**: This is the primary cause of the "tags stay
forever" behavior. Even if buffer staleness and URLIndex persistence were
fixed, tags would still persist because the cache merge is additive. This is a
semantic choice, not an implementation bug.


### 12.4 PreLoad vs Run Asymmetry

**Problem**: `PreLoad` and `Run` have different semantic guarantees:

| Property | PreLoad | Run |
|----------|---------|-----|
| Parse scope | Full (all bookmarks) | Chrome: full. Firefox: incremental. Qute: full. |
| URLIndex state | Fresh (PreLoad runs once at boot) | Chrome: rebuilt. Firefox: extended. |
| Buffer state | First population (relatively clean) | Accumulates on top of PreLoad data |
| Clock behavior | Ticks for all new inserts | Ticks only for changes detected by xhsum |

**Impact**: The first browser to finish PreLoad syncs a relatively clean
buffer. Subsequent browsers merge on top, which is correct. But if a browser
fails during PreLoad and restarts, its second buffer (new BufferDB) starts
fresh while the cache still has the old data. This is handled correctly by
the merge logic but may cause transient duplicate processing.

### 12.5 No Tag Attribution in Current Schema

**Problem**: The `gskbookmarks.tags` column stores a flat comma-delimited
string with no attribution to source browser/module. It is impossible to
determine which browser contributed which tag.

**Impact**: Without attribution, it's impossible to implement "remove tag if
all browsers drop it" without changing the schema.

---

## Appendix A: Key Files Reference

| File | Purpose |
|------|---------|
| `internal/database/schema.go` | DB schema definition, versioning, migrations |
| `internal/database/sync.go` | SyncToClock, cache scheduler, BackupToDisk |
| `internal/database/bookmark_tx.go` | UpsertBookmark (merge logic) |
| `internal/database/buffer.go` | NewBuffer, SyncURLIndexToBuffer, SyncTreeToBuffer |
| `internal/database/cache.go` | Cache + L2Cache initialization |
| `internal/database/clock.go` | LamportClock |
| `internal/database/tags.go` | Tags helper type |
| `browsers/chrome/chrome.go` | Chrome parser (full JSON rebuild) |
| `browsers/firefox/firefox.go` | Firefox parser (incremental CTE scan) |
| `browsers/qute/qutebrowser.go` | Qutebrowser parser (text file) |
| `pkg/modules/browser.go` | BrowserConfig, SetupBrowser, RebuildIndex |
| `pkg/modules/common.go` | Module interfaces (Initializer, PreLoader, etc.) |
| `pkg/watch/watcher.go` | WatchDescriptor, WatchLoop |
| `pkg/watch/reducer.go` | ReduceEvents debounce |
| `hooks/defined.go` | Hook registry |
| `hooks/hook.go` | Hook types, execution |
| `pkg/parsing/tags.go` | ParseNodeTags, ParseBkTags (regex tag extraction) |
| `pkg/tree/tree.go` | Node type, getTags(), WalkBuildIndex |
| `bookmark.go` | gosuki.Bookmark struct |
| `pkg/browsers/mozilla/recursive_all_bookmarks.sql` | Firefox full scan CTE |
| `pkg/browsers/mozilla/recursive_modified_bookmarks.sql` | Firefox incremental scan CTE |

## Appendix B: Timing Constants

| Parameter | Default | Configurable Via |
|-----------|---------|-----------------|
| Cache sync debounce | 4 seconds | `database.sync-interval` in config |
| Firefox event reducer interval | 1500ms | Hardcoded (`WatchMinJobInterval`) |
| Firefox time rollback (since last run) | -1 second | Hardcoded in `Run()` |
| WatchLoop tick interval | 1 second | Hardcoded in `WatchLoop` |
| Copy job cleanup age | 1 hour | Hardcoded in `cleanOldCopyJobs()` |

## Appendix C: Tag String Examples

```
Empty tags:              ","
Single tag:              ",work,"
Multiple sorted tags:    ",important,work,"
Sanitized (has comma):   ",tag--1,other,"    (original was "tag,1")
With hook-derived tag:   ",action,notify,ping,"
```
