<!-- ⚠️⚠️ Do Not Delete This! bug_report_template ⚠️⚠️ -->
<!-- Please read our Rules of Conduct: https://opensource.microsoft.com/codeofconduct/ -->
<!-- 🕮 Read our guide about submitting issues: https://github.com/microsoft/vscode/wiki/Submitting-Bugs-and-Suggestions -->
<!-- 🔎 Search existing issues to avoid creating duplicates. -->
<!-- 🧪 Test using the latest Insiders build to see if your issue has already been fixed: https://code.visualstudio.com/insiders/ -->
<!-- 💡 Instead of creating your report here, use 'Report Issue' from the 'Help' menu in VS Code to pre-fill useful information. -->
<!-- 🔧 Launch with `code --disable-extensions` to check. -->
**Does this issue occur when all extensions are disabled?:**
No, but because the issue stems with the [`json-language-features`](https://github.com/microsoft/vscode/tree/main/extensions/json-language-features) extention which is an built into `vscode`, such reports must be made to the extension's publisher as directed by the [guidelines](https://github.com/microsoft/vscode/wiki/Submitting-Bugs-and-Suggestions).
<!-- 🪓 If you answered No above, use 'Help: Start Extension Bisect' from Command Palette to try to identify the cause. -->
<!-- 📣 Issues caused by an extension need to be reported directly to the extension publisher. The 'Help > Report Issue' dialog can assist with this. -->
| **VS Code** | `1.73.1` |
|--:|:--|
| **Commit** | 6261075646f055b99068d3688932416f2346dd3b |
| **Date** | `2022-11-09T02:22:48.959Z` |
| **Electron** | `19.0.17` |
| **Chromium** | `102.0.5005.167` |
| **Node.js** | `16.14.2` |
| **V8** | `10.2.154.15-electron.0` |
| **OS** | `Darwin arm64 22.1.0` |
| **Sandboxed** | `No` |

## Problem

Strict file-matching within `json.schemas: [...]` is ignored under a specific workspace folder's `/.vscode/settings.json` when the tail-end of a mapping's `fileMatch` path is equal to the entirety of another mapping's `fileMatch` path. This causes validation errors on the `.json` file whose schema is being ignored given that both schema mappings are made to account for different specifications.

### Steps to Reproduce:

<details><summary><b><code>TL;DR</code></b></summary>
<p>

> 1. Create an empty workspace.
> 2. Perform [Step 2](#repr.step.2), below.
> 3. Clone [this repo](https://github.com/LeShaunJ/vsc-json-issue.git) into the workspace.
> 4. Perform [Step 6](#repr.step.6) and onward.

</p>
</details><br/>

1. Create the following folder structure:
   ```
   MyWorkspace/
   └─ MyProject/
      ├─ .vscode/
      ├─ configs/
      └─ container/
         └─ configs/
   ```
2. <a id="repr.step.2"></a>Set `[Workspace ]Settings > Extensions > JSON > Trace: Server` to `verbose`:
   * In `MyWorkspace/MyWorkspace.code-workspace`:
     ```json
     {
         "json.trace.server": "verbose"
     }
     ```
3. Under [`MyProject/configs/`](https://github.com/LeShaunJ/vsc-json-issue/blob/main/configs):
   1. Create the file, [`config.schema.json`](https://github.com/LeShaunJ/vsc-json-issue/blob/main/configs/config.schema.json):
      ```json
      {
          "$schema": "http://json-schema.org/draft-07/schema",
          "$id": "https://domain.com/schemas/MyProject/config/schema.json",
          "title": "config.json",
          "description": "Basic Config",
          "type": "object",
          "properties": {
              "env": { "type": "string" },
              "prefix": { "type": "string" },
              "suffix": { "type": "string" }
          },
          "required": ["env"],
          "additionalProperties": false
      }
      ```
   2. Create the file, [`config.json`](https://github.com/LeShaunJ/vsc-json-issue/blob/main/configs/config.json):
      ```json
      {
          "env": "staging"
      }
      ```
4. Under [`MyProject/container/configs/`](https://github.com/LeShaunJ/vsc-json-issue/blob/main/container/configs):
   1. Create the file, [`config.schema.json`](https://github.com/LeShaunJ/vsc-json-issue/blob/main/container/configs/config.schema.json):
      ```json
      {
          "$schema": "http://json-schema.org/draft-07/schema",
          "$id": "https://domain.com/schemas/MyProject/container/config/schema.json",
          "title": "config.json",
          "description": "Container Config",
          "type": "object",
          "properties": {
              "container_name": { "type": "string" },
              "image_location": { "type": "string", "format": "uri-reference" },
              "arch": { "type": "string" }
          },
          "required": ["container_name","image_location","public_key"],
          "additionalProperties": true
      }
      ```
   2. Create the file, [`config.json`](https://github.com/LeShaunJ/vsc-json-issue/blob/main/container/configs/config.json):
      ```json
      {
          "container_name": "ansible-runner",
          "image_location": "https://domain.com/images/ansible-runner:2.13",
          "arch": "amd64"
      }
      ```
5. Under `MyProject/.vscode/`, create the [`settings.json`](https://github.com/LeShaunJ/vsc-json-issue/blob/main/.vscode/settings.json) file with the following `json.schema` mappings:
   ```json
   {
       "json.schemas": [
           {
               "fileMatch": [
                   "/configs/config.json",
               ],
               "url": "/configs/config.schema.json"
           },
           {
               "fileMatch": [
                   "/container/configs/config.json",
               ],
               "url": "/container/configs/config.schema.json"
           }
       ]
   }
   ```
6. <a id="repr.step.6"></a>Open [`MyProject/container/configs/config.json`](https://github.com/LeShaunJ/vsc-json-issue/blob/main/container/configs/config.json):
   * Note the validation errors highlighted on each property.
   * Note how the file is highlighted yellow in the `Explore` sidebar.
7. <a id="repr.step.7"></a>Open the `PROBLEMS` and note the **second** validation error for [`MyProject/container/configs/config.json`](https://github.com/LeShaunJ/vsc-json-issue/blob/main/container/configs/config.json) (_more on this [later](#diagnosis.bug)_):
   ```
   ˅ {…} config.json MyProject • container/configs ➎
         ⚠️ Missing property "env". [Ln 1, Col 1]
         ⚠️ Missing property "public_key". [Ln 1, Col 1]
         ⚠️ Property container_name is not allowed. [Ln 2, Col 5]
         ⚠️ Property image_location is not allowed. [Ln 3, Col 5]
         ⚠️ Property arch is not allowed. [Ln 4, Col 5]
   ```
8. Open the `OUTPUT` tab:
   1. In the log source drop down, select `JSON Language Server`.
   2. <kbd>ctrl</kbd>+<kbd>f</kbd> (_or <kbd>⌘</kbd>+<kbd>f</kbd>_) to search:
      1. Select the <kbd>.*</kbd> icon to enable `regex`.
      2. Search for, `\/\*\/config\/config\.json"`.
   3. Cycle from the last search-match until you find the following (_file-path roots are truncated as they'd differ depending on the reproduction location_):
      ```json
      {
          "settings": {
              "http": {
                  "proxy": "",
                  "proxyStrictSSL": true
              },
              "json": {
                  "validate": {
                      "enable": true
                  },
                  "format": {
                      "enable": true
                  },
                  "keepLines": {
                      "enable": false
                  },
                  "schemas": [
                      {
                          "url": "file:///MyWorkspace/MyProject/config/config.schema.json",
                          "fileMatch": [
                              "file:///MyWorkspace/MyProject/config/config.json",
                              "file:///MyWorkspace/MyProject/*/config/config.json" /* root cause */
                          ]
                      },
                      {
                          "url": "file:///MyWorkspace/MyProject/container/config/config.schema.json",
                          "fileMatch": [
                              "file:///MyWorkspace/MyProject/container/config/config.json",
                              "file:///MyWorkspace/MyProject/*/container/config/config.json"
                          ]
                      }
                  ],
                  "resultLimit": 5001,
                  "jsonFoldingLimit": 5001,
                  "jsoncFoldingLimit": 5001
              }
          }
      }
      ```
      * Note the how the mappings are instructed to resolve.

## Diagnosis

This is because any path with `configs/config.json` is instructed to map to [`/configs/config.schema.json`](https://github.com/LeShaunJ/vsc-json-issue/blob/main/configs/config.schema.json). I've narrowed the greedy nature of this situation to way the mappings are globalized:

https://github.com/microsoft/vscode/blob/3c83412a4fc8d316aefb2385850564d7a21dc9f6/extensions/json-language-features/client/src/jsonClient.ts#L521-L527

Which is fine in terms of globally mapping a schema; however, it means that stricter mappings are ignored if the tail-end of one mapping's `fileMatch` path equal to the entirety of another mapping's `fileMatch` path.

<a id="diagnosis.bug"></a>Surely, this is *mostly* by design, but if you recall `PROBLEMS` that are raised ([Reproduction Step 7](#repr.step.7)), the validation is in fact validating against *both* schemas––as if one schema is the *parent* of the other. This is confirmed by the appearance of, `Missing property "public_key". [Ln 1, Col 1]`, which stems from a [`requirement`](https://github.com/LeShaunJ/vsc-json-issue/blob/25b468beb5920b20560db68a5811807f4089d31c/container/configs/config.schema.json#L12) in [`container/configs/config.schema.json`](https://github.com/LeShaunJ/vsc-json-issue/blob/main/container/configs/config.schema.json) that was *intentionally* left out of [`container/configs/config.json`](https://github.com/LeShaunJ/vsc-json-issue/blob/main/container/configs/config.json) to prove the claim as a **bug**. What we're seeing is what one can only assume is **unintentional inheritance**, as this quirk would have otherwise been [explained](https://code.visualstudio.com/docs/languages/json) in some detail or another.

## Solution

* While it is true that `$schema` directly in the `.json` files supersedes the the above, this is insufficient if the `.json` file is used by a 3rd-party module/package/app that does not allow foreign properties.
* Besides this, the fact that `config.json` tends to be a *go-to* name, the chances of two configuration files for two separate 3rd-party modules/packages/apps clashing is that much higher.
* I believe `fileMatch` paths should make a clear distinction between `directory/file.json` and `/directory/file.json`:
  * A path within `fileMatch` is **relative** if it *does not* start with `/`.
    * Relative `fileMatch` paths are *still* matched as both `/project/directory/file.json` and `/project/*/directory/file.json`
  * A path within `fileMatch` is **absolute** when it *does* begin with `/`:
    * Absolute `fileMatch` paths are matched *only* as `/project/directory/file.json`.

So instead of:

https://github.com/microsoft/vscode/blob/3c83412a4fc8d316aefb2385850564d7a21dc9f6/extensions/json-language-features/client/src/jsonClient.ts#L519-L531

We simply remove the globalization of paths that start with `/`:

```ts
	for (const fileMatch of fileMatches) { 
		if (fileMatchPrefix) { 
			if (fileMatch[0] === '/') { 
				addMatch(fileMatchPrefix + fileMatch); 
			} else { 
				addMatch(fileMatchPrefix + '/' + fileMatch); 
				addMatch(fileMatchPrefix + '/*/' + fileMatch); 
			} 
		} else { 
			addMatch(fileMatch); 
		}
	}
```

Unless there's something I'm missing as to why it was specifically done this way, I don't otherwise see why this cannot be done.
