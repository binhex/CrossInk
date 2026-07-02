# Web UI: Recursive Delete & Folder Upload — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use sub-agents (recommended) to implement this plan task-by-task.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add recursive folder deletion (checkbox in delete modal) and folder upload (smart drop zone with webkitdirectory) to the CrossInk web file manager.

**Architecture:** Extend the existing `/delete` endpoint with a `recursive` parameter that delegates to `Storage.removeDir()`. Enhance the upload modal with dual-mode file/folder selection using `webkitdirectory`, directory tree preview, and sequential upload reusing `/mkdir` and `/upload`.

**Tech Stack:** C++17 (PlatformIO/ESP32 Arduino framework), vanilla JavaScript (ES6+), CSS3, HTML5. Build system: PlatformIO + Python-based web asset compiler.

---

## File Map

| File | Role | Change |
|------|------|--------|
| `src/network/CrossPointWebServer.cpp` | HTTP request handlers | `handleDelete()`: accept `recursive` param, call `Storage.removeDir()` |
| `web/pages/files.html` | File manager page template | Delete modal: +recursive checkbox. Upload modal: +folder input, mode buttons, tree preview container |
| `web/pages/files.js` | File manager client logic | `confirmDelete()`: pass recursive flag. New: `setUploadMode()`, `renderFolderTree()`, `uploadFolder()`, smart drop zone detection |
| `web/pages/files.css` | File manager styles | Checkbox warning box, mode buttons, tree preview, dual-button drop zone layout |

---

### Task 1: Backend — Recursive Delete in handleDelete()

**Files:**
- Modify: `src/network/CrossPointWebServer.cpp` (lines ~1090-1115)

- [ ] **Step 1: Read the current handleDelete directory branch**

Read the current code at lines 1089-1116 to understand the exact structure:

```bash
sed -n '1089,1116p' src/network/CrossPointWebServer.cpp
```

Confirm the exact code matches this pattern:

```cpp
    // Decide whether it's a directory or file by opening it
    bool success = false;
    HalFile f = Storage.open(itemPath.c_str());
    if (f && f.isDirectory()) {
      // For folders, ensure empty before removing
      HalFile entry = f.openNextFile();
      if (entry) {
        entry.close();
        f.close();
        failedItems += itemPath + " (folder not empty); ";
        allSuccess = false;
        continue;
      }
      f.close();
      success = Storage.rmdir(itemPath.c_str());
    } else {
      // It's a file (or couldn't open as dir) — remove file
      if (f) f.close();
      success = Storage.remove(itemPath.c_str());
      clearBookCache(itemPath.c_str());
    }
```

- [ ] **Step 2: Replace the directory-deletion branch with recursive-aware logic**

Replace the block starting at `// Decide whether it's a directory or file by opening it` through the closing `}` of the `if (f && f.isDirectory())` branch. The replacement adds a `recursive` flag check before the empty-folder guard:

```cpp
    // Decide whether it's a directory or file by opening it
    bool success = false;
    bool recursive = server->hasArg("recursive") && server->arg("recursive") == "true";
    HalFile f = Storage.open(itemPath.c_str());
    if (f && f.isDirectory()) {
      f.close();
      if (recursive) {
        // Recursive delete — remove directory and all contents.
        // Storage.removeDir() delegates to SDCardManager::removeDir() which
        // performs a recursive tree removal.
        success = Storage.removeDir(itemPath.c_str());
      } else {
        // Non-recursive: ensure folder is empty before removing
        HalFile check = Storage.open(itemPath.c_str());
        if (check) {
          HalFile entry = check.openNextFile();
          if (entry) {
            entry.close();
            check.close();
            failedItems += itemPath + " (folder not empty); ";
            allSuccess = false;
            continue;
          }
          check.close();
        }
        success = Storage.rmdir(itemPath.c_str());
      }
    } else {
      // It's a file (or couldn't open as dir) — remove file
      if (f) f.close();
      success = Storage.remove(itemPath.c_str());
      clearBookCache(itemPath.c_str());
    }
```

**Why re-open:** In the non-recursive path, `f` was already closed by `f.close()` above the branch. We open a fresh `check` handle to call `openNextFile()`. The recursive path doesn't need `openNextFile()` — it just calls `Storage.removeDir()` with the path string.

- [ ] **Step 3: Verify the existing delete behavior is unchanged**

Check that the `recursive` variable defaults to `false` when the parameter is absent:

```bash
echo 'server->hasArg("recursive") with no arg:' && \
  grep -c 'server->hasArg("recursive")' src/network/CrossPointWebServer.cpp
```

The logic `server->hasArg("recursive") && server->arg("recursive") == "true"` means:
- No `recursive` arg → `false` → non-recursive path → existing behavior
- `recursive=false` → `false` → non-recursive path → existing behavior
- `recursive=true` → `true` → recursive path → new behavior

- [ ] **Step 4: Commit**

```bash
git add src/network/CrossPointWebServer.cpp
git commit -m "feat(web): add recursive parameter to /delete endpoint"
```

---

### Task 2: Frontend — Recursive Delete UI

**Files:**
- Modify: `web/pages/files.html` — add checkbox to delete confirmation modal
- Modify: `web/pages/files.js` — detect folders, show/hide checkbox, pass recursive flag
- Modify: `web/pages/files.css` — style the checkbox warning area

- [ ] **Step 1: Add the recursive checkbox to the delete modal (files.html)**

Find the delete confirmation modal (contains `id="deleteModal"`). Insert the checkbox block immediately after the `<div id="deleteItemList">` closing tag and before the button row (`<button class="delete-btn-confirm"`):

```html
      <!-- Recursive delete option — hidden when only files selected -->
      <div id="recursiveDeleteOption" style="display:none; background:rgba(243,156,18,0.13); border:1px solid #f39c12; border-radius:4px; padding:10px; margin-bottom:14px;">
        <label style="display:flex; align-items:center; gap:8px; cursor:pointer; font-size:0.92em;">
          <input type="checkbox" id="recursiveDeleteCheckbox" style="accent-color:#f39c12; width:16px; height:16px;">
          <span>Delete folder contents recursively</span>
        </label>
        <p style="color:#f39c12; font-size:0.82em; margin:6px 0 0 24px;">
          All files and subfolders inside selected folders will be permanently deleted.
        </p>
      </div>
```

Verify it's placed correctly — search for the insertion point:

```bash
grep -n 'deleteItemList\|delete-btn-confirm' web/pages/files.html
```

- [ ] **Step 2: Show/hide the checkbox based on selection (files.js)**

In `openDeleteModalForItems()`, at the end of the function (after building the item list, before `document.getElementById('deleteModal').classList.add('open')`):

```javascript
  // Show recursive delete checkbox only when folders are in the selection
  const hasFolder = items.some(it => it.isFolder);
  const recursiveDiv = document.getElementById('recursiveDeleteOption');
  recursiveDiv.style.display = hasFolder ? 'block' : 'none';
  if (!hasFolder) {
    document.getElementById('recursiveDeleteCheckbox').checked = false;
  }
```

The insertion point is between the list-building loop and the `classList.add('open')` line. Find it:

```bash
grep -n "classList.add('open')" web/pages/files.js | grep -A1 -B3 deleteModal
```

- [ ] **Step 3: Pass the recursive flag when deleting (files.js)**

In `confirmDelete()`, find the line that constructs the fetch body:

```javascript
body: 'path=' + encodeURIComponent(path)
```

There are two approaches — the function currently deletes one path at a time in a loop. Find the loop body and modify it to include the recursive flag when deleting folders:

```javascript
      for (const path of paths) {
        // Check if any selected item is a folder and the recursive checkbox is checked
        const isFolder = deleteItemsGlobal.some(it => it.isFolder && it.path === path);
        const recursive = isFolder && document.getElementById('recursiveDeleteCheckbox').checked;
        const body = 'path=' + encodeURIComponent(path) + (recursive ? '&recursive=true' : '');
        const res = await fetch('/delete', {
          method: 'POST',
          headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
          body: body
        });
```

Note: the check `it.path === path` must account for the leading `/` that's added earlier in the function. Ensure both use the same normalization.

- [ ] **Step 4: Add CSS for the checkbox area (files.css)**

Add at the end of `web/pages/files.css`:

```css
/* Recursive delete checkbox area */
#recursiveDeleteOption label {
  color: #f39c12;
}
#recursiveDeleteOption label input[type="checkbox"] {
  accent-color: #f39c12;
}
```

- [ ] **Step 5: Commit**

```bash
git add web/pages/files.html web/pages/files.js web/pages/files.css
git commit -m "feat(web): add recursive delete checkbox to file manager UI"
```

---

### Task 3: Frontend — Folder Upload (Smart Drop Zone)

**Files:**
- Modify: `web/pages/files.html` — dual buttons in drop zone, folder input, mode indicator, tree preview
- Modify: `web/pages/files.js` — upload mode switching, folder tree rendering, folder upload logic
- Modify: `web/pages/files.css` — mode buttons, tree preview, progress bar styles

- [ ] **Step 1: Add folder input and mode buttons to the upload modal (files.html)**

Find the `<div class="drop-zone" id="dropZone">` section in the upload modal. Replace the single file input paragraph with dual-purpose markup:

**Existing code** (inside `<!-- Upload Modal -->`):
```html
      <div class="drop-zone" id="dropZone">
        <input type="file" id="fileInput" onchange="validateFile()" multiple>
        <p class="drop-zone-hint">⬇ Drop files here — or click to browse</p>
      </div>
```

**Replace with:**
```html
      <div class="drop-zone" id="dropZone">
        <input type="file" id="fileInput" onchange="validateFile()" multiple>
        <input type="file" id="folderInput" webkitdirectory onchange="validateFolder()" style="display:none;">
        <p class="drop-zone-hint" id="dropZoneHint">⬇ Drop files here — or click to browse</p>
        <div class="upload-mode-buttons">
          <button type="button" class="mode-btn active" id="fileModeBtn" onclick="setUploadMode('file')">📄 Select Files</button>
          <button type="button" class="mode-btn" id="folderModeBtn" onclick="setUploadMode('folder')">📁 Select Folder</button>
        </div>
      </div>
```

- [ ] **Step 2: Add mode indicator and folder tree preview (files.html)**

After the drop zone div, before `<div class="picker-columns">`, add:

```html
      <!-- Upload mode indicator -->
      <div class="upload-mode-indicator" id="uploadModeIndicator" style="display:block;">
        Mode: <span id="uploadModeLabel">Files</span> — EPUB optimization available
      </div>
      <!-- Folder tree preview -->
      <div class="folder-tree-preview" id="folderTreePreview" style="display:none;">
        <div class="folder-tree-header">📂 Folder Structure</div>
        <div id="folderTreeContent"></div>
        <div class="folder-tree-count" id="folderTreeCount"></div>
      </div>
```

Insert between the drop zone closing `</div>` and `<div class="picker-columns"`. Verify the insertion point:

```bash
grep -n 'picker-columns\|dropZone' web/pages/files.html | head -10
```

- [ ] **Step 3: Add upload mode switching logic (files.js)**

Add these new functions to `web/pages/files.js`, placed after `closeUploadModal()` and before the EPUB/image functions:

```javascript
  // Upload mode state
  let uploadMode = 'file'; // 'file' or 'folder'

  function setUploadMode(mode) {
    uploadMode = mode;
    const fileModeBtn = document.getElementById('fileModeBtn');
    const folderModeBtn = document.getElementById('folderModeBtn');
    const fileInput = document.getElementById('fileInput');
    const folderInput = document.getElementById('folderInput');
    const dropZoneHint = document.getElementById('dropZoneHint');
    const modeLabel = document.getElementById('uploadModeLabel');
    const modeIndicator = document.getElementById('uploadModeIndicator');
    const folderTreePreview = document.getElementById('folderTreePreview');
    const convertOptions = document.getElementById('convertOptions');
    const uploadBtn = document.getElementById('uploadBtn');

    if (mode === 'file') {
      fileModeBtn.classList.add('active');
      folderModeBtn.classList.remove('active');
      dropZoneHint.textContent = '⬇ Drop files here — or click to browse';
      modeLabel.textContent = 'Files';
      modeIndicator.innerHTML = 'Mode: <span id="uploadModeLabel">Files</span> — EPUB optimization available';
      folderTreePreview.style.display = 'none';
      // Show EPUB optimization options when files are selected
      if (fileInput.files.length > 0) {
        convertOptions.style.display = 'block';
      }
      uploadBtn.textContent = 'Upload';
      uploadBtn.classList.remove('optimize');
      // Clear folder selection
      folderInput.value = '';
    } else {
      folderModeBtn.classList.add('active');
      fileModeBtn.classList.remove('active');
      dropZoneHint.textContent = '📁 Drop a folder here — or click to browse';
      modeLabel.textContent = 'Folder';
      modeIndicator.innerHTML = 'Mode: <span id="uploadModeLabel">Folder</span> — structure will be preserved';
      // Hide EPUB optimization for folder uploads
      convertOptions.style.display = 'none';
      uploadBtn.textContent = 'Upload Folder';
      uploadBtn.classList.add('optimize');
      // Clear file selection
      fileInput.value = '';
      fileInput.classList.remove('has-files');
      clearImagePicker();
    }
  }
```

- [ ] **Step 4: Add folder validation and tree preview (files.js)**

Add after `setUploadMode()`:

```javascript
  function validateFolder() {
    const folderInput = document.getElementById('folderInput');
    const files = folderInput.files;
    const uploadBtn = document.getElementById('uploadBtn');
    const folderTreePreview = document.getElementById('folderTreePreview');

    if (files.length === 0) {
      folderTreePreview.style.display = 'none';
      uploadBtn.disabled = true;
      return;
    }

    renderFolderTree(files);
    folderTreePreview.style.display = 'block';
    uploadBtn.disabled = false;
  }

  function renderFolderTree(files) {
    const treeContent = document.getElementById('folderTreeContent');
    const treeCount = document.getElementById('folderTreeCount');

    // Build a tree structure from webkitRelativePath
    const tree = {};
    let fileCount = 0;
    let dirCount = 0;

    for (const file of files) {
      const parts = file.webkitRelativePath.split('/');
      let node = tree;
      for (let i = 0; i < parts.length; i++) {
        const part = parts[i];
        if (i === parts.length - 1) {
          // It's a file
          if (!node._files) node._files = [];
          node._files.push(part);
          fileCount++;
        } else {
          // It's a directory
          if (!node[part]) {
            node[part] = {};
            dirCount++;
          }
          node = node[part];
        }
      }
    }

    // Render tree as nested HTML
    function renderNode(node, depth) {
      let html = '';
      // Render subdirectories first
      for (const [name, child] of Object.entries(node)) {
        if (name.startsWith('_')) continue;
        const indent = '&nbsp;&nbsp;'.repeat(depth);
        html += `<div style="padding-left:${depth * 16}px;">📁 ${escapeHtml(name)}/</div>`;
        html += renderNode(child, depth + 1);
      }
      // Render files
      if (node._files) {
        for (const fname of node._files) {
          html += `<div style="padding-left:${(depth + 1) * 16}px;">📄 ${escapeHtml(fname)}</div>`;
        }
      }
      return html;
    }

    const rootName = files[0].webkitRelativePath.split('/')[0];
    const rootNode = tree[rootName] || {};
    treeContent.innerHTML = `<div style="font-weight:600;margin-bottom:4px;">📁 ${escapeHtml(rootName)}/</div>` + renderNode(rootNode, 1);
    treeCount.textContent = `${dirCount} folder${dirCount !== 1 ? 's' : ''}, ${fileCount} file${fileCount !== 1 ? 's' : ''}`;
  }
```

- [ ] **Step 5: Modify uploadFile() to handle folder mode (files.js)**

In the existing `uploadFile()` function, add a mode check at the beginning. Find the function definition and add:

```javascript
  async function uploadFile() {
    // If in folder mode, use the folder upload path instead
    if (uploadMode === 'folder') {
      return uploadFolderContents();
    }
    // ... existing file upload code continues unchanged ...
```

- [ ] **Step 6: Add the folder upload function (files.js)**

Add the core folder upload logic after `renderFolderTree()`:

```javascript
  async function uploadFolderContents() {
    const folderInput = document.getElementById('folderInput');
    const files = folderInput.files;
    if (files.length === 0) return;

    const uploadBtn = document.getElementById('uploadBtn');
    const progressContainer = document.getElementById('progress-container');
    const progressFill = document.getElementById('progress-fill');
    const progressText = document.getElementById('progress-text');
    const modalClose = document.getElementById('uploadModalClose');

    uploadBtn.disabled = true;
    modalClose.classList.add('disabled');
    isUploadInProgress = true;
    progressContainer.style.display = 'block';
    progressFill.style.width = '0%';
    progressFill.style.backgroundColor = '#27ae60';

    // Step 1: Collect unique parent directories
    const dirs = new Set();
    for (const file of files) {
      const parts = file.webkitRelativePath.split('/');
      parts.pop(); // Remove filename
      let dirPath = '';
      for (const part of parts) {
        dirPath += (dirPath ? '/' : '') + part;
        dirs.add(dirPath);
      }
    }

    // Step 2: Create all directories
    let dirErrors = 0;
    for (const dir of dirs) {
      try {
        const fullPath = currentPath + (currentPath.endsWith('/') ? '' : '/') + dir;
        const res = await fetch('/mkdir', {
          method: 'POST',
          headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
          body: 'path=' + encodeURIComponent(fullPath)
        });
        if (!res.ok) {
          console.warn('Failed to create directory:', dir);
          dirErrors++;
        }
      } catch (e) {
        console.warn('Network error creating directory:', dir, e);
        dirErrors++;
      }
    }

    // Step 3: Upload each file
    let uploaded = 0;
    let failed = 0;
    const total = files.length;

    for (const file of files) {
      const fullPath = currentPath + (currentPath.endsWith('/') ? '' : '/') + file.webkitRelativePath;
      const pct = Math.round((uploaded / total) * 100);
      progressFill.style.width = pct + '%';
      progressText.textContent = `Uploading ${uploaded + 1}/${total} — ${file.webkitRelativePath}`;

      try {
        const formData = new FormData();
        formData.append('path', fullPath);
        formData.append('file', file, file.name);

        const res = await fetch('/upload', {
          method: 'POST',
          body: formData
        });

        if (res.ok) {
          uploaded++;
        } else {
          failed++;
          console.warn('Failed to upload:', file.webkitRelativePath, await res.text());
        }
      } catch (e) {
        failed++;
        console.warn('Network error uploading:', file.webkitRelativePath, e);
      }
    }

    // Step 4: Completion
    isUploadInProgress = false;
    modalClose.classList.remove('disabled');
    uploadBtn.disabled = false;
    progressFill.style.width = '100%';

    if (failed > 0 || dirErrors > 0) {
      progressFill.style.backgroundColor = '#f39c12';
      progressText.textContent = `Done — ${uploaded} uploaded, ${failed} failed` + (dirErrors > 0 ? `, ${dirErrors} dir errors` : '');
    } else {
      progressFill.style.backgroundColor = '#27ae60';
      progressText.textContent = `Done — ${uploaded} file${uploaded !== 1 ? 's' : ''} uploaded successfully`;
    }

    // Close and refresh after a short delay
    setTimeout(() => {
      closeUploadModal();
      window.location.reload();
    }, 1500);
  }
```

- [ ] **Step 7: Wire folder input click to mode button (files.js)**

In the drop zone click handler (the `setupFileInputListener` IIFE), modify the click event to respect the current mode. Find the existing `dropZone.addEventListener('click', ...)` and add mode-aware logic:

The existing handler triggers `fileInput.click()`. Replace that single line:

```javascript
      dropZone.addEventListener('click', function(e) {
        if (uploadBusy()) return;
        // Don't capture clicks on the mode buttons themselves
        if (e.target.closest('.mode-btn')) return;
        if (uploadMode === 'folder') {
          document.getElementById('folderInput').click();
        } else {
          fileInput.click();
        }
      });
```

Also update the drag-and-drop handler to detect folders. Find the `drop` event handler and add folder detection:

In the existing drop handler (after `validateFile()` call), add logic before `validateFile()`:

```javascript
      dropZone.addEventListener('drop', function(e) {
        e.preventDefault();
        dragDepth = 0;
        dropZone.classList.remove('dragover');
        if (uploadBusy()) return;

        const dropped = e.dataTransfer && e.dataTransfer.files;
        if (!dropped || dropped.length === 0) return;

        // Detect if a folder was dropped using DataTransferItem API
        let isFolderDrop = false;
        if (e.dataTransfer.items && e.dataTransfer.items.length > 0) {
          const firstItem = e.dataTransfer.items[0];
          if (firstItem.webkitGetAsEntry) {
            const entry = firstItem.webkitGetAsEntry();
            isFolderDrop = entry && entry.isDirectory;
          }
        }

        if (isFolderDrop) {
          // Switch to folder mode and assign to folder input
          setUploadMode('folder');
          const folderInput = document.getElementById('folderInput');
          const dt = new DataTransfer();
          for (const file of dropped) dt.items.add(file);
          folderInput.files = dt.files;
          validateFolder();
        } else {
          setUploadMode('file');
          const dt = new DataTransfer();
          for (const file of dropped) dt.items.add(file);
          fileInput.files = dt.files;
          validateFile();
        }
      });
```

- [ ] **Step 8: Reset folder state when closing upload modal**

In `closeUploadModal()`, add folder mode reset:

```javascript
    // Reset folder upload state
    uploadMode = 'file';
    document.getElementById('folderInput').value = '';
    document.getElementById('folderTreePreview').style.display = 'none';
    document.getElementById('fileModeBtn').classList.add('active');
    document.getElementById('folderModeBtn').classList.remove('active');
    document.getElementById('dropZoneHint').textContent = '⬇ Drop files here — or click to browse';
    document.getElementById('uploadModeIndicator').innerHTML = 'Mode: <span id="uploadModeLabel">Files</span> — EPUB optimization available';
```

Insert these lines in `closeUploadModal()`, before the function returns or at the end of the reset block.

- [ ] **Step 9: Add CSS for folder upload UI (files.css)**

Add at the end of `web/pages/files.css`:

```css
/* Upload mode buttons inside drop zone */
.upload-mode-buttons {
  display: flex;
  gap: 8px;
  justify-content: center;
  margin-top: 12px;
}
.mode-btn {
  padding: 6px 16px;
  border: 1px solid #555;
  border-radius: 4px;
  background: #2a2a2a;
  color: #95a5a6;
  cursor: pointer;
  font-size: 0.85em;
  transition: all 0.2s;
}
.mode-btn:hover {
  border-color: #777;
  color: #ccc;
}
.mode-btn.active {
  border-color: #3498db;
  background: #3498db22;
  color: #3498db;
  font-weight: 600;
}

/* Upload mode indicator */
.upload-mode-indicator {
  font-size: 0.82em;
  color: #95a5a6;
  text-align: center;
  margin-bottom: 12px;
  padding: 6px;
  background: #1a1a1a;
  border-radius: 4px;
}

/* Folder tree preview */
.folder-tree-preview {
  background: #1a1a1a;
  border-radius: 6px;
  padding: 12px;
  margin-bottom: 16px;
  max-height: 200px;
  overflow: auto;
  font-family: monospace;
  font-size: 0.82em;
}
.folder-tree-header {
  font-weight: 600;
  margin-bottom: 8px;
  color: #3498db;
}
#folderTreeContent {
  color: #ccc;
  line-height: 1.8;
}
.folder-tree-count {
  margin-top: 8px;
  padding-top: 8px;
  border-top: 1px solid #333;
  font-size: 0.9em;
  color: #777;
}
```

- [ ] **Step 10: Commit**

```bash
git add web/pages/files.html web/pages/files.js web/pages/files.css
git commit -m "feat(web): add folder upload with smart drop zone to file manager"
```

---

### Task 4: Regenerate Web Headers & Build Verification

**Files:**
- Regenerate: `src/network/html/*.generated.h` (auto-generated by build script)
- Verify: PlatformIO build output

- [ ] **Step 1: Run the web asset compiler**

```bash
python3 scripts/build_web.py
```

Expected output: lists each page with original and compressed sizes, ending without errors.

- [ ] **Step 2: Verify generated headers are updated**

```bash
ls -la src/network/html/FilesPageHtml.generated.h
stat -c '%y' src/network/html/FilesPageHtml.generated.h
```

The modification timestamp should be recent (within the last minute).

- [ ] **Step 3: Build the firmware (tiny variant)**

```bash
pio run -e tiny
```

Expected: successful build with no errors. Pay attention to any warnings about the modified `CrossPointWebServer.cpp`.

Watch for the output lines like:
```
RAM:   [======    ]  60.2% (used 196908 bytes from 327680 bytes)
Flash: [======    ]  58.3% (used 1875949 bytes from 3219200 bytes)
```

Confirm RAM and Flash are not overflowing (both under 95%).

- [ ] **Step 4: Commit generated headers**

```bash
git add src/network/html/*.generated.h
git commit -m "chore: regenerate web headers after UI changes"
```

---

### Task 5: Firmware Build & Flash Guide

**Files:**
- Create: `docs/building-firmware.md` (user-facing guide)

- [ ] **Step 1: Write the firmware build guide**

Create `docs/building-firmware.md`:

```markdown
# Building and Flashing Firmware

## Option A: Build Locally

### Prerequisites
- Python 3.14+
- PlatformIO Core (`pip install platformio`)
- USB-C cable

### Build
```bash
# Clone the repo
git clone https://github.com/uxjulia/CrossInk.git
cd CrossInk

# Build the tiny variant (recommended for most users)
pio run -e tiny
```

The firmware binary is at `.pio/build/tiny/firmware-tiny.bin`.

### Flash
1. Connect your Xteink X4/X3 via USB-C
2. Run: `pio run -e tiny --target upload`
3. Wait for the upload to complete

## Option B: GitHub Actions (CI Build)

1. Push your changes to the `main` branch
2. Go to Actions tab → select the latest CI run
3. Download the `firmware-tiny` artifact
4. Flash using the [web installer](https://crosspoint-reader.github.io/crosspoint-reader/flash) or `esptool.py`

## Option C: Release Build (workflow_dispatch)

1. Go to Actions → "Compile Release"
2. Click "Run workflow" → enter version (e.g. `1.4.0-rc1`)
3. Wait for build to complete
4. Download the firmware binary from the draft release
```

- [ ] **Step 2: Commit**

```bash
git add docs/building-firmware.md
git commit -m "docs: add firmware build and flash guide"
```

---

## Task Summary

| Task | Description | Files | Commit Message |
|------|-------------|-------|----------------|
| 1 | Backend recursive delete | `CrossPointWebServer.cpp` | `feat(web): add recursive parameter to /delete endpoint` |
| 2 | Frontend recursive delete UI | `files.html`, `files.js`, `files.css` | `feat(web): add recursive delete checkbox to file manager UI` |
| 3 | Frontend folder upload | `files.html`, `files.js`, `files.css` | `feat(web): add folder upload with smart drop zone to file manager` |
| 4 | Regenerate headers + build | `*.generated.h` | `chore: regenerate web headers after UI changes` |
| 5 | Build/flash guide | `docs/building-firmware.md` | `docs: add firmware build and flash guide` |
