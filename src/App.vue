<script setup>
import { ref, watchEffect } from "vue";

// 服務帳號 ID
const CLIENT_ID = "<服務帳號 ID>";
// API 金鑰
const API_KEY = "<API KEY>";
// Google Sheet ID
const SHEET_ID = "<Google Sheets ID>";
// Sheets 中要取得的資料範圍，下面為格式範例
const RANGE = "工作表1!A1:B5";

const DISCOVERY_DOC =
  "https://sheets.googleapis.com/$discovery/rest?version=v4";
const SCOPES = "https://www.googleapis.com/auth/spreadsheets.readonly";

// 判斷 gapi 是否載入
const gapiInited = ref(false);
// 判斷 gis 是否載入
const gisInited = ref(false);

// 驗證按鈕文字
const authorizeBtnText = ref("Authorize");
// 登出按鈕狀態
const isSignOutBtnShow = ref(false);
// 資料內容
const contentText = ref("");

const gapiLoaded = () => {
  gapi.load("client", initializeGapiClient);
};

const gisLoaded = () => {
  window.tokenClient = google.accounts.oauth2.initTokenClient({
    client_id: CLIENT_ID,
    scope: SCOPES,
    callback: "",
  });
  gisInited.value = true;
};

const initializeGapiClient = async () => {
  await gapi.client.init({ apiKey: API_KEY, discoveryDocs: [DISCOVERY_DOC] });
  gapiInited.value = true;
};

// 按下登入按鈕
const handleAuthClick = () => {
  tokenClient.callback = async (resp) => {
    if (resp.error !== undefined) {
      throw resp;
    }
    isSignOutBtnShow.value = true;
    authorizeBtnText.value = "Refresh";
    await listData();
  };

  if (gapi.client.getToken() === null) {
    tokenClient.requestAccessToken({ prompt: "consent" });
  } else {
    tokenClient.requestAccessToken({ prompt: "" });
  }
};

// 按下登出按鈕
const handleSignoutClick = () => {
  const token = gapi.client.getToken();
  if (token !== null) {
    google.accounts.oauth2.revoke(token.access_token);
    gapi.client.setToken("");
    contentText.value = "";
    authorizeBtnText.value = "Authorize";
    isSignOutBtnShow.value = false;
  }
};

// 列出資料
const listData = async () => {
  let response;
  try {
    response = await gapi.client.sheets.spreadsheets.values.get({
      spreadsheetId: SHEET_ID,
      range: RANGE,
    });
  } catch (err) {
    contentText.value = err.message;
    return;
  }
  const range = response.result;
  if (!range || !range.values || range.values.length == 0) {
    contentText.value = "No values found.";
    return;
  }
  const output = range.values.reduce((str, row) => `${str}\n${row[0]}`);
  contentText.value = output;
};

watchEffect(() => {
  if (window.gapi) {
    gapiLoaded();
  }
  if (window.google) {
    gisLoaded();
  }
});

const loadScript = (src, onLoad) => {
  const script = document.createElement("script");
  script.async = true;
  script.defer = true;
  script.src = src;
  script.onload = onLoad;
  document.head.appendChild(script);
};

loadScript("https://apis.google.com/js/api.js", gapiLoaded);
loadScript("https://accounts.google.com/gsi/client", gisLoaded);
</script>
<template>
  <div>
    <p>Sheets API Quickstart</p>
    <button v-if="gapiInited && gisInited" @click="handleAuthClick">
      {{ authorizeBtnText }}
    </button>
    <button
      v-if="gapiInited && gisInited"
      v-show="isSignOutBtnShow"
      @click="handleSignoutClick"
    >
      Sign Out
    </button>
    <pre id="content" style="white-space: pre-wrap">{{ contentText }}</pre>
  </div>
</template>
