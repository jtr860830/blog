---
title: Unit Test 單元測試(二)
date: 2018-12-13 00:00:00
categories:
  - Software Engineering
tags:
  - JavaScript
  - Unit Test
  - Test
  - BDD
---

## 環境

這次來說明使用 BDD 的方式進行單元測試，使用的環境如下：

1. [Node.js]()
2. [Chai.js]()
3. [Cucumber.js]()

BDD (Behavior Driven Development) 是 TDD 的改版，跟 TDD 同樣地先撰寫測試，但是在撰寫測試前還要先寫一份規格，以達成能夠跟非技術人員溝通的目的，以下用 cucumber 套件做示範

## 專案配置

創建專案資料夾，並進入專案資料夾，如果不熟悉指令操作，直接用編輯器打開整個資料夾比較方便，並依序用下列指令初始化專案：

1. `npm init`

- test command: `cucumber-js --require-module @babel/register`

2. `npm install cucumber @babel/core @babel/preset-env @babel/register chai`

> 同樣需要 babel 去支援 ES6 語法

最後在專案資料夾底下新增 .babelrc，內容如下：

```json
{
  "presets": ["@babel/preset-env"]
}
```

## 撰寫規格

跟第一篇一樣以加減函數為範例，在專案資料夾底下新增 features 資料夾，並在裡面新增 cal.feature 檔案，內容如下：

中文版:

```gherkin
#language: zh-TW
功能: 計算機
  進行加減乘除四則運算

  場景: 1 加 2 會等於 3
    假設 第一個數字是 1
    並且 第二個數字是 2
    當 兩者相加
    那麼 結果會是 3

  場景: 2 減 1 會等於 1
    假設 第一個數字是 2
    並且 第二個數字是 1
    當 第一個數字減去第二個
    那麼 結果會是 1
```

英文版:

```gherkin
Feature: Calculator
  In order to calculate

  Scenario: Add 1 and 2 should be 3
    Given the first number is 1
    And the second number is 2
    When add two number
    Then result should be 3

  Scenario: 2 minus 1 should be 1
    Given the first number is 2
    And the second number is 1
    When the first number minus the second
    Then result should be 1
```

- `Feature (功能)` 功能的敘述
- `Scenario (場景)` 描述測試案例，並使用 `Given (假設)` `When (當)` `Then (那麼)` `And (並且)` 詳細說明案例
- 規格檔案的命名也是推薦跟被測試的功能一樣，副檔名為 .feature，會放置在專案資料夾底下的 features 資料夾裡面
- .feature 檔案使用 Gherkin 語法，有支援多國語言，預設應為英文，若要使用其他語言要加上 `#language: ...` 註解，上方分別為中文及英文範例，擇一即可

## 撰寫測試

使用 `npm test` ，會出下方結果：

![result](result-1.png)

黃色文字搭配問號的意思是此步驟尚未實作，注意：Undefined. Implement with the following snippet:這段文字，下面會直接給測試用的程式碼片段，把這些片段複製下來，並在 features 資料夾底下新增 step_definitions 資料夾，裡面再新增一個 cal.step.js 檔案，最後把程式碼貼上去，然後完成它們，結果如下：

```javascript
// features/step_definitions/cal.step.js
import { Given, When, Then } from "cucumber";
import * as chai from "chai";

chai.should();

Given("the first number is {int}", function (first) {
  // Write code here that turns the phrase above into concrete actions
  this.first = first;
});

Given("the second number is {int}", function (second) {
  // Write code here that turns the phrase above into concrete actions
  this.second = second;
});

When("add two number", function () {
  // Write code here that turns the phrase above into concrete actions
  this.result = add(this.first, this.second);
});

Then("result should be {int}", function (result) {
  // Write code here that turns the phrase above into concrete actions
  this.result.should.be.equal(result);
});

When("the first number minus the second", function () {
  // Write code here that turns the phrase above into concrete actions
  this.result = minus(this.first, this.second);
});
```

- 這次斷言的套件使用 Chai，選擇 should 風格的敘述
- 從 Scenario 中整數會被解析成 `{int}` 、雙引號刮起來的文字會被解析成 `{string}` 並帶入測試用的 callback 當中
- 無需定義重複的測試步驟
- 步驟的命名方式相同，副檔名為 .step.js，會放置在專案中 features/step_definitions 底下

## 撰寫功能

在專案資料夾底下新增 cal.js 檔案，完成 add 跟 minus，內容如下：

```javascript
function add(x, y) {
  return x + y;
}

function minus(x, y) {
  return x - y;
}

export { add, minus };
```

最後記得在 cal.step.js 中 `import { add, minus } from "../../cal"` 就能夠進行測試了，結果如下：

![result](result-2.png)

> 附上完整程式碼: https://github.com/jtr860830/UnitTest_Example_BDD

單元測試的部分到此告一段落，未來應該會再講到系統測試，敬請期待。下一篇可能會寫關於 zsh 或是 vim 客製化設定的入門，同樣地，歡迎指正文章內容的錯誤！

[轉載](https://medium.com/%E6%B6%88%E5%A4%B1%E7%9A%84%E8%B3%87%E9%A1%A7%E5%AE%A4/unit-test-%E5%96%AE%E5%85%83%E6%B8%AC%E8%A9%A6-%E4%BA%8C-c0556eae1cf4)

> 2023 轉移至此
