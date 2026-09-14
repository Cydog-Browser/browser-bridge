# Cydog File System Bridge & Write Integration Guide

This guide details how to integrate and implement the File System Bridge and the new file writing method (`writeFile`) within web frontends running on the Cydog Browser and web platforms using our custom web engine. 

- [Cydog Browser Engine Guidelines](https://demos.cydogbrowser.com/browser-bridge/index.html)
- [Cydog Browser File Bridge Demo](https://demos.cydogbrowser.com/browser-bridge/browser-bridge.html)
- [Cydog Browser CyDrive Importer](https://demos.cydogbrowser.com/browser-bridge/cydrive.html)

## Compatibility Requirements

This integration requires specific minimum browser versions equipped with the native Cydog Bridge and `window.showDirectoryPicker()` polyfill support:

* **Android:** Version 69.0.3 or higher
* **iOS:** Version 32.0 or higher

---

## Architecture Overview

The Cydog Browser injects a JavaScript bridge (`window.__CydogBridge`) and polyfills the standard W3C File System Access API (`window.showDirectoryPicker()`). This allows standard web applications to seamlessly prompt for native directories, iterate through files and subdirectories asynchronously, lazily read file blobs, and write new files directly via the native layer.

---

## Implementation Walkthrough

### 1. Directory Selection & Traversal

To prompt the user for a workspace folder and iterate through its contents, utilize the standard asynchronous directory picker and generator pattern:

```javascript
async function importFolder() {
    try {
        // Triggers the native folder picker via the injected polyfill
        const dirHandle = await window.showDirectoryPicker();
        console.log(`Opened directory: ${dirHandle.name}`);

        // Iterate through entries using the standard async generator
        for await (const entry of dirHandle.values()) {
            if (entry.kind === 'file') {
                console.log(`Found file: ${entry.name} (${entry.size} bytes)`);
                
                // Fetch the actual file blob data on demand
                const fileBlob = await entry.getFile();
                const textContent = await fileBlob.text();
            } else if (entry.kind === 'directory') {
                console.log(`Found subdirectory: ${entry.name}`);
            }
        }
    } catch (err) {
        console.error(`Error or cancellation: ${err.message}`);
    }
}
```

### 2. Implementing the Write File Method

The new `writeFile` method interfaces directly with the Cydog Bridge backend to securely write text files into permitted directories. 

You can initialize and attach the helper method to your application code as follows:

```javascript
// Ensure bridge support and define the writeFile helper if not already present
if (window.__CydogBridge && typeof window.__CydogBridge.writeFile !== 'function') {
    window.__CydogBridge.writeFile = async function(filePath, textData) {
        const domain = window.location.hostname || window.location.protocol || 'local_app';
        return await window.__CydogBridge.request('writeFile', {
            path: filePath,
            text: textData,
            domain: domain
        });
    };
}
```

### 3. Cross-Compatible Writing Execution

To ensure your application remains robust across different environments (supporting both the native Cydog Bridge and standard W3C File System Access API Writable Streams where available), use the following cross-compatible implementation pattern:

```javascript
async function writeDiagnosticFile(dirHandle, fileName, fileContent) {
    const targetFilePath = `${dirHandle._path}/${fileName}`;
    const domain = window.location.hostname || window.location.protocol || 'local_app';

    // Check if Cydog native bridge write is available
    if (window.__CydogBridge && typeof window.__CydogBridge.request === 'function') {
        await window.__CydogBridge.request('writeFile', {
            path: targetFilePath,
            text: fileContent,
            domain: domain
        });
    } 
    // Fallback to standard File System Access API WritableStream if supported
    else if (dirHandle && typeof dirHandle.getFileHandle === 'function') {
        const fileHandle = await dirHandle.getFileHandle(fileName, { create: true });
        const writable = await fileHandle.createWritable();
        await writable.write(fileContent);
        await writable.close();
    } else {
        throw new Error("No compatible file writing mechanism found for this environment.");
    }
}
```

---

## Complete Integration Example

Below is a complete reference implementation demonstrating the initialization check, folder picker trigger, traversal, lazy reading, and file writing sequence:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Cydog Integration Test</title>
</head>
<body>
    <button id="pickBtn">Select Directory & Test</button>
    
    <script>
        async function runIntegration() {
            try {
                // 1. Verify environment support
                if (typeof window.showDirectoryPicker !== 'function') {
                    throw new Error("window.showDirectoryPicker is not supported in this environment.");
                }

                // 2. Prompt user for directory handle
                const dirHandle = await window.showDirectoryPicker();
                console.log(`Acquired directory: ${dirHandle.name}`);

                // 3. Iterate directory contents
                for await (const entry of dirHandle.values()) {
                    if (entry.kind === 'file') {
                        console.log(`File found: ${entry.name}`);
                    }
                }

                // 4. Write test file using the bridge method
                const fileName = `output_${Date.now()}.txt`;
                const targetPath = `${dirHandle._path}/${fileName}`;
                const payload = "Integration test payload data.";

                if (window.__CydogBridge) {
                    await window.__CydogBridge.request('writeFile', {
                        path: targetPath,
                        text: payload,
                        domain: window.location.hostname
                    });
                    console.log("File written successfully via Cydog Bridge.");
                }

            } catch (err) {
                console.error(`Operation failed: ${err.message}`);
            }
        }

        document.getElementById('pickBtn').addEventListener('click', runIntegration);
    </script>
</body>
</html>
```

---

## Troubleshooting

* **Bridge Undefined:** Ensure your application is running inside Cydog Browser version 69.0.3+ on Android or 32.0+ on iOS. Older versions do not expose `window.__CydogBridge`.
* **Permission Errors:** Confirm that the user has selected a valid workspace directory through the native picker prompt (`window.showDirectoryPicker()`) before attempting write operations on restricted paths.