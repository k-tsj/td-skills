---
name: agent-experimental
description: Build advanced LLM agents with experimental_knowledge_base and experimental_artifact using `tdx agent pull/push`. Covers JavaScript execution in llm-api platform, custom tools, interactive artifacts, Sandpack rendering, and Univer spreadsheets. Use for TD AI agents requiring custom logic or dynamic visualizations.
---

# tdx Agent Experimental - Advanced Agent Capabilities

Build LLM agents with custom JavaScript execution and interactive artifacts using experimental_knowledge_base and experimental_artifact features.

## Overview

**experimental_knowledge_base** と **experimental_artifact** enable JavaScript code execution within the llm-api platform:

- **experimental_knowledge_base**: Custom JavaScript functions that agents can call as tools
- **experimental_artifact**: Reusable JavaScript templates and files rendered in chat

Manage these resources using `tdx agent pull/push` with local directory structures.

## Key Commands

```bash
# Pull project resources (includes experimental features)
tdx agent pull "project-name"

# Push experimental resources to remote
tdx agent push agents/project-name
tdx agent push agents/project-name --dry-run  # Preview changes

# Typical workflow
# 1. Create experimental_artifacts/ and experimental_knowledge_bases/ directories
# 2. Define YAML config, JavaScript handlers, and files
# 3. Push to remote with tdx agent push
# 4. Update agent.yml to reference experimental resources
```

## Folder Structure

```
agents/{project-name}/
├── experimental_knowledge_bases/
│   └── {kb-name}/
│       ├── {kb-name}.yml
│       └── functions/
│           └── {function-name}/
│               ├── code.js
│               └── json_schema.json
└── experimental_artifacts/
    └── {artifact-name}/
        ├── {artifact-name}.yml
        ├── code.js
        └── files/
            └── {file-name}
```

## experimental_knowledge_base

### Structure

```yaml
# hello_world_kb.yml
name: hello_world_kb
timeout_seconds: 30
variables:
  - name: ARTIFACT_ID
    value: YOUR_ARTIFACT_UUID
```

```javascript
// functions/create_hello/code.js
async function handler(args, ctx) {
  const artifact = await ctx.createArtifact({
    type: "EXPERIMENTAL",
    data: "{}",
    experimentalArtifactId: td.env.ARTIFACT_ID
  });
  await ctx.putChatFile(artifact.pathPrefix + "hello.txt", "Hello, world");
  return "Artifact created with ID: " + artifact.id;
}
```

```json
// functions/create_hello/json_schema.json
{
  "type": "object",
  "properties": {},
  "required": []
}
```

### JavaScript API

**td object (global)** - External resources:

| Method | Description | Returns |
|--------|-------------|---------|
| `td.env` | Environment variables from KB variables | Object |
| `td.fetch(url, options)` | External HTTP requests | Promise\<Response\> |
| `td.internalFetch(service, path, options)` | Internal TD API calls | Promise\<Response\> |

**Usage examples:**

```javascript
// Access environment variables
const env = td.env;  // { GREETING: "Hello from Knowledge Base!", ... }
const projectId = td.env.PROJECT_ID;

// External HTTP request
const response = await td.fetch("https://api.example.com/data", {
  method: "POST",
  headers: {
    "Content-Type": "application/json"
  },
  body: JSON.stringify({ key: "value" })
});
const data = await response.json();  // or response.text()

// Internal Treasure Data API call
const response = await td.internalFetch("api", "/api/projects", {
  method: "GET",
  headers: {
    "X-Custom-Header": "value"
  }
});
const projects = await response.json();

// Response object properties
response.status      // HTTP status code (200, 404, etc.)
response.statusText  // Status text ("OK", "Not Found", etc.)
response.ok          // true if 200-299
response.headers     // Headers object
await response.text()  // Get response body as text
await response.json()  // Parse response body as JSON

// Headers object methods
response.headers.get("content-type")  // Get header value
response.headers.has("content-type")  // Check if header exists
response.headers.keys()               // Iterator of header names
response.headers.values()             // Iterator of header values
response.headers.entries()            // Iterator of [name, value] pairs
```

**ctx object (handler arg 2)** - Chat and artifact operations:

| Method | Description | Returns |
|--------|-------------|---------|
| `createArtifact(options)` | Create artifact | `{id, pathPrefix}` |
| `listChatFiles(path)` | List chat files | `[{path}, ...]` |
| `getChatFile(path)` | Get file as Response object | Response or null |
| `readChatFile(path)` | Read file as text | string or null |
| `putChatFile(path, data, options)` | Create/update file | null |
| `deleteChatFile(path)` | Delete file | null |
| `copyChatFile(srcPath, dstPath)` | Copy file | null |

**Usage examples:**

```javascript
// Create artifact
const artifact = await ctx.createArtifact({
  type: "EXPERIMENTAL",
  data: JSON.stringify({ message: "Hello!" }),
  experimentalArtifactId: td.env.ARTIFACT_ID  // Optional: reference artifact ID from KB variables
});
// Returns: { id: "artifact-uuid", pathPrefix: "/path/to/artifact/" }

// Best practice:
// Generally, artifacts linked to a KB are fixed, so define the artifact ID
// in the KB's variables attribute and reference it as td.env.ARTIFACT_ID.
// Example: variables: [{ name: "ARTIFACT_ID", value: "artifact-uuid-here" }]

// List chat files
const files = await ctx.listChatFiles("/path/to/dir");
// Returns: [{ path: "/path/to/dir/file1.txt" }, { path: "/path/to/dir/file2.txt" }]

// Get chat file (Response object or null)
const response = await ctx.getChatFile("/path/to/file.txt");
if (response) {
  const content = await response.text();
  console.log(response.headers.get("content-type"));
}

// Read chat file (text only)
const content = await ctx.readChatFile("/path/to/file.txt");
// Returns: file contents (string) or null

// Create/update chat file
await ctx.putChatFile("/path/to/file.txt", "Hello, world", {
  contentType: "text/plain"  // Optional
});
// Returns: null

// Delete chat file
await ctx.deleteChatFile("/path/to/file.txt");
// Returns: null

// Copy chat file
await ctx.copyChatFile("/source.txt", "/dest.txt");
// Returns: null
```

### Using in agent.yml

```yaml
tools:
  - type: experimental_knowledge_base
    target: '@ref(type: "experimental_knowledge_base", name: "hello_world_kb")'
    target_function: create_hello
    function_name: create_hello
    function_description: Creates a hello world artifact
```

## experimental_artifact

### Structure

```yaml
# hello_world_renderer.yml
name: hello_world_renderer
```

```javascript
// code.js
async function handler(config, artifact) {
  const helloContent = config.files["hello.txt"]?.content || "Hello, world";
  const styleContent = config.files["style.css"]?.content || "";
  return {
    files: {
      "/index.html": {
        content: "<style>" + styleContent + "</style><h1>" + helloContent + "</h1>"
      }
    },
    template: "vanilla"
  };
}
```

```
files/hello.txt
files/style.css
```

### Handler Function

Artifact handlers receive two arguments:

```javascript
async function handler(config, artifact) {
  // config.files: Static files defined in ExperimentalArtifact
  //   - Shared across all artifact instances
  //   - Format: { "filename": { content: "..." }, ... }

  // artifact.files: Dynamic files from ctx.putChatFile()
  //   - Unique per artifact instance
  //   - Format: { "filename": { content: "..." }, ... }

  // artifact.content: Data from createArtifact({ data: "..." })

  return {
    files: { "/index.html": { content: "..." } },
    template: "react"  // vanilla, react, vue, angular
  };
}
```

## Complete Example: Spreadsheet Generation and Display

This example demonstrates creating an interactive spreadsheet using the Univer library. The Knowledge Base creates spreadsheet data, and the Experimental Artifact renders it in the browser.

### Overview

Implemented functionality:
- **create_spreadsheet**: Create a new spreadsheet workbook
- **put_spreadsheet_sheet**: Add sheets (cell data, styles, merge info) to workbook
- **Univer Artifact**: Display the spreadsheet in browser

### Key Takeaways

Essential patterns from this Complete Example:

Artifact Design:
- Separate static files into the `files/` directory and reference them via `config.files[filename].content` rather than implementing everything in the handler
- Declare dependencies in `package.json` (always include react and react-dom)
- Keep `code.js` handler focused on dynamic processing while placing static components (App.js, index.js, index.html) under `files/`
- Clearly separate responsibilities between Knowledge Base and Artifact for reusability:
  - Knowledge Base: Data creation and manipulation logic
  - Artifact: Data display and rendering

Knowledge Base Design:
- Store Artifact IDs in KB variables and reference via `td.env.EXPERIMENTAL_ARTIFACT_ID`
- Carefully design data flow between functions:
  - Creation functions return paths in their response (e.g., `create_spreadsheet` returns `artifact.pathPrefix`)
  - Manipulation functions accept paths as arguments (e.g., `put_spreadsheet_sheet` takes path parameter)

### Step 1: Create Experimental Artifact

Create an Artifact that displays spreadsheets using the Univer library. This Artifact loads workbook data created by the Knowledge Base and renders it in the browser.

#### Directory Structure

```
agents/
└── my-project/
    └── experimental_artifacts/
        └── univer_spreadsheet/
            ├── univer_spreadsheet.yml
            ├── code.js
            └── files/
                ├── App.js
                ├── index.js
                ├── index.html
                └── package.json
```

#### univer_spreadsheet.yml

```yaml
name: univer_spreadsheet
```

#### code.js

```javascript
async function handler(config, artifact) {
  const metadata = JSON.parse(artifact.content);

  const workbookData = {
    id: 'workbook',
    name: metadata.title,
    sheets: { }
  };

  // Load all sheets from artifact.files (sheet-1.json, sheet-2.json, ...)
  for (const [name, file] of Object.entries(artifact.files)) {
    const m = name.match(/^sheet-([1-9][0-9]*)\.json$/);
    if (m) {
      const sheetId = m[1];
      const worksheetData = JSON.parse(file.content);
      const maxRowIndex = Math.max(...Object.keys(worksheetData['cellData']).map(i => parseInt(i))) || 0;
      const maxColumnIndex = Math.max(...Object.values(worksheetData['cellData']).map(cells => Math.max(...Object.keys(cells).map(i => parseInt(i))))) || 0;

      workbookData.sheets[sheetId] = {
        id: sheetId,
        rowCount: maxRowIndex + 1,
        columnCount: maxColumnIndex + 1,
        ...worksheetData,
      };
    }
  }

  workbookData['sheetOrder'] = Object.keys(workbookData.sheets).map(i => parseInt(i)).sort().map(i => i.toString());

  const files = {
    '/App.js': config.files['App.js'].content,
    '/index.js': config.files['index.js'].content,
    '/index.html': config.files['index.html'].content,
    '/package.json': config.files['package.json'].content,
    '/workbook.js': `export const WORKBOOK = ${JSON.stringify(workbookData)};`,
  };

  return { files, template: "react" };
}
```

#### files/App.js

```javascript
import React, { useEffect, useRef } from 'react';
import { UniverSheetsCorePreset } from '@univerjs/preset-sheets-core'
import UniverPresetSheetsCoreEnUS from '@univerjs/preset-sheets-core/locales/en-US'
import { createUniver, LocaleType, mergeLocales } from '@univerjs/presets'
import '@univerjs/preset-sheets-core/lib/index.css'
import { WORKBOOK } from './workbook';

export default function App() {
  const containerRef = useRef(null);
  const workbook = WORKBOOK;

  useEffect(() => {
    if (containerRef.current) {
    const { univerAPI } = createUniver({
      locale: LocaleType.EN_US,
      locales: {
        [LocaleType.EN_US]: mergeLocales(
          UniverPresetSheetsCoreEnUS,
        ),
      },
      presets: [
        UniverSheetsCorePreset({
          container: containerRef.current,
        }),
      ],
    });
    univerAPI.createUniverSheet(workbook);
    return () => { univerAPI.dispose() };
    }
  }, [WORKBOOK, containerRef.current]);

  return (
    <div ref={containerRef} style={{width: "100%", height: "80vh"}}/>
  );
}
```

#### files/index.js

```javascript
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(<App />);
```

#### files/index.html

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Univer Spreadsheet</title>
</head>
<body>
  <div id="root"></div>
</body>
</html>
```

#### files/package.json

```json
{
    "main": "index.js",
    "dependencies": {
        "react": "^19.0.0",
        "react-dom": "^19.0.0",
        "@univerjs/preset-sheets-core": "^0.10.6",
        "@univerjs/presets": "^0.10.6"
    }
}
```

#### Push Command

```bash
tdx agent push agents/my-project
```

**IMPORTANT**: After pushing, you MUST ask the user to provide the Artifact ID. The `tdx` CLI does not display the Artifact ID, so you cannot retrieve it yourself. Ask the user to check the web interface and provide the ID.

### Step 2: Create Knowledge Base

Create a Knowledge Base with two functions for creating and editing spreadsheets.

#### Directory Structure

```
agents/
└── my-project/
    └── experimental_knowledge_bases/
        └── spreadsheet_tools/
            ├── spreadsheet_tools.yml
            └── functions/
                ├── create_spreadsheet/
                │   ├── code.js
                │   └── json_schema.json
                └── put_spreadsheet_sheet/
                    ├── code.js
                    └── json_schema.json
```

#### spreadsheet_tools.yml

```yaml
name: spreadsheet_tools
timeout_seconds: 30
variables:
  - name: EXPERIMENTAL_ARTIFACT_ID
    value: YOUR_ARTIFACT_ID_FROM_STEP1
```

**Note:** Replace `YOUR_ARTIFACT_ID_FROM_STEP1` with the Artifact ID from Step 1.

#### functions/create_spreadsheet/code.js

```javascript
async function handler(args, ctx) {
  const artifact = await ctx.createArtifact({
    type: "EXPERIMENTAL",
    experimentalArtifactId: td.env.EXPERIMENTAL_ARTIFACT_ID,
    data: JSON.stringify(args),
  });
  return [
    `Created a spreadsheet workbook at ${artifact.pathPrefix}`,
    `Add sheets to ${artifact.pathPrefix}sheet-1.json,  ${artifact.pathPrefix}sheet-2.json, ...`
  ].join("\n");
}
```

#### functions/create_spreadsheet/json_schema.json

```json
{
  "type": "object",
  "properties": {
    "title": {
      "type": "string",
      "description": "Title of the spreadsheet workbook"
    }
  }
}
```

#### functions/put_spreadsheet_sheet/code.js

```javascript
async function handler(args, ctx) {
  const { path, ...sheetData } = args;
  await ctx.putChatFile(path, JSON.stringify(sheetData), {contentType: "application/json"});
  return `Updated sheet at ${path}`;
}
```

#### functions/put_spreadsheet_sheet/json_schema.json

This file defines a detailed schema based on Univer's IWorksheetData type. Key properties:

- `path`: Sheet file path (e.g., `/experimental/123/sheet-1.json`)
- `name`: Sheet name
- `cellData`: Cell data (nested object keyed by row/column indices)
- `mergeData`: Merged cell ranges
- `defaultColumnWidth`, `defaultRowHeight`: Default sizes
- `rowHeader`, `columnHeader`: Header settings
- `showGridlines`, `gridlinesColor`: Gridline display settings

#### Push Command

```bash
tdx agent push agents/my-project
```

### Step 3: Create Agent and Execute in Chat

#### agents/my-project/spreadsheet-agent/agent.yml

```yaml
name: Spreadsheet Agent
model: claude-4.5-sonnet
tools:
  - type: experimental_knowledge_base
    target: '@ref(type: "experimental_knowledge_base", name: "spreadsheet_tools")'
    target_function: create_spreadsheet
    function_name: create_spreadsheet
    function_description: Creates a new spreadsheet workbook
  - type: experimental_knowledge_base
    target: '@ref(type: "experimental_knowledge_base", name: "spreadsheet_tools")'
    target_function: put_spreadsheet_sheet
    function_name: put_spreadsheet_sheet
    function_description: Adds or updates a sheet in the spreadsheet workbook
```

```bash
tdx agent push agents/my-project
```

### Step 4: Create Spreadsheet in Chat

In the agent chat, you can create spreadsheets like this:

**User**: "Create a spreadsheet with sales data"

**Agent**: (Automatically calls the following tools)
1. `create_spreadsheet({ title: "Sales Data" })` to create workbook
2. `put_spreadsheet_sheet({ path: "/experimental/123/sheet-1.json", name: "Q1 Sales", cellData: {...} })` to add sheet

The created spreadsheet is displayed interactively in the browser using Univer:
- Cell editing and formatting
- Formula calculation
- Cell merging
- Show/hide gridlines

## How to Debug Experimental Resources

### Debugging Knowledge Bases

Use the `agent` skill to chat for testing KB function calls, and use the `agent-debug` skill to analyze chat logs.

### Debugging Artifacts (Browser Rendering)

Use Playwright CLI to debug artifact rendering issues interactively.

#### 1. Setup Playwright CLI

```bash
# Check if playwright-cli is available
which playwright-cli

# If not available, install it
npm install -g @playwright/cli@latest
playwright-cli install chromium
```

#### 2. Open Browser and Navigate to Chat

Ask the user to navigate to the target chat in LLM Console, then open Playwright:

```bash
# Open browser with Playwright CLI
playwright-cli --headed --browser chrome open https://www.treasuredata.com/
```

**Ask user**: "Please navigate to the LLM Console chat where the artifact is displayed, then let me know when ready."

#### 3. Debug Artifact Rendering

After user confirms page navigation is complete, start interactive debugging:

```bash
# Take snapshot of current page state
playwright-cli snapshot

# Click on UI elements (use refs from snapshot)
playwright-cli click <menu-ref>  # Click resource menu item

# Take screenshot to verify artifacts are loaded
playwright-cli screenshot
```

##### Take output of console.log

For general debugging, use console.log to output debug information, push changes, reproduce the issue in browser to generate debug logs, and then analyze logs to identify and fix the bug:

1. Add console.log statements to code.js or artifact files
2. Push updated artifact with `tdx agent push`
3. Clear existing console logs:
   ```bash
   playwright-cli console --clear
   ```
4. Reproduce the issue in browser (ask user to perform actions that trigger the bug)
5. Capture console output:
   ```bash
   playwright-cli console
   ```
6. Analyze the Result section (ignore Page and Events sections)

Example console output:

```
### Result
- [Console](.playwright-cli/console-2026-02-06T11-12-09-811Z.log)
### Page
- Page URL: https://example.com/app/chats/show/019c326c-b7e9-7543-8ce5-7da9aa85ffee
- Page Title: #019c326c-b7e9-7543-8ce5-7da9aa85ffee Show Chat | LLM Console
- Console: 2 errors, 0 warnings
### Events
- New console entries: .playwright-cli/console-2026-02-06T09-41-40-945Z.log#L289-L310
```

**Focus on the Result section** - the console log file contains the actual debug output. Read this file to analyze errors and debug information.

Reference:
- [playwright-mcp GitHub](https://github.com/microsoft/playwright-mcp)
- Run `playwright-cli --help` for full command list

## Best Practices

- Store artifact IDs in KB variables: `td.env.ARTIFACT_ID`
- Use `artifact.pathPrefix` for file paths: `artifact.pathPrefix + "data.json"`
- Static templates go in artifact files/, dynamic content uses ctx.putChatFile()

## Related Skills

- **agent** - Core agent configuration and workflow
- **agent-debug** - Troubleshooting agent behavior with logs
