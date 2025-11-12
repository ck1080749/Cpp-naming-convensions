
# 🧱 C++ 設計實踐指南
> **目標：** 寫出安全、清晰、可維護、可擴充的 C++ 類別。
## 1️⃣ 基本設計原則（Core Principles）
### **1. 封裝性 (Encapsulation)**
* 盡可能將成員資料設為 `private` 或 `protected`。
* 提供明確、最小化的 public 介面。
* 使用 `const` 成員函式以保證不修改物件狀態。

```cpp
class Account {
private:
    double balance;

public:
    double getBalance() const { return balance; }
    void deposit(double amount);
};
```

### **2. RAII（Resource Acquisition Is Initialization）**

* 在建構子中取得資源，在解構子中釋放資源。
* 使用智慧指標 (`std::unique_ptr`, `std::shared_ptr`) 管理記憶體。

```cpp
class FileHandler {
    std::unique_ptr<std::FILE, decltype(&std::fclose)> file{nullptr, std::fclose};
public:
    explicit FileHandler(const char* path)
        : file(std::fopen(path, "r"), std::fclose) {}
};
```

### **3. Rule of Zero / Three / Five**

| 規則                | 意涵                                                           |
| ----------------- | ------------------------------------------------------------ |
| **Rule of Zero**  | 使用標準型別與智慧指標時，不需手動定義 destructor / copy / move。                |
| **Rule of Three** | 若類別需自訂 destructor，就應一併定義 copy constructor 與 copy assignment。 |
| **Rule of Five**  | 若支援 move semantics，也要定義 move constructor 與 move assignment。  |


### **4. 單一職責原則（Single Responsibility Principle）**

每個類別應只負責一個明確功能。
若類別變得太複雜，應拆分成多個子類別或協助類別。


### **5. 常數正確性（Const-Correctness）**

* 函式不修改狀態 → 標記為 `const`。
* 不會被修改的參數 → 傳 `const &` 或 `const *`。


## 2️⃣ 介面設計哲學（Interface Design Philosophy）

### **1. Minimal Public Interface**

* 公開最少必要的函式。
* 可使用 **pImpl idiom** 隱藏實作細節。

```cpp
class Widget {
public:
    Widget();
    ~Widget();
    void doSomething();
private:
    struct Impl;
    std::unique_ptr<Impl> pImpl;  // 隱藏內部實作
};
```

### **2. Prefer Composition over Inheritance**

* 優先使用「組合」（has-a）而非「繼承」（is-a）。
* 只有在需要多型時才使用繼承。

### **3. 小心使用多型（Polymorphism）**

* 若有虛擬函式，基底類別必須有 `virtual destructor`。
* 若能以模板或 `concept` 實現靜態多型，通常更高效。

### **4. Immutable 與 Thread Safety**

* 儘量讓物件不可變（immutable）。
* 若有多執行緒共享資料，應在類別內封裝同步機制。

## 3️⃣ 語言層級規範（Language-Level Rules）

### **1. 使用 `explicit` 關鍵字**

防止意外隱式轉換：

```cpp
explicit MyClass(int n);
```

### **2. 使用 `override` / `final`**

避免覆寫錯誤：

```cpp
class Derived : public Base {
    void foo() override;   // 編譯器會檢查 Base 有無 foo()
};
```

### **3. 避免裸指標持有資源**

* **擁有資源** → `unique_ptr` 或 `shared_ptr`
* **僅參考他人資源** → `T*` 或 `T&`

### **4. 命名風格建議**

| 元素   | 建議命名風格                                  |
| ---- | --------------------------------------- |
| 類別   | `PascalCase` (`MyClass`)                |
| 函式   | `camelCase` (`doWork()`)  或   `snake_case` (`do_work()`)           |
| 成員變數 | `m_` 前綴或尾底線（`m_count`, `count_`）        |
| 常數   | `ALL_CAPS_WITH_UNDERSCORE` (`MAX_SIZE`) |


## 4️⃣ 進階設計理念（Advanced Design Philosophy）

### **1. Value Semantics（值語義）**

* 類別應該能像基本型別一樣被安全複製、移動與比較。
* 若需要共享狀態，明確以指標封裝。

### **2. Safety before Performance**

* 先保證程式正確性，再做效能優化。
* 使用 `std::vector`、`std::string` 等標準容器取代裸陣列。

### **3. Interface Stability**

* Public API 一旦公開就不應輕易改動。
* 若需變更，可使用版本控制或 adapter pattern。

## 5️⃣ 推薦學習資源（Recommended References）

| 類別        | 資源                                                                                                             |
| --------- | -------------------------------------------------------------------------------------------------------------- |
| 標準風格      | [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html)                                    |
| 現代 C++ 準則 | [C++ Core Guidelines (Stroustrup & Sutter)](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines.html) |
| 書籍        | Scott Meyers — *Effective Modern C++*、*More Effective C++*                                                     |

## ✅ 總結（Quick Checklist）

| ✅ 檢查項目                           | 狀態 |
| -------------------------------- | -- |
| 使用封裝、最小化 public 介面               | ☐  |
| 遵循 RAII，避免裸 `new` / `delete`     | ☐  |
| 遵守 Rule of Zero / Three / Five   | ☐  |
| 標註 `const`、`explicit`、`override` | ☐  |
| 優先使用組合而非繼承                       | ☐  |
| 成員命名清晰、API 穩定                    | ☐  |
| 類別行為具值語義與例外安全性                   | ☐  |

