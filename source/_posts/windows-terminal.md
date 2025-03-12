---
title: 更改cmd命令提示視窗背景
date: 2025-03-12 15:46:58
description: 讓 Terminal 視窗變好看，增加上班時的小確幸
categories: memo
tags: terminal
---

## 安裝 Windows Terminal
在 Microsoft Store 搜尋 Windows Terminal 下載安裝

---

## 更改預設啟動
開啟 Windows Terminal，下拉選單 > 設定
預設啟動 > 調整自己最常用的設定檔
預設終端應用程式 > 調整為 Windows 終端機
![01](../images/windows-terminal/01.png)

---

## 改背景

左下角開啟 setting.json
![02](../images/windows-terminal/02.png)

profiles > defaults 加入背景圖片與透明度設定
```
"backgroundImage": "D:\\terminal_background\\chiikawa.jpg",
"backgroundImageOpacity": 0.11,
```
![03](../images/windows-terminal/03.png)
