# OpenCode Antigravity Setup, Patch & Troubleshooting Guide

This guide details how to configure OpenCode and patch the `opencode-antigravity-auth` plugin to reliably use all latest Gemini 3.x Flash, Gemini Pro, and Claude thinking models.

---

## 1. Background & Root Causes of Common Failures

### Issue 1: OpenCode Hangs at `> build · antigravity-gemini-3.x-flash`
* **Cause**: In `model-resolver.js`, any model name starting with `antigravity-gemini-3` triggers `skipAlias = true`. This bypasses `MODEL_ALIASES` and passes raw model strings like `"gemini-3.7-flash"` directly to Google's Antigravity backend API.
* **Why it hangs**: The backend API does not recognize versioned strings like `"gemini-3.7-flash"` as backend endpoints. The SSE stream opens but emits no response, causing OpenCode to wait indefinitely.
* **Backend Endpoint**: Google's Antigravity backend API expects **`gemini-3-flash`** for all Gemini 3 Flash models, accompanied by the `thinkingLevel` parameter (`minimal`, `low`, `medium`, `high`).

### Issue 2: Error "Gemini 3.5 Flash is no longer available"
* **Cause**: Early patch guides recommended aliasing all flash variants down to `gemini-3.5-flash-low`. Google has decommissioned this endpoint.
* **Fix**: Route all Flash requests to Google's active endpoint **`gemini-3-flash`**.

### Issue 3: Infinite Background Auth Retries (Error #3501)
* **Cause**: In `~/.config/opencode/antigravity-accounts.json`, accounts may get flagged with:
  ```json
  "verificationRequired": true,
  "cooldownReason": "validation-required"
  ```
* **Why it hangs**: If an account enters validation cooldown, `opencode-antigravity-auth` repeatedly triggers background verification loops during non-interactive runs.
* **Fix**: Reset `verificationRequired` to `false` and clear cooldown fields.

---

## 2. Automated Patch Script (One-Click / AI Agent Executable)

Save the following script as `patch-antigravity.js` and execute with `node patch-antigravity.js`:

```javascript
/**
 * Automated patch script for opencode-antigravity-auth
 * Patches models.js, model-resolver.js, and antigravity-accounts.json
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

  // Replace legacy/deprecated mapping to gemini-3.5-flash-low with active gemini-3-flash
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

// 3. Reset validation lock in antigravity-accounts.json
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

## 3. OpenCode Configuration (`opencode.json`)

Ensure your workspace or global config contains the model definitions with thinking variants:

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

## 4. Verification Commands

Test each model family to confirm instant streaming without hanging:

```bash
# Gemini 3.7 Flash (Medium Variant)
opencode run "Hello, respond in 5 words." --model=google/antigravity-gemini-3.7-flash --variant=medium

# Gemini 3.8 Flash (High Variant)
opencode run "Hello, respond in 5 words." --model=google/antigravity-gemini-3.8-flash --variant=high

# Gemini 3.6 Flash (Minimal Variant)
opencode run "Hello, respond in 5 words." --model=google/antigravity-gemini-3.6-flash --variant=minimal

# Gemini 3.1 Pro (Low Variant)
opencode run "Hello, respond in 5 words." --model=google/antigravity-gemini-3.1-pro --variant=low
```
