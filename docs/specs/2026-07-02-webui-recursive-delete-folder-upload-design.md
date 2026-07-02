# Web UI: Recursive Delete & Folder Upload

**Date:** 2026-07-02
**Status:** Approved
**Files affected:** 4 (CrossPointWebServer.cpp, files.html, files.js, files.css)

## Overview

Add two enhancements to the CrossInk web file manager (`/files`):

1. **Recursive folder delete** — delete a folder and all its contents (files and subfolders)
   with a checkbox confirmation in the existing delete modal.
2. **Folder upload** — select and upload an entire folder, preserving its directory
   structure on the SD card, via a smart drop zone integrated into the existing upload modal.

All changes are backward compatible and stay within existing endpoints — no new HTTP routes.

---

## Feature 1: Recursive Delete

### Frontend (files.html, files.js, files.css)

**Delete confirmation modal** gains a conditional checkbox:

- A new checkbox labeled "Delete folder contents recursively" appears inside the
  delete modal whenever at least one selected item is a directory.
- The checkbox is accompanied by warning text:
  "All files and subfolders inside selected folders will be permanently deleted."
- The checkbox is hidden when only files are selected (no change from current behavior).
- When checked, `confirmDelete()` sends `recursive=true` alongside `path` in the
  POST body to `/delete`.

**CSS additions:**
- Warning box styling for the checkbox area (amber border, subtle background).
- Checkbox label alignment to match the modal's existing typography.

### Backend (CrossPointWebServer.cpp)

**`handleDelete()` modification:**

- Accepts a new optional parameter `recursive` (values: `"true"` or absent/false).
- When processing a directory and `recursive == "true"`:
  - Calls `Storage.removeDir(itemPath.c_str())` instead of the empty-folder check
    followed by `Storage.rmdir()`.
  - **Assumption:** `Storage.removeDir()` is the recursive variant (HalStorage exposes
    both `rmdir` for empty dirs and `removeDir` as a separate method). If the SDK's
    `SDCardManager::removeDir` is not recursive, implement a recursive walk using
    `HalFile::openNextFile()` inside the handler.
  - `Storage.removeDir()` delegates to `SDCardManager::removeDir()`, which performs
    a recursive removal of the directory and all its contents.
- When `recursive` is absent or `"false"`: existing behaviour is preserved (empty-folder
  check, rejection of non-empty folders).
- Protected path check (`isProtectedPath`) still applies regardless of recursive flag.
- EPUB cache clearing (`clearBookCache`) is called for each EPUB file encountered
  during recursive deletion.

**Behaviour matrix:**

| `recursive` | Item type | Non-empty? | Action |
|-------------|-----------|------------|--------|
| false/absent | file | — | `Storage.remove()` + cache clear |
| false/absent | dir | no | `Storage.rmdir()` |
| false/absent | dir | yes | reject: "folder not empty" |
| true | file | — | `Storage.remove()` + cache clear |
| true | dir | any | `Storage.removeDir()` (recursive) |

---

## Feature 2: Folder Upload (Smart Drop Zone)

### Frontend (files.html, files.js, files.css)

**Upload modal** is enhanced with a smart drop zone that supports both files and folders:

**New drop zone layout:**
- The existing drag-and-drop area gains two explicit buttons:
  - "Select Files" — opens `<input type="file" multiple>` (current behavior).
  - "Select Folder" — opens `<input type="file" webkitdirectory>` (new).
- A mode indicator text below the drop zone shows the active mode:
  - "Mode: Files — EPUB optimization available" for file mode.
  - "Mode: Folder — structure will be preserved" for folder mode.
- Drag-and-drop auto-detection: if dropped items include a directory
  (`DataTransferItem.webkitGetAsEntry().isDirectory`), the mode switches to folder.

**Folder mode behavior:**
- EPUB optimization UI (convert checkbox, image picker, advanced options) is
  hidden in folder mode — these are not applicable to batch folder uploads.
- After folder selection, a tree preview is rendered showing the folder structure
  (directories with `📁`, files with `📄`, indented by depth).
- The "Upload" button changes to "Upload Folder" with distinct styling.

**Upload flow (client-side):**

1. User selects a folder via `webkitdirectory` input.
2. JavaScript iterates the selected `FileList`, reading `file.webkitRelativePath`
   from each entry (e.g. `"MyBooks/SciFi/chapter1.epub"`).
3. Unique parent directories are collected (e.g. `"MyBooks"`, `"MyBooks/SciFi"`).
4. All directories are created first via sequential `POST /mkdir` calls
   (the `/mkdir` endpoint is idempotent — it no-ops if the directory already exists).
5. Files are uploaded sequentially via `POST /upload` with FormData containing
   the destination path (`currentPath + "/" + file.webkitRelativePath`).
6. A progress bar shows "Uploading X/Y files — filename.ext".
7. On completion, the page refreshes to show the new files.

**Edge cases:**
- **Empty subdirectories:** `webkitdirectory` does not report empty directories.
  Empty folders in the source are not recreated. This is an accepted limitation
  of the web API.
- **Existing files:** Overwritten without confirmation (matches current upload
  behaviour for single files).
- **Very large folders:** Sequential upload avoids overwhelming the ESP32's
  HTTP server. The progress bar provides visibility.

### Backend

No backend changes are required for folder upload. The existing endpoints are reused:

- `POST /mkdir` — creates directories (already idempotent via `Storage.ensureDirectoryExists`).
- `POST /upload` — accepts a file and destination path (existing behaviour).

---

## Error Handling

### Recursive Delete

- Protected paths (`/crosspoint`, `/XTCache`, etc.) are rejected before any
  deletion attempt, regardless of the recursive flag.
- If `Storage.removeDir()` fails, the item is added to `failedItems` and
  `allSuccess` is set to false, consistent with current error reporting.
- The root path `/` cannot be deleted.

### Folder Upload

- Network errors during upload: the progress bar stops, the current file name
  remains displayed, and subsequent files are not attempted. The modal stays
  open so the user can see what failed.
- `/mkdir` failures: logged to console, but upload continues — the file upload
  will fail with a clearer error if the parent directory doesn't exist.
- Individual file upload failures: the file is skipped and counted as failed.
  The progress bar updates to reflect completion. A summary of failures is
  shown at the end.

---

## Backward Compatibility

- `/delete` without `recursive` parameter: identical behavior to current.
- Upload modal without folder selection: identical behavior to current
  (file mode is the default).
- No new HTTP endpoints are added.
- No changes to the `HalStorage` or `SDCardManager` APIs.
- The build script (`scripts/build_web.py`) is not modified — new HTML/CSS/JS
  is compiled into the same generated headers.

---

## Files Changed

| File | Changes |
|------|---------|
| `web/pages/files.html` | Delete modal: recursive checkbox. Upload modal: "Select Folder" button, mode indicator, tree preview container. |
| `web/pages/files.js` | `confirmDelete()`: passes `recursive` flag. New functions: `detectUploadMode()`, `renderFolderTree()`, `uploadFolder()`. Mode switching logic for smart drop zone. |
| `web/pages/files.css` | Checkbox warning box, mode indicator, tree preview, dual-button drop zone layout. |
| `src/network/CrossPointWebServer.cpp` | `handleDelete()`: accepts `recursive` parameter, delegates to `Storage.removeDir()` when set. |

---

## Non-Goals (Out of Scope)

- Preserving empty folders during upload (webkitdirectory API limitation).
- Upload conflict resolution UI (overwrites, matching current behavior).
- Progress indicator for recursive delete (SD card operations are near-instant).
- Batch EPUB optimization for folder uploads.
- New HTTP endpoints or routes.
- Changes to the device-side file browser UI (only the web UI is affected).
