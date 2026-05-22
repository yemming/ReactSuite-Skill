---
name: NetSuite React Application
description: 建立在 NetSuite 平台上以 React + Vite 開發的 SPA 應用程式架構
---

# NetSuite React Application Skill

當用戶要求建立一個 NetSuite SPA 應用程式時，請遵循以下架構與模式生成專案檔案。

---

## 架構概述

「True SPA embedded in Suitelet」架構：

- **前端打包**：React + Vite + TypeScript，透過 `vite-plugin-singlefile` 編譯成單一 HTML 字串。
- **後端整合**：Build Script 將 HTML 字串直接嵌入 Suitelet JS 原始碼（Inlined）。
- **運行時**：NetSuite Suitelet 作為單一入口，直接回傳包含完整 SPA 程式碼的 HTML，**不依賴額外的 File Cabinet HTML 檔案**。
- **SDF 支援**：所有 Script 檔案均對應獨立的 SDF XML 物件定義，確保 Bundling 與 Deploy 完整性。

**運作流程**：
1. 使用者存取 Suitelet URL。
2. Suitelet 執行，讀取自身內嵌的 HTML 字串，注入環境變數（Context）與初始資料（Data）。
3. Client 端 React SPA 啟動，接管頁面互動，後續透過 `fetch` 呼叫同一 Suitelet URL（Action）進行資料交換。

---

## 專案目錄結構

```
{PROJECT_NAME}/
├── src/
│   ├── App.tsx              # 主應用程式
│   ├── App.css              # 全域樣式
│   ├── main.tsx             # React 掛載點
│   ├── index.css            # Tailwind CSS 入口
│   ├── components/          # React 元件
│   └── services/
│       └── api.ts           # NetSuite API 整合層
├── netsuite/
│   └── {PROJECT_NAME}_Suitelet.js  # Suitelet 後端
├── deploy/
│   ├── FileCabinet/SuiteScripts/{PROJECT_NAME}/
│   ├── Objects/             # SDF 物件定義
│   ├── manifest.xml
│   ├── deploy.xml
│   └── project.json
├── scripts/
│   └── build_suitelet.js    # 建構腳本
├── vite.config.ts
├── package.json
├── tsconfig.json
└── PROJECT_SPEC.md
```

---

## 技術堆疊

| 類別 | 技術 |
|------|------|
| 前端框架 | React 19+ with Hooks |
| 前端建構 | Vite 7+ |
| 型別系統 | TypeScript 5+ |
| CSS 框架 | Tailwind CSS 4+ |
| 打包策略 | vite-plugin-singlefile + HTML Injection（注入至 Suitelet） |
| 後端平台 | NetSuite Suitelet（SuiteScript 2.1） |
| 部署工具 | SuiteCloud CLI（SDF） |
| Bundling | 所有 Script 必須建立 Script Record 以利 Bundle 選取 |

---

## 核心檔案模板

### vite.config.ts
```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';
import { viteSingleFile } from 'vite-plugin-singlefile';

export default defineConfig({
  plugins: [react(), tailwindcss(), viteSingleFile()],
  build: {
    outDir: 'dist',
    target: 'esnext',
    assetsInlineLimit: 100000000,
    chunkSizeWarningLimit: 100000000,
    cssCodeSplit: false,
    rollupOptions: {
      output: {
        inlineDynamicImports: true,
      },
    },
  },
});
```

### index.css（NetSuite CSS 覆蓋）

```css
@import "tailwindcss";

html, body, #root {
    height: 100%;
    margin: 0;
    padding: 0;
    overflow: hidden;
}

/* 隱藏 NetSuite 頁面框架元素 */
.uir-form-header,
.uir-page-title-row, .uir-page-title, .uir-global-message,
.uir-header-rows, #bgslash, .div__header,
tr.uir-page-title-row, .bg_white.pg-title-row,
.uir-label-row,
.uir-page-footer {
    display: none !important;
}

/* 重設容器 margin/padding */
html, body, #root,
#custpage_html, #custpage_html_fs, #custpage_html_val, #custpage_app_html,
.uir-page-body, #body, form[name="main_form"] {
    margin: 0 !important;
    padding: 0 !important;
    width: 100% !important;
}

/* 強制表格全寬 */
.uir-outside-fields-table, .uir-fields-table, .uir-list-form-table {
    width: 100% !important;
    padding: 0 !important;
    border: none !important;
}

td.uir-page-body-td {
    padding: 0 !important;
}

#root {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
}
```

### package.json 腳本
```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "deploy:prepare": "tsc -b && vite build && node scripts/build_suitelet.js"
  }
}
```

### scripts/build_suitelet.js（HTML Injection）

```javascript
import fs from 'fs';
import path from 'path';
import { fileURLToPath } from 'url';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

const NETSUITE_DIR = path.join(__dirname, '../netsuite');
const DEPLOY_DIR = path.join(__dirname, '../deploy/FileCabinet/SuiteScripts/{PROJECT_NAME}');
const DIST_DIR = path.join(__dirname, '../dist');

if (!fs.existsSync(DEPLOY_DIR)) {
    fs.mkdirSync(DEPLOY_DIR, { recursive: true });
}

const htmlFiles = fs.readdirSync(DIST_DIR).filter(f => f.endsWith('.html'));
if (htmlFiles.length === 0) throw new Error('No HTML files found in dist/');

const rawHtml = fs.readFileSync(path.join(DIST_DIR, htmlFiles[0]), 'utf8');

const escapedHtml = rawHtml
    .replace(/\\/g, '\\\\')
    .replace(/`/g, '\\`')
    .replace(/\$/g, '\\$');

const suiteletPath = path.join(NETSUITE_DIR, '{PROJECT_NAME}_Suitelet.js');
let suiteletContent = fs.readFileSync(suiteletPath, 'utf8');

const START_MARKER = '// <!-- REACT_HTML_CONTENT_START -->';
const END_MARKER = '// <!-- REACT_HTML_CONTENT_END -->';
const regex = new RegExp(`${START_MARKER}[\\s\\S]*?${END_MARKER}`);
const replacement = `${START_MARKER}\n            let html = \`${escapedHtml}\`;\n            ${END_MARKER}`;

if (!regex.test(suiteletContent)) {
    console.warn('Warning: Injection markers not found. File copied without injection.');
    fs.writeFileSync(path.join(DEPLOY_DIR, '{PROJECT_NAME}_Suitelet.js'), suiteletContent);
} else {
    fs.writeFileSync(path.join(DEPLOY_DIR, '{PROJECT_NAME}_Suitelet.js'), suiteletContent.replace(regex, replacement));
    console.log('✅ Suitelet built with embedded HTML.');
}
```

---

## 前端 API 服務層（api.ts）

```typescript
const isNetSuite = typeof window !== 'undefined' && window.NETSUITE_CONTEXT;

declare global {
  interface Window {
    NETSUITE_CONTEXT?: {
      suiteletUrl: string;
      userId: string;
      userName: string;
    };
    NETSUITE_DATA?: {
      // 根據專案需求定義資料結構
    };
  }
}

// 取得初始資料（優先使用伺服器注入）
export async function fetchData(): Promise<DataType[]> {
  if (isNetSuite && window.NETSUITE_DATA) {
    return window.NETSUITE_DATA.items;
  }
  return DEMO_DATA;
}

// API 呼叫（使用 GET 避免 CSRF）
export async function callApi(action: string, params: Record<string, string>): Promise<any> {
  if (!isNetSuite) return { success: true };

  const url = new URL(window.NETSUITE_CONTEXT!.suiteletUrl);
  url.searchParams.set('action', action);
  Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v));

  const response = await fetch(url.toString());
  return response.json();
}
```

---

## 後端 Suitelet 模式

```javascript
/**
 * @NApiVersion 2.1
 * @NScriptType Suitelet
 * @NModuleScope SameAccount
 */
define(['N/ui/serverWidget', 'N/runtime', 'N/record', 'N/query', 'N/url'],
    (serverWidget, runtime, record, query, url) => {

        function onRequest(context) {
            const action = context.request.parameters.action;

            if (action) {
                handleApiRequest(context, action);
            } else {
                handlePageRequest(context);
            }
        }

        function handlePageRequest(context) {
            // <!-- REACT_HTML_CONTENT_START -->
            let html = '<h1>HTML not injected. Please run npm run deploy:prepare.</h1>';
            // <!-- REACT_HTML_CONTENT_END -->

            const contextScript = `<script>
                window.NETSUITE_CONTEXT = ${JSON.stringify({
                    suiteletUrl: url.resolveScript({
                        scriptId: runtime.getCurrentScript().id,
                        deploymentId: runtime.getCurrentScript().deploymentId
                    }),
                    userId: runtime.getCurrentUser().id,
                    userName: runtime.getCurrentUser().name
                })};
                window.NETSUITE_DATA = ${JSON.stringify(getData())};
            </script>`;

            html = html.includes('</head>')
                ? html.replace('</head>', contextScript + '</head>')
                : contextScript + html;

            context.response.setHeader({ name: 'Content-Type', value: 'text/html; charset=utf-8' });
            context.response.write(html);
        }

        function handleApiRequest(context, action) {
            let result;
            switch (action) {
                case 'getData': result = getData(); break;
                case 'saveRecord': result = saveRecord(context.request.parameters); break;
            }
            context.response.setHeader({ name: 'Content-Type', value: 'application/json' });
            context.response.write(JSON.stringify(result));
        }

        function getData() {
            const sql = `SELECT id, name FROM customrecord_xxx WHERE isinactive = 'F'`;
            return query.runSuiteQL({ query: sql }).asMappedResults();
        }

        return { onRequest };
    });
```

---

## SDF XML 定義與 Bundling 規範

每一個 Script 檔案都必須有對應的 Script Record XML 定義，否則 Bundle Builder 可能無法選取。

### Suitelet 定義
```xml
<suitelet scriptid="customscript_{project_name}">
    <n>{Project Display Name}</n>
    <notifyadmins>F</notifyadmins>
    <notifyowner>T</notifyowner>
    <notifyuser>F</notifyuser>
    <scriptfile>[/SuiteScripts/{PROJECT_NAME}/{PROJECT_NAME}_Suitelet.js]</scriptfile>
    <scriptdeployments>
        <scriptdeployment scriptid="customdeploy_{project_name}">
            <title>{Project Display Name}</title>
            <status>RELEASED</status>
            <loglevel>DEBUG</loglevel>
            <allroles>T</allroles>
            <isdeployed>T</isdeployed>
        </scriptdeployment>
    </scriptdeployments>
</suitelet>
```

### Custom Record 範例
```xml
<customrecord scriptid="customrecord_{record_name}">
    <recordname>{Record Display Name}</recordname>
    <includeinsearchmenu>T</includeinsearchmenu>
    <customrecordcustomfields>
        <customrecordcustomfield scriptid="custrecord_{field_name}">
            <fieldtype>TEXT</fieldtype>
            <label>{Field Label}</label>
        </customrecordcustomfield>
    </customrecordcustomfields>
</customrecord>
```

### deploy.xml
```xml
<deploy>
    <files>
        <path>~/SuiteScripts/{PROJECT_NAME}/*</path>
    </files>
    <objects>
        <path>~/Objects/*</path>
    </objects>
</deploy>
```

---

## 開發守則

1. **`deploy/` 是自動生成的** — 不要手動修改，會被覆蓋
2. **Suitelet 源碼在 `netsuite/`** — 這是 Source of Truth
3. **API 使用 GET 請求** — 避免 NetSuite CSRF 問題
4. **本地用 Mock Data** — `api.ts` 中準備 DEMO_DATA
5. **HTML 使用路徑載入** — `file.load({ path: ... })`，不用 ID

---

## 生成新專案時

當用戶說「用 NetSuite React App Skill 建立 XXX 專案」時：

1. 確認專案名稱和主要功能需求
2. 根據上述結構生成所有必要檔案
3. 將 `{PROJECT_NAME}` 替換為實際專案名稱
4. 根據功能需求設計資料結構和 API actions
5. 生成對應的 Custom Records 和 Suitelet 邏輯

---

## 部署指令

```bash
npm run deploy:prepare   # Build + 注入 HTML 到 Suitelet

cd deploy
suitecloud project:deploy  # 部署到 NetSuite
```
