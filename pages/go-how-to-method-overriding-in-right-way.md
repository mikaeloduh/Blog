# Go 如何正確的使用 Method Overriding

你也許在想，為什麼前面要寫一篇「Go 如何錯誤的使用 Method Overriding」，然後再寫一篇正確的教學？因為透過這樣的對比，能更循序漸進理解不同語言的設計理念，以下就帶你如何使用 Go「正確的」做到 method overriding 的效果。

## 傳統物件導向繼承

繼承是一種物件導向語言（OOP）的核心特性，允許一個類別（子類別）繼承另一個類別（父類別）的屬性和方法。這使得子類別能夠重用父類別的程式碼，並根據需要覆寫（override）或擴充父類別的功能。

在 UML（統一建模語言）中，繼承關係通常用 Generalization（一般化）來表示，這種關係以帶有空心箭頭的線條連接子類別和父類別，箭頭指向父類別，象徵「**is-a**」的關係。

```
+-------------+    is-a     +-------------+
|   Animal    |◁————————————|     Dog     |
+-------------+             +-------------+
|+ makeSound()|             |+ makeSound()|
+-------------+             +-------------+
```

在這個例子中，`Dog` 一般化自 `Animal`，表示 `Dog` 是 `Animal` 的一種。



## Go 的 struct embedding 與介面

### Struct Embedding 的意義

Go 並沒有傳統的子類別與父類別關係，但可以透過將一個結構「嵌入」到另一個結構中，達到類似繼承的效果，但更強調組合而非繼承。嵌入（embedding）讓外層結構獲得被嵌入結構的方法與欄位，無需額外撰寫轉接（forwarding）程式碼。

然而，這種嵌入更偏向「**has-a**」的組合關係，而非「is-a」的類別繼承。這表示物件的能力來自於它所「擁有」的結構與介面實作，而非固定於類別層級結構中。在《Effective Go》中所述，嵌入的目的在於借用（borrow）另一個結構或介面的行為，而非從語法上宣告某種型態階層。

```
+---------+      has-a       +-------------+
|   Dog   |----------------->|   Animal    |
+---------+                  +-------------+
|         |                  |+ makeSound()|
+---------+                  +-------------+
```

用 UML 表達，在這個例子中，`Dog` 結構擁有一個 `Animal` 介面，表示 `Dog` 擁有 `makeSound` 的操作能力。

### Interface 與隱性實作

Go 的另一個設計關鍵是介面（interface）。Go 的介面僅描述方法特徵（method signatures），任何實作了該方法特徵的型態即「隱性地 (implicitly)」實作了該介面。這種設計避免了在程式碼中額外宣告「implements」關係，降低了不同套件間的依賴。（參考《Go Bootcamp》by Matt Aimonetti）

介面在此扮演多型的角色：透過定義介面方法，並在不同結構中提供各自的實作，我們得以在運行時選擇使用哪個具體實作。這樣的特性使我們能在 Go 中模擬傳統 OOP 的覆寫行為——不是透過繼承關係，而是透過替換不同實作（struct 實例）給同一個介面欄位。

## Go 對 method overriding 的解答

以下展示上篇範例的 Go 慣用語的寫法：

```go
package main

import "fmt"

// 定義一個 Getter 介面，描述必須具備 Get 方法
type Getter interface {
	Get() string
}

// 基礎結構，透過嵌入 Getter 來取得多型行為
type Base struct {
	Getter
}

// 預設的 Get 實作（此處不做實作，panic 表明需由具體類別實作）
func (b *Base) Get() string {
	panic("unimplemented")
}

func (b *Base) GetName() string {
	return b.Getter.Get()
}

// Sub 實作 Getter
type Sub struct {
}

func (t *Sub) Get() string {
	return "Sub"
}

// NewSub 建立一個 Base 實例並賦予 Sub 的實作
func NewSub() *Base {
	return &Base{
		Getter: &Sub{},
	}
}

func main() {
	userType := NewSub()

	fmt.Println(userType.GetName()) // output: Sub
}
```

**範例解析**

1. **介面定義**：Getter 介面要求實作 Get() string 方法。
2. **Base 結構**：Base 中嵌入了 Getter 介面，此時 Base 本身並無法提供有效的 Get 實作（以 panic 表達未完成實作）。GetName 則透過 Getter.Get() 呼叫真正的實作。
3. **Sub 實作 Getter**：Sub 結構實作了 Get 方法，返回 "Sub"。
4. **NewSub 建構函式**：透過 NewSub 將創建了的Base 結構的 Getter 欄位，綁定到 Sub 的實作。此時，Base 擁有 Sub 的行為。
5. **呼叫流程**：當 GetName 透過 b.Getter.Get() ，實際上呼叫到 Sub 的 Get 方法。

透過這種設計，我們在不使用類別繼承的情況下，做到了「當呼叫 GetName 時，自動根據注入的實作呼叫合適的 Get 方法」。此時的多型行為並非來自於繼承，而是來自於介面的動態實作替換。



## 這件事的根本意義

在傳統的物件導向程式設計（OOP）中，**繼承 (Inheritance)** 是一種核心概念，而**多型 (Polymorphism)** 則是OOP的另一個重要特性。繼承與多型之間有著密切的關聯，繼承是實現多型的基礎。

### 多型的定義

**多型 (Polymorphism)** 指的是同一個介面（方法）可以對應到不同的實作。這意味著可以使用相同的方法名稱來呼叫不同類別中的具體實作，根據物件的實際類型執行相應的行為。

多型主要分為兩種類型：

1. **編譯時多型（Compile-time Polymorphism）**：透過方法重載（Method Overloading）或運算子重載（Operator Overloading）實現。
2. **運行時多型（Runtime Polymorphism）**：透過方法覆寫（Method Overriding）和動態綁定（Dynamic Binding）實現。

在繼承的上下文中，我們主要關注的是**運行時多型**。

### 繼承如何支援多型

在運行時，Java 會根據物件的實際類型，來決定哪一個方法實作，這種機制稱為**動態綁定（Dynamic Binding）**

#### 以 Java 的繼承為例：

```java
class Base {
    public String get() {
        return "Base";
    }

    public String getName() {
        return get();
    }
}

class Sub extends Base {
    @Override
    public String get() {
        return "Sub";
    }
}

public class Main {
    public static void main(String[] args) {
        Base userType = new Sub();
        System.out.println(userType.getName()); // 輸出: Sub
    }
}
```

#### 動態綁定（Dynamic Binding）

Java 支援動態綁定，意味著方法的呼叫是根據物件的**實際類型**（在運行時確定）而非**引用類型**（在編譯時確定）來決定的。在上述範例中，`userType` 的引用類型是 `Base`，但實際指向的是 `Sub` 的實例。因此，當呼叫 `userType.getName()` 時，`getName` 方法內部的 `get()` 調用會根據實際類型 `Sub` 來決定，從而輸出 `"Sub"`。

這種行為使得 Java 能夠實現多型（polymorphism），允許相同的介面在不同的實作之間自由切換。

### Go 的靜態綁定

#### 靜態綁定（Static Binding）

在 Go 中，方法的綁定是**靜態的**，即方法的調用是根據變數的**靜態類型**（編譯時已知的類型）來決定的，而不是根據變數的**動態類型**（運行時的實際類型）。這意味著，除非通過介面進行調用，否則 Go 無法實現與 Java 類似的多型。

在上述範例中，為了實現類似 Java 的動態綁定效果，我們使用了介面 `Getter`。通過將 `Getter` 介面嵌入到 `Base` 結構中，並在 `NewSub` 函數中將 `Getter` 設置為 `Sub` 的實例，我們使得 `GetName` 方法能夠動態調用 `Sub` 的 `Get` 方法。這是通過介面的多型實現的，而不是語言本身的動態綁定。

### 為什麼 Go 選擇這種方式？

Go 採用「組合優於繼承」的理念，透過結構嵌入（struct embedding）和介面（interface）達到多型。這種做法避免了傳統類別複雜的繼承層次，使得程式碼更容易理解和維護。

此外，Go 採用靜態綁定（Static Binding），讓編譯器更容易對程式碼進行優化。然而，透過介面來實現的多型行為，開發者仍能夠在運行時靈活替換不同的行為實作。如此一來，即便沒有傳統繼承，Go 依然能提供足夠的彈性來滿足各種多型需求，並符合 Go 強調清晰性和明確性的設計哲學。

## 結論

抽象來看，UML 的 Generalization 通常用於表示傳統 OOP 中的繼承（is-a）關係；然而在 Go 中，物件的行為能力是被「**賦予**」而非來自「**血統**」，替代了過去自上而下的繼承架構。Go 透過「**組合（Composition）**」與「**介面（Interface**）」，將物件間的相依關係抽離至更抽象的層次，在精神層面上達到了 IoC 所追求的效果：控制權不再局限於固定的類別繼承架構中，而是透過替換不同的實作、組合出所需的行為。



## 參考文獻

[Effective Go - The Go Programming Language](https://go.dev/doc/effective_go#embedding)

[Go by Example: Interfaces](https://gobyexample.com/interfaces)

[Go Bootcamp - Everything you need to know to get started with Go (Matt Aimonetti)](https://archive.org/details/GoBootcamp)
