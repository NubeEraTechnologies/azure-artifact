---

# 🚀 Azure Artifacts – npm Package Management with Azure Pipelines

This guide explains **step-by-step** how to set up, host, and publish **npm packages** to **Azure Artifacts**, then integrate it with **Azure DevOps Pipelines** for automated builds and publishing.

---

## 📘 Topics Covered

1. Package Management in Azure Artifacts
2. Hosting and Consuming npm Packages
3. Integrating with Build Pipelines
4. Connecting to External Package Feeds
5. Manual `.npmrc` Configuration on Ubuntu

---

## 🧩 Prerequisites

Before starting, make sure you have:

* An **Azure DevOps organization** and **project** (e.g., `nubeeratechnologies/Artifacts`)
* An **Artifacts feed** created (e.g., `myartifact`)
* A **Personal Access Token (PAT)** with `Packaging (Read & Write)` permissions
* A **build agent** (either self-hosted or Microsoft-hosted)
* Installed tools:

  ```bash
  sudo apt update
  sudo apt install -y nodejs npm git
  node -v
  npm -v
  ```

---

## ⚙️ Step 1 — Create a `.npmrc` File (Ubuntu Manual Setup)

You’ll need to manually create an `.npmrc` file to point npm to your Azure Artifacts feed.

### 🧱 1.1 Create Directory and File

```bash
mkdir -p /home/azureuser/npm-artifact
cd /home/azureuser/npm-artifact
nano .npmrc
```

### 🧾 1.2 Add the Following Content

Replace `nubeeratechnologies`, `Artifacts`, and `myartifact` with your actual organization, project, and feed names.

```ini
registry=https://pkgs.dev.azure.com/nubeeratechnologies/Artifacts/_packaging/myartifact/npm/registry/
always-auth=true

; begin auth token
//pkgs.dev.azure.com/nubeeratechnologies/Artifacts/_packaging/myartifact/npm/registry/:username=nubeeratechnologies
//pkgs.dev.azure.com/nubeeratechnologies/Artifacts/_packaging/myartifact/npm/registry/:_password=[BASE64_ENCODED_PERSONAL_ACCESS_TOKEN]
//pkgs.dev.azure.com/nubeeratechnologies/Artifacts/_packaging/myartifact/npm/registry/:email=npm requires email to be set but doesn't use the value
; end auth token
```

Save and exit:
`CTRL + O`, `ENTER`, `CTRL + X`

---

## 🔑 Step 2 — Generate & Encode Personal Access Token (PAT)

1. Go to **Azure DevOps → User Settings → Personal Access Tokens → New Token**
2. Choose:

   * **Organization:** nubeeratechnologies
   * **Scope:** `Packaging (Read, Write, Manage)`
3. Copy your token.

### Encode it to Base64

Run this command:

```bash
node -e "require('readline').createInterface({input:process.stdin,output:process.stdout,historySize:0}).question('PAT> ',p => { b64=Buffer.from(p.trim()).toString('base64');console.log(b64);process.exit(); })"
```

Paste your PAT when prompted and press Enter.

Copy the Base64 output and replace it in `.npmrc` where `[BASE64_ENCODED_PERSONAL_ACCESS_TOKEN]` appears.

---

## 📦 Step 3 — Create Your npm Package

In your repo, create a simple npm package structure:

```
package/
 ├── index.js
 ├── package.json
 ├── README.md
```

Example contents:

**index.js**

```js
console.log("Hello from @nubeeratechnologies/my-lib!");
```

**package.json**

```json
{
  "name": "@nubeeratechnologies/my-lib",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "build": "echo Building... && mkdir -p dist && cp index.js dist/"
  }
}
```

---

## 🔄 Step 4 — Push to Azure Repo

```bash
git init
git add .
git commit -m "Initial commit with npm package and pipeline setup"
git remote add origin https://dev.azure.com/nubeeratechnologies/Artifacts/_git/my-lib
git push -u origin main
```

---

## 🧰 Step 5 — Create Azure Pipeline (YAML)

Create a file **`azure-pipelines.yml`** in your repo root:

```yaml
trigger:
  branches:
    include:
      - main

pool:
  name: 'default'

variables:
  NODE_VERSION: '18.x'
  PACKAGE_PATH: './package'
  ORGANIZATION: 'nubeeratechnologies'
  FEED: 'myartifact'

steps:
# 1️⃣ Use Node.js
- task: UseNode@1
  inputs:
    versionSpec: '$(NODE_VERSION)'
  displayName: 'Use Node.js $(NODE_VERSION)'

# 2️⃣ Authenticate npm to Azure Artifacts
- task: npmAuthenticate@0
  displayName: 'Authenticate npm to Azure Artifacts'
  inputs:
    workingFile: '.npmrc'

# 3️⃣ Verify .npmrc
- script: |
    echo "Sanitized .npmrc content:"
    grep -v "_authToken" .npmrc || true
  displayName: 'Debug: Verify .npmrc exists'

# 4️⃣ Install and build
- script: |
    cd $(PACKAGE_PATH)
    npm ci || npm install
    npm run build
  displayName: 'Install & Build Package'

# 5️⃣ Publish to Azure Artifacts
- script: |
    cd $(PACKAGE_PATH)
    echo "Publishing package..."
    npm publish --registry=https://pkgs.dev.azure.com/$(ORGANIZATION)/_packaging/$(FEED)/npm/registry/
  displayName: 'Publish npm package to Azure Artifacts'

# 6️⃣ Cleanup
- script: |
    echo "Cleaning up .npmrc..."
    rm -f $(Build.SourcesDirectory)/.npmrc || true
  displayName: 'Cleanup .npmrc'
```

---

## ▶️ Step 6 — Run the Pipeline

1. Go to **Azure DevOps → Pipelines**
2. Click **“New Pipeline”**
3. Choose your repository
4. Select **“Existing YAML file”** and pick `azure-pipelines.yml`
5. Run manually (first time)

---

## ✅ Step 7 — Verify Package Published

Check **Azure DevOps → Artifacts → Feed → Packages**

You should see:

```
@nubeeratechnologies/my-lib  v1.0.0
```

---

## 🔗 Step 8 — Consuming the Package

To install your internal package in another project:

1. Copy `.npmrc` to that project root.
2. Install:

   ```bash
   npm install @nubeeratechnologies/my-lib
   ```
3. Import and use:

   ```js
   const mylib = require('@nubeeratechnologies/my-lib');
   ```

---

## 🌍 Optional — Connect to External Feeds

Enable **Upstream Sources** to use public feeds like npmjs.org.

**Azure DevOps → Artifacts → Feed Settings → Upstream sources → Add**

Then update `.npmrc`:

```ini
registry=https://pkgs.dev.azure.com/nubeeratechnologies/Artifacts/_packaging/myartifact/npm/registry/
always-auth=true
@*registry=https://registry.npmjs.org/
```

---

## 🧼 Step 9 — Cleanup

To remove credentials:

```bash
rm ~/.npmrc
```

To clean local artifacts:

```bash
npm cache clean --force
```

---

## ✅ Summary

| Step | Description                         | Status |
| ---- | ----------------------------------- | ------ |
| 1️⃣  | Create `.npmrc` manually in Ubuntu  | ✅      |
| 2️⃣  | Generate & Base64 encode PAT        | ✅      |
| 3️⃣  | Create npm package                  | ✅      |
| 4️⃣  | Push to Azure Repo                  | ✅      |
| 5️⃣  | Create and run pipeline             | ✅      |
| 6️⃣  | Publish package to Azure Artifacts  | ✅      |
| 7️⃣  | Consume package from other projects | ✅      |

---

## 📚 References

* [Microsoft Learn: Azure Artifacts and npm](https://learn.microsoft.com/azure/devops/artifacts/npm/npmrc)
* [Azure Pipelines npmAuthenticate Task](https://learn.microsoft.com/azure/devops/pipelines/tasks/package/npm-authenticate)
* [Azure Artifacts Overview](https://learn.microsoft.com/azure/devops/artifacts/overview)
* [Azure Pipelines YAML Schema](https://learn.microsoft.com/azure/devops/pipelines/yaml-schema)

---
