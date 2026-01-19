# Go 如何錯誤地使用 Method Overriding

### 前言

許多初學者在 Go 中嘗試覆寫（override）方法時，可能會遇到一些預期外的行為，如以下：

```go
package main

import "fmt"

// The abstract
type Base struct {
}

func (b *Base) Get() string {
    return "Base"
}

func (b *Base) GetName() string {
    return b.Get()
}

// The user implement
type Sub struct {
    Base
}

func (t *Sub) Get() string {
    return "Sub"
}

func NewSub() *Sub {
    return &Sub{}
}

func main() {
    userType := NewSub()
    fmt.Println(userType.GetName()) // output: Base
}
```

### 實際行為與預期不同

在上述程式碼中，我們預期 `GetName` 方法應該輸出 `"Sub"`，因為我們在 `Sub` 中覆寫了 `Get` 方法。然而，實際輸出卻是 `"Base"`。這是為什麼？

### 為什麼會這樣？

在 Go 中，method 的呼叫是基於**靜態類型**而非**動態類型**。讓我詳細解釋：

1. **method embedded 的 method 呼叫**：
   - `Sub` Struct embedded 了 `Base` ，這意味著 `Sub` 擁有 `Base` 的所有 method。
   - 當我們創建 `Sub` 的實例並呼叫 `GetName` method 時，實際上是呼叫 `Base` 的 `GetName` method。

2. **靜態類型決定 method 呼叫**：
   - 在 `GetName` 中，呼叫了 `b.Get()`。
   - 由於 `b` 的靜態類型是 `*Base`，因此 `b.Get()` 會呼叫 `Base` 的 `Get` method，而不是 `Sub` 的 method。

Go 中沒有傳統意義上的類別繼承。方法呼叫不會根據實例的動態類型進行分派，而是根據變數的靜態類型。這就是為什麼即使 `Sub` 覆寫了 `Get` 方法，呼叫 `GetName` 時仍然會使用 `Base` 的 `Get` 方法。

## 解決方案

如果真地想要在 Go 使用傳統物件導向語言的 method overriding 寫法，我們可以利用 interface 來達成，

以下是修正後的程式碼：

```go
package main

import "fmt"

// The interface
type Getter interface {
    Get() string
}

// The abstract
type Base struct {
    Getter
}

func (b *Base) Get() string {
    return "Base"
}

func (b *Base) GetName() string {
    return b.Getter.Get()
}

// The user implement
type Sub struct {
    Base
}

func (t *Sub) Get() string {
    return "Sub"
}

func NewSub() *Sub {
    userType := &Sub{}
    userType.Getter = interface{}(userType).(Getter)
    return userType
}

func main() {
    userType := NewSub()
    fmt.Println(userType.GetName()) // output: Sub
}
```

### 這是怎麼辦到的？

1. **定義 interface**：
   - 我們定義了一個 `Getter` 接口，其中包含 `Get` 方法。

2. **Struct embedding**：

   - 在 `Base` struct 中，嵌入 `Getter` 介面：
     ```go
     type Base struct {
         Getter
     }
     ```
   - 這使得 `Base` 擁有一個 `Getter` 介面的欄位，可以動態指向任何 `Getter`介面的實作。

3. **interface 實作**：

   - 使用者在 `Sub` 實作自己的  `Get` method：
     ```go
     func (t *Sub) Get() string {
         return "Sub"
     }
     ```

4. **初始化 Sub 並覆寫**

   - 在 `NewSub` 建構函式中，也必須給 `Getter` 介面指定 `Sub` 的實作：
     ```go
     func NewSub() *Sub {
         userType := &Sub{}
         userType.Getter = interface{}(userType).(Getter)
         return userType
     }
     ```
   -  `interface{}(userType).(Getter)` 是 Type Assersions，確保 `userType` 是 `Getter` interface 實作。

這樣，當呼叫 `GetName` 方法時，裡面的 `b.Getter.Get()` 呼叫的就會是 `Sub` 中的 `Get` method。


## 結論

希望這篇文章能幫助你更好地理解 Go 中的方法覆寫機制。等等，為什麼你標題寫這是一個錯誤的方法，難道有正確的方法？下一篇我會有深入的介紹。
