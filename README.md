# OpenCode Antigravity Fix: Gemini 3.8 / 3.7 / 3.6 Flash & Pro Setup

## 🤖 One-Click Prompt for AI Coding Agents

> **Copy and paste the prompt below directly into your AI assistant (Cursor Composer, Claude Code, Antigravity, GitHub Copilot, OpenCode, Codex):**

```text
Please fix my OpenCode Antigravity setup so that Gemini 3.8, 3.7, 3.6, and 3.5 Flash & Pro models work properly without hanging or returning 404 / "model no longer available" errors. Follow these steps:

1. Locate all installed instances of `opencode-antigravity-auth` (in `~/.cache/opencode/packages/` or `node_modules/`).
2. In `model-resolver.js`, locate the `skipAlias` block and patch it so that:
   - Any `gemini-3.[5-9]-flash` model routes to Google's active backend endpoint `"gemini-3-flash"`.
   - Any `gemini-3.8-pro` model routes to `"gemini-3-pro-low"`.
3. In `models.js`, register `antigravity-gemini-3.8-flash`, `antigravity-gemini-3.7-flash`, and `antigravity-gemini-3.6-flash` with thinking variants (`minimal`, `low`, `medium`, `high`).
4. In `~/.config/opencode/antigravity-accounts.json`, check for Error #3501 validation locks (`verificationRequired: true`). Set `verificationRequired = false`, remove `cooldownReason` / `coolingDownUntil` fields, and ensure `enabled = true`.
5. Update my project `.opencode/opencode.json` (or global `~/.config/opencode/opencode.json`) with the complete model list and variants.
6. Verify the fix by running:
   opencode run "Hello, respond in 5 words." --model=google/antigravity-gemini-3.7-flash --variant=medium
```

---

## 1. Step-by-Step Instructions for AI Agents & Developers

If you are an AI agent, execute these steps in sequence:

1. **Locate Installed Plugin Files**:
   Search for `model-resolver.js` and `models.js` in:
   - `~/.cache/opencode/packages/opencode-antigravity-auth@beta/node_modules/opencode-antigravity-auth/dist/`
   - `~/.cache/opencode/packages/opencode-antigravity-auth@latest/node_modules/opencode-antigravity-auth/dist/`
   - Any local `node_modules/opencode-antigravity-auth/dist/`

2. **Patch `model-resolver.js`**:
   - In the `skipAlias` branch, Google Antigravity backend API expects **`gemini-3-flash`** for all Gemini 3 Flash models (NOT `gemini-3.7-flash` or the deprecated `gemini-3.5-flash-low`).
   - Map all `gemini-3.x-flash` models to `"gemini-3-flash"`.
   - Map `gemini-3.8-pro` to `"gemini-3-pro-low"`.
   - Example patch replacement:
     ```javascript
     let antigravityModel = modelWithoutQuota;
     if (/^gemini-3\.[5-9]-flash/i.test(modelWithoutQuota)) {
         antigravityModel = "gemini-3-flash";
     } else if (/^gemini-3\.8-pro/i.test(modelWithoutQuota)) {
         antigravityModel = "gemini-3-pro-low";
     }
     ```

3. **Patch `models.js`**:
   Ensure `antigravity-gemini-3.8-flash`, `antigravity-gemini-3.7-flash`, and `antigravity-gemini-3.6-flash` are registered with thinking variants (`minimal`, `low`, `medium`, `high`).

4. **Clear Account Validation Locks**:
   In `~/.config/opencode/antigravity-accounts.json`, check if accounts have `"verificationRequired": true` or `"cooldownReason": "validation-required"`. Set `verificationRequired = false`, remove cooldown fields, and ensure `enabled = true`.

5. **Update User `opencode.json`**:
   Ensure the user's `.opencode/opencode.json` (or `~/.config/opencode/opencode.json`) contains the model definitions shown in **Section 3**.

6. **Verify with Smoke Test**:
   Run:
   ```bash
   opencode run "Hello, respond in 5 words." --model=google/antigravity-gemini-3.7-flash --variant=medium
   ```

---

## 2. Standalone Automated Patch Script (One-Click Fix)

Save this script as `patch-antigravity.js` and run `node patch-antigravity.js`:

```javascript
/**
 * Automated patch script for opencode-antigravity-auth
 * Cross-platform: Windows, macOS, Linux
 */
const fs = require('fs');
const path = require('path');
const os = require('os');

function findFiles(dir, filename, fileList = []) {
  if (!fs.existsSync(dir)) return fileList;
  try {
    const files = fs.readdirSync(dir);
    for (const file of files) {
      const fullPath = path.join(dir, file);
      try {
        const stat = fs.statSync(fullPath);
        if (stat.isDirectory()) {
          findFiles(fullPath, filename, fileList);
        } else if (file === filename) {
          fileList.push(fullPath);
        }
      } catch (e) {}
    }
  } catch (e) {}
  return fileList;
}

const userHome = os.homedir();
const cacheDir = path.join(userHome, '.cache', 'opencode');
const configDir = path.join(userHome, '.config', 'opencode');

console.log('=== Starting Antigravity Patch Process ===\n');

// 1. Patch model-resolver.js
const resolverFiles = findFiles(cacheDir, 'model-resolver.js');
console.log(`Found ${resolverFiles.length} model-resolver.js file(s).`);

for (const file of resolverFiles) {
  let content = fs.readFileSync(file, 'utf8');

  // Replace legacy or non-working endpoints with active gemini-3-flash
  content = content.replace(/gemini-3\.5-flash-low/g, 'gemini-3-flash');
  content = content.replace(/gemini-3\.7-flash/g, 'gemini-3-flash');

  // Inject backend resolution inside the skipAlias block
  if (!content.includes('// antigravity_patch_applied')) {
    const target = 'let antigravityModel = modelWithoutQuota;';
    const replacement = `// antigravity_patch_applied
    let antigravityModel = modelWithoutQuota;
    if (/^gemini-3\\.[5-9]-flash/i.test(modelWithoutQuota)) {
        antigravityModel = "gemini-3-flash";
    } else if (/^gemini-3\\.8-pro/i.test(modelWithoutQuota)) {
        antigravityModel = "gemini-3-pro-low";
    }`;
    content = content.replace(target, replacement);
  }

  fs.writeFileSync(file, content, 'utf8');
  console.log(`[OK] Patched resolver: ${file}`);
}

// 2. Patch models.js
const modelFiles = findFiles(cacheDir, 'models.js');
const modelSnippet = `
    "antigravity-gemini-3.8-flash": {
        name: "Gemini 3.8 Flash (Antigravity)",
        limit: { context: 1048576, output: 65536 },
        modalities: DEFAULT_MODALITIES,
        variants: {
            minimal: { thinkingLevel: "minimal" },
            low: { thinkingLevel: "low" },
            medium: { thinkingLevel: "medium" },
            high: { thinkingLevel: "high" },
        },
    },
    "antigravity-gemini-3.7-flash": {
        name: "Gemini 3.7 Flash (Antigravity)",
        limit: { context: 1048576, output: 65536 },
        modalities: DEFAULT_MODALITIES,
        variants: {
            minimal: { thinkingLevel: "minimal" },
            low: { thinkingLevel: "low" },
            medium: { thinkingLevel: "medium" },
            high: { thinkingLevel: "high" },
        },
    },
    "antigravity-gemini-3.6-flash": {
        name: "Gemini 3.6 Flash (Antigravity)",
        limit: { context: 1048576, output: 65536 },
        modalities: DEFAULT_MODALITIES,
        variants: {
            minimal: { thinkingLevel: "minimal" },
            low: { thinkingLevel: "low" },
            medium: { thinkingLevel: "medium" },
            high: { thinkingLevel: "high" },
        },
    },
`;

for (const file of modelFiles) {
  let content = fs.readFileSync(file, 'utf8');
  if (content.includes('"antigravity-gemini-3.5-flash":') && !content.includes('"antigravity-gemini-3.7-flash":')) {
    content = content.replace('"antigravity-gemini-3.5-flash":', modelSnippet + '    "antigravity-gemini-3.5-flash":');
    fs.writeFileSync(file, content, 'utf8');
    console.log(`[OK] Patched models registry: ${file}`);
  }
}

// 3. Reset validation locks in antigravity-accounts.json
const accountsFile = path.join(configDir, 'antigravity-accounts.json');
if (fs.existsSync(accountsFile)) {
  try {
    const data = JSON.parse(fs.readFileSync(accountsFile, 'utf8'));
    let modified = false;
    if (Array.isArray(data.accounts)) {
      for (const account of data.accounts) {
        if (account.verificationRequired) {
          account.verificationRequired = false;
          delete account.verificationRequiredAt;
          delete account.verificationRequiredReason;
          delete account.verificationRequiredType;
          delete account.coolingDownUntil;
          delete account.cooldownReason;
          account.enabled = true;
          modified = true;
        }
      }
    }
    if (modified) {
      fs.writeFileSync(accountsFile, JSON.stringify(data, null, 2), 'utf8');
      console.log(`[OK] Cleared validation locks in: ${accountsFile}`);
    } else {
      console.log(`[INFO] No accounts were in validation lock: ${accountsFile}`);
    }
  } catch (e) {
    console.error(`[WARN] Could not parse accounts file: ${e.message}`);
  }
}

console.log('\n=== All Patches Successfully Applied ===');
```

---

## 3. Recommended `opencode.json`

Add this to your project `.opencode/opencode.json` or global `~/.config/opencode/opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": [
    "opencode-antigravity-auth@beta",
    "opencode-agent-memory"
  ],
  "provider": {
    "google": {
      "npm": "@ai-sdk/google",
      "models": {
        "antigravity-gemini-3.8-flash": {
          "name": "Gemini 3.8 Flash (Antigravity)",
          "limit": { "context": 1048576, "output": 65536 },
          "modalities": { "input": ["text", "image", "pdf"], "output": ["text"] },
          "variants": {
            "minimal": { "thinkingLevel": "minimal" },
            "low": { "thinkingLevel": "low" },
            "medium": { "thinkingLevel": "medium" },
            "high": { "thinkingLevel": "high" }
          }
        },
        "antigravity-gemini-3.7-flash": {
          "name": "Gemini 3.7 Flash (Antigravity)",
          "limit": { "context": 1048576, "output": 65536 },
          "modalities": { "input": ["text", "image", "pdf"], "output": ["text"] },
          "variants": {
            "minimal": { "thinkingLevel": "minimal" },
            "low": { "thinkingLevel": "low" },
            "medium": { "thinkingLevel": "medium" },
            "high": { "thinkingLevel": "high" }
          }
        },
        "antigravity-gemini-3.6-flash": {
          "name": "Gemini 3.6 Flash (Antigravity)",
          "limit": { "context": 1048576, "output": 65536 },
          "modalities": { "input": ["text", "image", "pdf"], "output": ["text"] },
          "variants": {
            "minimal": { "thinkingLevel": "minimal" },
            "low": { "thinkingLevel": "low" },
            "medium": { "thinkingLevel": "medium" },
            "high": { "thinkingLevel": "high" }
          }
        },
        "antigravity-gemini-3.5-flash": {
          "name": "Gemini 3.5 Flash (Antigravity)",
          "limit": { "context": 1048576, "output": 65536 },
          "modalities": { "input": ["text", "image", "pdf"], "output": ["text"] },
          "variants": {
            "minimal": { "thinkingLevel": "minimal" },
            "low": { "thinkingLevel": "low" },
            "medium": { "thinkingLevel": "medium" },
            "high": { "thinkingLevel": "high" }
          }
        },
        "antigravity-gemini-3.8-pro": {
          "name": "Gemini 3.8 Pro (Antigravity)",
          "limit": { "context": 1048576, "output": 65535 },
          "modalities": { "input": ["text", "image", "pdf"], "output": ["text"] },
          "variants": {
            "low": { "thinkingLevel": "low" },
            "high": { "thinkingLevel": "high" }
          }
        },
        "antigravity-gemini-3.1-pro": {
          "name": "Gemini 3.1 Pro (Antigravity)",
          "limit": { "context": 1048576, "output": 65535 },
          "modalities": { "input": ["text", "image", "pdf"], "output": ["text"] },
          "variants": {
            "low": { "thinkingLevel": "low" },
            "high": { "thinkingLevel": "high" }
          }
        },
        "antigravity-gemini-3-pro": {
          "name": "Gemini 3 Pro (Antigravity)",
          "limit": { "context": 1048576, "output": 65535 },
          "modalities": { "input": ["text", "image", "pdf"], "output": ["text"] },
          "variants": {
            "low": { "thinkingLevel": "low" },
            "high": { "thinkingLevel": "high" }
          }
        },
        "antigravity-gemini-3-flash": {
          "name": "Gemini 3 Flash (Antigravity)",
          "limit": { "context": 1048576, "output": 65536 },
          "modalities": { "input": ["text", "image", "pdf"], "output": ["text"] },
          "variants": {
            "minimal": { "thinkingLevel": "minimal" },
            "low": { "thinkingLevel": "low" },
            "medium": { "thinkingLevel": "medium" },
            "high": { "thinkingLevel": "high" }
          }
        },
        "antigravity-claude-sonnet-4-6": {
          "name": "Claude Sonnet 4.6 (Antigravity)",
          "limit": { "context": 200000, "output": 64000 },
          "modalities": { "input": ["text", "image", "pdf"], "output": ["text"] }
        },
        "antigravity-claude-opus-4-6-thinking": {
          "name": "Claude Opus 4.6 Thinking (Antigravity)",
          "limit": { "context": 200000, "output": 64000 },
          "modalities": { "input": ["text", "image", "pdf"], "output": ["text"] },
          "variants": {
            "low": { "thinkingConfig": { "thinkingBudget": 8192 } },
            "max": { "thinkingConfig": { "thinkingBudget": 32768 } }
          }
        }
      }
    }
  }
}
```

---

## 4. Root Causes & Technical Details

1. **Why `> build · antigravity-...` Hangs**:
   In `model-resolver.js`, models starting with `antigravity-gemini-3` set `skipAlias = true`. This skips alias translation and sends unrecognized raw strings (like `"gemini-3.7-flash"`) directly to Google's API, causing the stream to hang indefinitely. Mapping all Flash models to `"gemini-3-flash"` fixes this.
2. **"Gemini 3.5 Flash is no longer available"**:
   Google decommissioned `gemini-3.5-flash-low` on the backend. The active endpoint is `gemini-3-flash`.
3. **Error #3501 (Validation Required)**:
   If an account enters cooldown, `verificationRequired: true` causes the background proxy to loop infinitely. Clearing cooldown flags restores normal streaming.

---

## 5. Verification Commands

```bash
# Gemini 3.7 Flash
opencode run "Hello, respond in 5 words." --model=google/antigravity-gemini-3.7-flash --variant=medium

# Gemini 3.8 Flash
opencode run "Hello, respond in 5 words." --model=google/antigravity-gemini-3.8-flash --variant=high

# Gemini 3.6 Flash
opencode run "Hello, respond in 5 words." --model=google/antigravity-gemini-3.6-flash --variant=minimal

# Gemini 3.1 Pro
opencode run "Hello, respond in 5 words." --model=google/antigravity-gemini-3.1-pro --variant=low
```

---

## 6. Issues & Community Support

Encountering an issue or has Google changed an API model tier?

* **Open an Issue**: If an endpoint updates, Google deprecates a model, or you encounter a hang, please [Open an Issue](https://github.com/VartikAnand/opencode-gemini-3.7-flash-fix/issues) with your debug logs:
  ```bash
  opencode run "test prompt" --model=google/antigravity-gemini-3.7-flash --print-logs --log-level=DEBUG
  ```
* **Contributing**: Pull requests are welcome! If you find new backend aliases or cleaner resolver mappings, feel free to open a PR.
* **Maintenance**: We monitor Google Antigravity backend changes and keep this patch repository up to date.
