---
title: 淺談程式語言中的 interface
date: 2025-10-11 04:00:00
categories:
  - Software Engineering
tags:
  - OOP
code_block_shrink: false
---

物件導向程式設計的三大核心概念: **封裝 (Encapsulation)**、**繼承 (Inheritance)**、**多型 (Polymorphism)**，而 **Interface** 是**實現多型的一種重要方式**。

> 並非只有 interface 能實現多型，繼承 (inheritance) 與方法覆寫 (override) 也能達到相同目的

先來談談什麼是多型

# 多型 (Polymorphism)

多型的定義是: 同一個方法呼叫，根據物件的實際型態，會有不同的行為。聽起來可能很拗口，不過這段話最重要的關鍵字其實是**行為 (Behavior)**。

例如在一些常見的物件導向程式語言的寫法:

```java
// 假設 Animal 是一種 interface
Animal a = new Dog();
a.speak();

a = new Cat();
a.speak();
```

可以看到上方程式碼中的 `a` 變數，他被宣告為一種 `Animal`，而狗 (`Dog`) 跟貓 (`Cat`) 都會叫 (`speak`)，所以這兩種型別的實例 (instance)，都能夠被視為一種 `Animal`。

我們關注的不是它的實際型別，而是能否表現出特定的行為。

# Interface

Interface 能夠被視為一種行為的**契約**，定義一種 interface 會需要在其中定義一些函數 (或方法) 的簽名，但不定義任何具體的實作方式，任何型別只要**遵守**這個契約，就能被視為同一種 `interface`，以上方的 Animal 為例，程式碼可能就會如下:

```java
interface Animal {
    void speak();
}

class Dog implements Animal {
    public void speak() {
        /* 旺旺旺 */
    }
}

class Cat implements Animal {
    public void speak() {
        /* 喵喵喵 */
    }
}
```

這樣的設計使得不同型別可以在相同的語境下被操作，而不需要知道具體型別。這樣的範例可能還無法理解真正的實際用途，我們來看看一些程式語言在標準函式庫的使用。

## Java 的範例: `Comparable`

`Comparable` 是 Java 標準函式庫中非常經典的範例。它定義了物件間的比較方式，讓像 `Collections.sort()`` 這樣的函式能夠運作。

```java
public interface Comparable<T> {
    int compareTo(T o);
}
```

使用範例如下:

```java
import java.util.*;

class Student implements Comparable<Student> {
    String name;
    int score;

    Student(String name, int score) {
        this.name = name;
        this.score = score;
    }

    // compareTo 決定排序邏輯
    public int compareTo(Student other) {
        return Integer.compare(this.score, other.score);
    }

    public String toString() {
        return name + "(" + score + ")";
    }
}

public class Main {
    public static void main(String[] args) {
        List<Student> students = Arrays.asList(
            new Student("Alice", 90),
            new Student("Bob", 80),
            new Student("Charlie", 85)
        );

        // 這裡 Comparable 就會被使用到，然後依照分數大小把 students 排序好
        Collections.sort(students);
        System.out.println(students);
    }
}
```

## Golang 的範例: `fmt.Stringer`

在 Golang 中，`fmt.Stringer` 是最常見的 interface 之一。

```go
type Stringer interface {
    String() string
}
```

只要實作了 `String` 這個方法，就能被 `fmt.Print`、`fmt.Println`、`fmt.Sprintf` 正確輸出

```go
package main

import "fmt"

type Person struct {
    Name string
    Age  int
}

// 實作 fmt.Stringer 介面
func (p Person) String() string {
    return fmt.Sprintf("%s (%d)", p.Name, p.Age)
}

func main() {
    p := Person{"Alice", 25}
    fmt.Println(p) // 自動呼叫 p.String()
}
```

輸出:

```
Alice (25)
```

## Rust 的範例: `Display` Trait

在 Rust 中，trait 與 interface 十分類似。只要實作了 `Display` trait，就能被 `println!` 印出

```rust
pub trait Display {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result<(), Error>;
}
```

範例如下:

```rust
use std::fmt;

struct Person {
    name: String,
    age: u8,
}

// 為 Person 實作 Display trait
impl fmt::Display for Person {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{} ({})", self.name, self.age)
    }
}

fn main() {
    let p = Person { name: "Alice".to_string(), age: 25 };
    println!("{}", p); // 自動呼叫 fmt::Display::fmt()
}
```

輸出:

```
Alice (25)
```

# 結語

無論是 Java 的 `interface`、Go 的 `interface`，還是 Rust 的 `trait`，它們的本質其實都是在描述行為的抽象 (abstraction of behavior)。

物件導向程式設計 (OOP) 並不只是語法糖 (syntax sugar) 或關鍵字 (keyword) 的使用，而是一種設計思維。多型讓我們能**以行為為核心**設計程式，讓不同型別的物件在相同介面下展現出各自的特性。

OOP 不是語言的特性，而是一種思維模式。而多型的精神，也不僅存在於傳統的 OOP 語言中——無論使用的是 Java、Go、Rust，或任何沒有 `class` 關鍵字的語言，都能透過不同的設計方式，實現同樣的以行為為核心的多型概念。