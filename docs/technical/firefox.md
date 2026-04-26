# Firefox Bookmarks Parsing Guide

## File Locking & VFS Strategy

### Key Challenges
- Firefox uses SQLite file locking via VFS (Virtual File System)
- Access restrictions have evolved over versions
- No official documentation for `storage.multiProcessAccess.enabled` preference

### Recommended Strategy
1. **Primary Approach**: 
   - Attempt to configure Firefox for non-exclusive locks
   - Use `storage.multiProcessAccess.enabled` preference

2. **Fallback Approach**:
   - Copy `places.sqlite*` files to temporary directory
   - Parse bookmarks from the copy

### Implementation Notes
- Monitor GitHub commits for VFS changes: 
  - [A543F35](https://github.com/mozilla/gecko-dev/commit/a543f35d4be483b19446304f52e4781d7a4a0a2f)
  - [14784DC](https://github.com/mozilla/gecko-dev/commit/14784dc42d7994ea9fc8ff279e5f685501289d60)

## Database Structure

### Key Tables
#### `moz_bookmarks`
| Column         | Type    | Description                          |
|----------------|---------|--------------------------------------|
| `id`           | Integer | Unique identifier                    |
| `type`         | Integer | 1=URL, 2=Folder/Tag                  |
| `fk`           | Integer | References `moz_places.id`           |
| `parent`       | Integer | Folder ID (see root folder list)     |
| `position`     | Integer | Order within parent                  |

### Root Folders
| ID   | Name                | Description                     |
|------|---------------------|---------------------------------|
| 1    | Root                | Base folder                     |
| 2    | Bookmarks Menu      | Main bookmarks menu             |
| 3    | Bookmarks Toolbar   | Toolbar bookmarks               |
| 4    | Tags                | Tag folder                      |
| 5    | Unfiled Bookmarks   | Default folder                  |
| 6    | Mobile Bookmarks    | Mobile-specific bookmarks       |

### Bookmark Relationships
1. **Basic Bookmark**:
   ```
   [Bookmark] -> [moz_places]
   ```

2. **Tagged Bookmark**:
   ```
   [Tag] (type=2) 
   └── [Bookmark] (type=1) 
       └── [Tag-Bookmark Link] (type=1)
   ```

3. **Foldered Bookmark**:
   ```
   [Bookmark] (type=1) 
   └── [Folder] (parent=folder_id)
   ```

4. **Tagged & Foldered**:
   ```
   [Tag] (type=2) 
   └── [Bookmark] (type=1) 
       └── [Folder] (parent=folder_id)
   ```

## Parsing Algorithm

1. **Initialization**
   - Create root node
   - Identify target root folders (3, 4, 6)

2. **Recursive Parsing**
   - Traverse from root folders
   - Use `parent` field to build hierarchy
   - Distinguish between:
     - Type 1: URLs
     - Type 2: Folders/Tags

3. **Special Cases**
   - Handle mobile bookmarks via `browser.bookmarks.showMobileBookmarks` preference
   - Process tags from `moz_bookmarks.parent = 4`

## Timestamp Handling
- Firefox stores timestamps in **milliseconds**
- SQLite `strftime('%s', ...)` returns **seconds**
- Convert using: `timestamp / 1000`

## Query Examples

### Find Duplicates
```sql
SELECT metadata, url 
FROM bookmarks 
JOIN (
    SELECT url, metadata, COUNT(url) AS x
    FROM bookmarks 
    GROUP BY url 
    HAVING x > 1
) USING (metadata, url)
LIMIT 10;
```

### Find Tagged Bookmarks
```sql
SELECT moz_places.id, moz_places.url, moz_places.title, moz_bookmarks.parent    
FROM moz_places    
LEFT OUTER JOIN moz_bookmarks    
ON moz_places.id = moz_bookmarks.fk    
WHERE moz_bookmarks.parent = ?;  -- Replace with tag folder ID
```

## Technical Notes

### WAL File Handling
- SQLite WAL (Write-Ahead Logging) may delay changes
- Consider using tools like [walitean](https://github.com/n0fate/walitean) for analysis
- Implement debouncing for frequent `WRITE` events

### ID Ranges
- Mozilla reserves IDs 1-12 for system folders
- User-defined bookmarks start at ID 13+

### File Locking Workarounds
- Check `storage.multiProcessAccess.enabled` preference
- Implement file copying strategy as fallback
- Monitor Firefox source changes for VFS updates
