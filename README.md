# HPC-QC 全球量子電腦分布圖

這是一個 Vue 3 + Vite + Leaflet 的互動式網頁前端，資料已從附件工作表 `三版資料_260415` 轉成 `src/data/hpc_qc_centers.json`。

## 功能

- 全球地圖總覽
- 區域切換：Global / Americas / EuroHPC JU / Asia-Pacific
- 國家顯示 / 隱藏
- Institution、Provider、Country 關鍵字搜尋
- 點擊地圖 marker 顯示詳細資料
- 下方詳細清單，可點擊後定位到地圖
- 可選擇是否顯示地圖標籤
- 可選擇是否群聚相近點位

## 安裝與執行

```bash
npm install
npm run dev
```

開啟終端機顯示的網址，通常是：

```text
http://localhost:5173
```

## 打包成靜態網頁

```bash
npm run build
```

輸出會在 `dist/` 資料夾，可以部署到 GitHub Pages、Nginx、校內伺服器或其他靜態網站服務。

## 更新資料

未來若量子中心增加，可修改：

```text
src/data/hpc_qc_centers.json
```

每筆資料格式如下：

```json
{
  "id": 1,
  "area": "Asia-Pacific",
  "countryRaw": "Japan",
  "country": "Japan",
  "institution": "University of Tokyo",
  "provider": "IBM",
  "address": "7 Chome-3-1 Hongo, Bunkyo City, Tokyo 113-8654日本",
  "longitude": 139.7601595757892,
  "latitude": 35.71381587
}
```
