---
title: Unit Test 單元測試(一)
date: 2018-09-09 00:00:00
categories:
  - Software Engineering
tags:
  - JavaScript
  - Unit Test
  - Test
  - TDD
---

## 單元測試

寫測試在進行專案開發佔了很重要的一部份，不少人應該會用 print 之類的方式去做測試，但是這不容易系統化而且肉眼觀察會出錯，而測試的最基礎的單位就是單元測試，基本上就是測試一個功能是否能符合需求，要注意以下幾點：

1. 無外部相依性
2. Test Case 彼此間沒有相依性
3. 一個 Test Case 一次只測一件事

以下使用 TDD 的方式來進行單元測試，我使用的環境如下：

- [Node.js](https://nodejs.org/en)
- [mocha](https://mochajs.org/)

## TDD

TDD (Test Driven Development) 為測試導向的開發模式，簡單來說就是先寫好測試再開始開發，好處是在寫測試的時候會想好每個部分的功能，例如函數輸入輸出的格式，可以避免開發過程中，寫出不必要的功能或是冗余的程式碼，以下以 mocha 套件去做示範

## 專案配置

創建專案資料夾，並進入專案資料夾，如果不熟悉指令操作，直接用編輯器打開整個資料夾比較方便，並依序用下列指令初始化專案：

1. `npm init`
  -  test command: `mocha --compilers js:@babel/register`
2. `npm install mocha @babel/core @babel/preset-env @babel/register`

> 安裝 babel 是因為我想用 ES6 module 的語法，而 mocha 還尚未支援

最後在專案資料夾底下新增 .babelrc，內容如下：

```json
{
  "presets": [ "@babel/preset-env" ]
}
```

## 撰寫測試

以簡單的加減函數為範例，在專案資料夾底下新增 test 資料夾，並在裡面新增 cal.test.js 檔案，內容如下：

```javascript
import * as assert from 'assert'

describe('add function', function () {
  it('add(1, 2) = 3', function () {
    assert.deepStrictEqual(add(1, 2), 3)
  })

  it('add(1, -2) = -1', function () {
    assert.deepStrictEqual(add(1, -2), -1)
  })
})

describe('minus function', function () {
  it('minus(1, 2) = -1', function () {
    assert.deepStrictEqual(minus(1, 2), -1)
  })

  it('minus(1, -2) = 3', function () {
    assert.deepStrictEqual(minus(1, -2), 3)
  })
})
```

- `assert` 是 Node.js 內建的套件，作用是為了標示與驗證程式開發者預期的結果。當程式執行到斷言的位置時，對應的斷言應該為 true。 若 assert 不為 true 時，程式會中止執行，並給出錯誤訊息
- `describe()` 描述區塊測試內容，可視為一個測試的群組，裡面可以跑很多測試，第一個參數僅為對功能的描述，第二個參數為測試用的 callback
- `it()` 可撰寫每條測試內容，參數內容同上
- 測試檔的命名推薦跟被測試的功能一樣，副檔名為 .test.js，然後會放置在專案資料夾底下的 test 資料夾裡面

> 到這裡可以下 `npm test` 進行測試，會發現是錯誤的，是因為尚未實現 `add` 跟 `minus` 函數

## 根據測試撰寫功能

在專案資料夾底下新增 cal.js 檔案，內容如下：

```javascript
function add(x, y) {
  return x + y
}

function minus(x, y) {
  return x - y
}

export {
  add, 
  minus
}
```

之後在測試檔最上面補上 `import { add, minus } from '../cal.js'` 再進行測試 `npm test`，就能看到結果為 pass

![result](result.webp)

> 附上完整程式碼: https://github.com/jtr860830/UnitTest_Example_TDD

第一次寫 Medium 若有什麼錯誤或是要改善的地方都歡迎告訴我！下一篇將會寫關於 BDD (Behavior Driven Development) 的 Story

[轉載](https://medium.com/%E6%B6%88%E5%A4%B1%E7%9A%84%E8%B3%87%E9%A1%A7%E5%AE%A4/unit-test-%E5%96%AE%E5%85%83%E6%B8%AC%E8%A9%A6-%E4%B8%80-24db7534c1a6)

> 2023 轉移至此