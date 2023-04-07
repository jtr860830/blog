---
title: 使用 Cloudflare worker 以及 Google sheet 自製一個免費的短網址服務
date: 2023-04-08 05:26:58
categories:
  - Toys
tags:
  - Cloudflare worker
  - Cloudflare
  - Google API
  - Google sheet
  - JavaScript
---

最近租了一個新網域 (josh-hsieh.tw)，在設定 Cloudflare 的時候看到 Cloudflare worker 這有趣的東西。其實之前看到的時候就很想試試看，但後來都忘記了XD，想說不妨趁著這次來試試吧！

本來是想做一個 ChatGPT 的 ChatBot 的，但在做的過程中發現 OpenAI 的 API 要錢...！就打了退堂鼓，選擇做了這個小玩具。

## 環境準備

- [Node.js](https://nodejs.org)

## 建立一個有讀取 Google sheet 權限的 API Key

1. 到 [GCP (Google Cloud Platform) console](https://console.cloud.google.com/?hl=zh-TW) 新增一個專案，或是使用現有的專案也可以
2. 到 [GCP APIs Library](https://console.cloud.google.com/apis/library?hl=zh-TW)，啟用 Google Sheet API

![Activate Google Sheet API](acitve-google-sheet-api.png)

3. 接著進入[憑證](https://console.cloud.google.com/apis/credentials?hl=zh-TW)，點擊上方的"建立憑證"，選擇"API 金鑰"，接著就會看見跳出一個寫著 API Key 的小視窗

![Create Google API Key](create-google-api-key.png)

4. (Optional) 限制這個 Key 的權限來增加安全性，例如：只能使用 Google Sheet API 之類的

## 在 Google Drive 建立一個 Google Sheet

1. 進入 [Google Drive](https://drive.google.com/drive/my-drive) 後，點選"新增 Google 試算表"
2. 接著點進 sheet，看到網址會長這樣：`https://docs.google.com/spreadsheets/d/{GOOGLE_SHEET_ID}/edit#gid=0`
3. 把 GOOGLE_SHEET_ID 記下來，待會兒會用到

## 初始化一個 Cloudflare worker 專案

1. `npx wrangler init url-shortener`

![Init Cloudflare Worker](init-cloudflare-worker.png)

2. 進入專案資料夾，在 wrangler.toml 新增環境變數：

```toml
[vars]

# 填入你的 API Key
GOOGLE_SHEET_API_KEY = YOUR_API_KEY
# 填入你的 Google sheet ID
GOOGLE_SHEET_ID = YOUR_SHEET_ID
# URL 的數量限制
URL_MAX_NUM = 20
```

3. 修改 src/index.js: 

```javascript
export default {
  async fetch(request, env) {
    // 取得 URL Path -> URL 的縮寫
    const alias = new URL(request.url)
      .pathname
      .substring(1);
    // 取得 Google sheet 中的資料
    const sheetApiUrl = `https://sheets.googleapis.com/v4/spreadsheets/${env["GOOGLE_SHEET_ID"]}/values/A2:B${env["URL_MAX_NUM"] + 1}?key=${env["GOOGLE_SHEET_API_KEY"]}`;
    const res = await fetch(sheetApiUrl);
    const data = await res.json();
    // 查詢縮寫對應的 URL
    const row = data
      .values
      .find(el => el[0] === alias);
    // 如果找不到救回傳 404 Not Found
    if (row === undefined) return new Response("Not found", { status: 404 });
    const url = row[1];
    // 跳轉網頁
    return Response.redirect(url, 302);
  },
};
```

4. (Optional) `npm run start` 可以在 http://localhost:8787/ 做個簡單測試

> 記得要去 Google sheet 當中填入測試資料，不然永遠都會回 404

> 可以注意到 Google sheet API 的網址當中有一段 `A2:B${env["URL_MAX_NUM"] + 1}` 這是用來標示取得試算表內容的範圍，這個寫法會被 JS template literals 解析為 `A2:B21`，範圍如下圖，可以想像為使用滑鼠選取試算表範圍

![Google sheet range](google-sheet-range.png)

5. `npm run deploy`
6. 進入 Cloudflare dashboard 的 workers tab 就能看到 worker 出現了

![Cloudflare Worker](cloudflare-worker.png)

7. 接著可以點選 worker 進入 trigger 的 tab，查看自動分配給 worker 的網址，就能透過網址使用短網址服務了

![Cloudflare Worker Route](cloudflare-worker-route.png)

8. 也可以設定用 custom domain

![Cloudflare worker Custom Domain](cloudflare-worker-custom-domain.png)

> 本來有想要用 googleapis 這個 Node module，但因為 Cloudflare worker 的 runtime 跟 Node.js 不同所以不能使用，希望未來會越來越兼容

> 附上完整程式碼: https://github.com/jtr860830/cloudflare-worker-url-shortener

## 參考資料

- https://developers.cloudflare.com/workers/
- https://developers.google.com/sheets/api/guides/concepts
- https://hsuan9522.medium.com/google-sheet-v4-api-efdec9a96bf3
