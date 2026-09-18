# 學校圖書館管理與統計系統

**Library Record System** 是一套面向教職員的行動優先網頁系統。此版本以 React、TypeScript、Tailwind CSS、Firebase Authentication、Cloud Firestore 與 Firebase Hosting 建置。系統已預先連結至 Firebase 專案 `chclsclib`，並將預設管理員固定為 `laisl@lsc.edu.hk`。

> **安全設計**：Firebase Web SDK 的設定值會隨網頁發佈，這是 Firebase 的正常架構。資料存取安全由 `firestore.rules` 執行。請勿將規則改成 `allow read, write: if true`。

## 交付內容與功能範圍

| 模組 | 已完成的功能 |
| --- | --- |
| Google Workspace 登入 | 使用 Firebase Authentication 的 Google Provider。`laisl@lsc.edu.hk` 首次登入時自動取得 Admin 身分。其他帳號必須出現在 Firestore 白名單。 |
| 權限控制 | Admin 可使用完整後台；Teacher 只可進入早讀欠書登記及學生總覽。未列入白名單的登入會被登出並顯示「無存取權限」。 |
| 學生資料 | 學號唯一、姓名、班別、性別及在學狀態。支援班別、性別、在學狀態及姓名/學號篩選。Admin 可新增個案或標記離校。 |
| 閱讀出奇蛋 | Admin 可依 1、2、3 個印章類別選取指定活動，也可依原印章類別輸入自訂活動。支援按班別批次勾選及完整登記日誌。 |
| 禮物換領 | 系統依累計印數提示小獎（3 印）、中獎（6 印）及大獎（10 印）。核銷時須選擇經手人，並鎖定領取時間及人員。 |
| 早讀欠帶書 | Teacher 與 Admin 均可批次登記。累計 3 次顯示高優先級藍紙警示；發紙後會把本學年計數歸零，並將三筆對應欠書日期、處理老師及發紙時間寫入審計日誌。 |
| 分析儀表板 | 顯示本月蓋印 KPI、各班蓋印排行榜、欠書與藍紙比較、換領狀態及最近藍紙審計日誌。離校學生不計入有效統計。 |
| Excel | 可下載學生、教職員白名單及禮物經手人員範本；支援批量匯入及匯出完整備份工作簿。 |
| 學年升班 | 升班前強制下載全系統備份。1A 至 5D 會升一級；6A 至 6D 轉為離校封存；新學年欠書計數會歸零，原始藍紙與閱讀日誌保留。 |
| 手機體驗與 PWA | Mobile-First 版面、大型觸控勾選框、Sticky Bottom Bar、手機底部導覽與 Web Manifest，適合巡班使用。 |

## Firebase 資料結構

Cloud Firestore 使用下列集合。每個文件 ID 均可追溯至系統內的唯一鍵或操作流水號。

| 集合 | 用途 | 主要權限 |
| --- | --- | --- |
| `whitelist_teachers` | 教職員 Email、姓名、`Admin` 或 `Teacher` 角色 | Admin 管理；登入者可讀自己的白名單文件。 |
| `students` | 學號、姓名、班別、性別、學籍、印數、換領與統計快取 | 所有白名單教職員可讀；Admin 才可新增或修改。 |
| `reading_egg_logs` | 每次出奇蛋活動、印數、時間及管理員 | 全體白名單可讀；Admin 寫入。 |
| `overdue_books` | 每一次早讀欠帶書紀錄及是否已由藍紙處理 | 全體白名單教職員可讀、建立及標記已處理。 |
| `blue_slips_audit` | 藍紙歷史、三個對應日期、處理老師及備註 | 全體白名單可讀及建立；僅 Admin 可修改或刪除。 |
| `reward_handlers` | 禮物發放經手人清單 | 全體白名單可讀；Admin 管理。 |
| `academic_archives` | 學年封存摘要及升班前快照 | 僅 Admin 可讀寫。 |
| `system_config` | 可擴充的系統設定 | 全體白名單可讀；僅 Admin 可修改。 |

## 首次上線前的 Firebase Console 設定

請以擁有 `chclsclib` 專案 Owner 或 Firebase Admin 權限的 Google 帳號完成此節。這些設定需要該帳號在 Google/Firebase Console 內授權，無法由前端設定檔取代。

1. 在 Firebase Console 開啟 `chclsclib` 專案。如果尚未建立 Cloud Firestore，請先建立 **Native mode** 資料庫，並按學校資料政策選定區域。
2. 前往 **Authentication → Sign-in method**，啟用 **Google** 供應商並儲存。Firebase 官方流程要求先啟用 Google provider，前端才能使用 Google SDK 登入。[1]
3. 前往 **Authentication → Settings → Authorized domains**。確認 `chclsclib.web.app` 與 `chclsclib.firebaseapp.com` 位於授權網域清單；若日後使用自訂網域，必須一併加入。
4. 保留本專案的 `firebase.json`、`.firebaserc` 與 `firestore.rules`。它們已設定單頁應用程式重寫規則、Firebase 專案別名及角色型 Firestore 規則。

## 部署至 Firebase Hosting

本機已安裝 Firebase CLI 作為開發相依套件，因此不需要全域安裝。部署前先確認帳號對 Firebase 專案具有 Firebase Hosting 與 Firestore Rules 的發佈權限。

```bash
cd /home/ubuntu/library-record-system
pnpm install
pnpm run check
pnpm run build
pnpm exec firebase login
pnpm exec firebase deploy --only firestore,hosting
```

`firebase deploy --only firestore,hosting` 會同時發佈 Firestore Rules 及 `dist/public` 的前端檔案。Firebase Hosting 會產生以下公開網址：

```text
https://chclsclib.web.app
https://chclsclib.firebaseapp.com
```

Firebase Hosting 的標準流程會將靜態內容部署至專案的 `web.app` 與 `firebaseapp.com` 子網域。[2] Firestore Rules 可與程式碼一起透過 CLI 發佈，便於版本控制與回溯。[3]

## 上線後的首次建檔順序

正式資料庫在首次使用時會是空白狀態。請依下列順序完成啟用。

1. 使用 `laisl@lsc.edu.hk` 的 Google Workspace 帳號登入。該帳號由程式碼及 Firestore Rules 視為最高層級 Admin，不需要先匯入自己。
2. 進入 **管理員後台**，下載三套 Excel 範本。
3. 匯入教職員白名單，再匯入禮物經手人員清單與學生資料。
4. 建議使用另一個 Teacher 白名單帳號登入，驗證其只看得到「早讀欠書登記」及「學生總覽」。
5. 以未列入白名單的帳號測試登入，預期結果是系統登出並顯示拒絕存取訊息。

## 日常操作與資料保護

每日巡班時，Teacher 可先篩選班別，以大尺寸勾選框選擇學生，再在底部確認列一次過完成登記。達到三次時，請從同一頁的紅色警示區按下「已成功發放藍紙」。系統會把這三筆未處理日期寫入藍紙審計紀錄，並把該生目前學年的欠書計數歸零。

每個學年結束時，Admin 必須先在「年度升班與學年歸零封存」頁面下載完整 Excel 備份。系統只有在完成備份下載後才會開啟升班操作。升班會保存封存快照，將 6 年級學生標為離校，並重設其餘學生的新學年欠書計數。

## 原始碼位置與維護入口

| 位置 | 說明 |
| --- | --- |
| `client/src/pages/` | 各業務頁面，包括儀表板、出奇蛋、欠書、獎品、學生、後台及維護入口。 |
| `client/src/contexts/LibraryContext.tsx` | 用戶角色、示範模式、本機快取、Firestore 載入與所有業務寫入流程。 |
| `client/src/lib/firebase.ts` | Firebase 專案配置、Google Provider 與預設 Admin Email。 |
| `client/src/lib/excel.ts` | Excel 範本、匯入解析及完整報表工作簿輸出。 |
| `firestore.rules` | 角色型資料存取規則；每次修改後必須重新部署。 |
| `firebase.json` | Firebase Hosting 的 `dist/public` 發佈路徑與 SPA rewrite。 |
| 系統內「程式碼升級與維護入口」 | 向下一位維護者顯示 Firebase 配置、版本、修訂摘要及可複製的 AI 升級需求。 |

## 已完成的驗證

本版本已通過 TypeScript 型別檢查及正式前端建置。瀏覽器驗證確認：儀表板正常顯示，藍紙發放會新增審計記錄並把欠書計數歸零，Teacher 角色進入管理員頁面時會被權限閘門拒絕。預覽環境使用示範資料；當使用已授權的 Firebase 帳號登入正式網站時，應用程式會改從 Cloud Firestore 讀取與寫入中央資料。

## References

[1]: https://firebase.google.com/docs/auth/web/google-signin "Authenticate Using Google with JavaScript | Firebase"
[2]: https://firebase.google.com/docs/hosting/quickstart "Get started with Firebase Hosting"
[3]: https://firebase.google.com/docs/firestore/security/get-started "Get started with Cloud Firestore Security Rules"
