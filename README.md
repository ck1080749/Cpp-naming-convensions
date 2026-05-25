# 🧱 C++ 設計實踐指南

> **目標：** 寫出安全、清晰、可維護、可擴充的 C++ 類別。

---

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

#### 容易踩雷的隱藏規則

* **特殊成員函式的「抑制」規則**：一旦宣告了 destructor（或任一 copy 操作），編譯器**不會**自動生成 move 操作，類別會默默退化成只能 copy。因此現代 C++ 中「只做到 Rule of Three」往往本身就是 bug 訊號 —— 你以為的 move 其實是 copy。
* 明確使用 `= default` 與 `= delete` 表達意圖，不要依賴隱式生成的有無：

```cpp
class Resource {
public:
    ~Resource();                                   // 自訂 dtor
    Resource(const Resource&) = delete;             // 明確禁止 copy
    Resource& operator=(const Resource&) = delete;
    Resource(Resource&&) noexcept = default;        // 明確要求 move
    Resource& operator=(Resource&&) noexcept = default;
};
```

* **moved-from 物件** 應處於「有效但未指定」的狀態：可被解構、可被重新賦值，但不該假設其內容。

### **4. 單一職責原則（Single Responsibility Principle）**

每個類別應只負責一個明確功能。
若類別變得太複雜，應拆分成多個子類別或協助類別。

### **5. 常數正確性（Const-Correctness）**

* 函式不修改狀態 → 標記為 `const`。
* 不會被修改的參數 → 傳 `const &` 或 `const *`。

### **6. 初始化（Initialization）**

* **成員初始化順序陷阱**：成員一定依「**宣告順序**」初始化，與 initializer list 的撰寫順序無關。

```cpp
class C {
    int b_;
    int a_;
public:
    C(int x) : a_(x), b_(a_) {}  // bug：b_ 先初始化，此時 a_ 尚未初始化
};
```

* **in-class member initializer** 可減少多建構子間的重複，並讓 Rule of Zero 更容易達成：

```cpp
class Config {
    int  retries_ = 3;
    bool verbose_ = false;
};
```

* **統一初始化 `{}`** 會阻擋 narrowing conversion，比 `()` 安全；同時也避開 *most vexing parse*：

```cpp
int x{3.14};        // 編譯錯誤（narrowing）；用 () 反而會靜默截斷
Widget w{};         // 明確是物件，不是函式宣告
```

---

## 2️⃣ 介面設計哲學（Interface Design Philosophy）

### **1. Minimal Public Interface**

* 公開最少必要的函式。
* 可使用 **pImpl idiom** 隱藏實作細節 —— 同時也將實作相依藏進 `.cpp`，降低編譯耦合並維持 ABI 穩定（見第 7 章）。

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
* 組合也是「可測試性」的基礎 —— 相依以組合方式注入，測試時才能替換（見第 7 章）。

### **3. 小心使用多型（Polymorphism）**

* 若有虛擬函式，基底類別必須有 `virtual destructor`。
* 若能以模板或 `concept` 實現靜態多型，通常更高效。
* **NVI（Non-Virtual Interface）idiom**：public 介面設為 non-virtual，真正可覆寫的點設為 private/protected virtual。基底類別可在呼叫前後插入不變式檢查、log、鎖等，子類別無法繞過。

```cpp
class Shape {
public:
    void draw() const { preDraw(); doDraw(); postDraw(); }  // 穩定的 public 介面
private:
    virtual void doDraw() const = 0;                        // 子類別客製點
};
```

* **不要在建構子／解構子中呼叫 virtual 函式** —— 此時動態型別仍是（或已退回）基底，不會分派到子類別。
* **物件切片（slicing）**：以 by-value 傳遞多型物件會丟失子類別部分。多型物件一律以 reference 或（智慧）指標傳遞。

### **4. Immutable 與 Thread Safety**

* 儘量讓物件不可變（immutable）。
* 若有多執行緒共享資料，應在類別內封裝同步機制（詳見第 6 章）。

---

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

#### 智慧指標使用細則

* 一律用 `std::make_unique` / `std::make_shared`，不要裸 `new`（例外安全 + 較少配置）。
* **預設用 `unique_ptr`**；只有當所有權「真的共享」時才用 `shared_ptr`（後者有原子計數開銷）。
* `shared_ptr` 的循環參照會造成記憶體洩漏，用 `weak_ptr` 打破環（如 parent/child、observer）。
* 參數傳遞約定：要轉移所有權才傳 `unique_ptr`（by value）；只是觀察就傳 `T*` / `T&` / `string_view`，別傳 `const shared_ptr&`。

### **4. 非擁有檢視（Non-owning Views）**

`std::string_view`、`std::span`（C++20）作為**唯讀參數**型別，避免不必要的複製與多重重載：

```cpp
void parse(std::string_view text);        // 接受 std::string、const char*、字面值，零複製
void process(std::span<const int> data);  // 接受 vector、array、C 陣列
```

> ⚠️ 它們**不擁有**資料，不可儲存成成員，也不可回傳指向暫存物件的 view，否則 dangling。

### **5. 型別安全的現代特性**

* **`enum class` 取代 plain `enum`**：有作用域、不隱式轉成整數、可指定底層型別。
* **`nullptr` 取代 `NULL` / `0`**：避免在重載解析時被當成整數。
* **`[[nodiscard]]`**：標在「回傳值不該被忽略」的函式或型別上（如 error code、`expected`、工廠函式），呼叫端忽略時編譯器會警告。
* **三向比較 `<=>`（C++20）**：`auto operator<=>(const T&) const = default;` 一行生成全部六個比較運算子（對應第 5 章 Value Semantics 的「可比較」要求）。

### **6. 命名風格建議**

| 元素   | 建議命名風格                                  |
| ---- | --------------------------------------- |
| 類別   | `PascalCase` (`MyClass`)                |
| 函式   | `camelCase` (`doWork()`) 或 `snake_case` (`do_work()`) |
| 成員變數 | `m_` 前綴或尾底線（`m_count`, `count_`）        |
| 常數   | `ALL_CAPS_WITH_UNDERSCORE` (`MAX_SIZE`) |

---

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

### **4. 編譯期計算（Compile-time Computation）**

把能在編譯期決定的東西移到編譯期，可同時提升執行效能與型別安全（編譯期錯誤勝於執行期錯誤）。三個關鍵字的分工：

| 關鍵字              | 意涵                                                      |
| ---------------- | ------------------------------------------------------- |
| **`constexpr`**  | 「**可以**」在編譯期求值；若情境提供常數引數就在編譯期算，否則退回執行期。最常用、最寬鬆。 |
| **`consteval`** (C++20) | 「**必須**」在編譯期求值，否則編譯失敗。適合保證零執行期成本的工廠函式或編譯期檢查。 |
| **`constinit`** (C++20) | 不保證 const，但保證以常數初始化。消滅 *static initialization order fiasco*。 |

```cpp
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}
constexpr int kTableSize = factorial(5);   // 編譯期算出 120
int arr[kTableSize];                       // 可當陣列大小

constinit int gCounter = compute();        // 保證編譯期初始化，無 SIOF 風險
```

* 常數一律優先用 `constexpr` 取代 `const`（意圖更明確、可用於需要 constant expression 的場合）。
* C++20 起 `if constexpr` 可在編譯期分支，取代許多 SFINAE 與標籤分派的樣板程式碼。

---

## 5️⃣ 例外安全與錯誤處理（Exception Safety & Error Handling）

### **1. 例外安全的三種保證等級**

| 等級                  | 意涵                                                 |
| ------------------- | -------------------------------------------------- |
| **Basic guarantee**  | 拋例外後物件仍處於有效狀態（不洩漏、不損毀不變式），但具體狀態未指定。      |
| **Strong guarantee** | 操作要嘛完全成功，要嘛完全不發生（commit-or-rollback）。      |
| **Nothrow guarantee**| 保證不拋例外，用 `noexcept` 標記。                    |

### **2. `noexcept` 不只是文件標記**

`std::vector` 在 reallocation 時，只有當元素的 move constructor 是 `noexcept` 才會用 move，否則為維持 strong guarantee 會退回成 **copy**。沒標 `noexcept` 的 move ctor 會讓型別在容器中默默喪失移動效能。

```cpp
class Buffer {
public:
    Buffer(Buffer&&) noexcept;             // move 操作幾乎必須是 noexcept
    Buffer& operator=(Buffer&&) noexcept;
};
```

### **3. Copy-and-Swap Idiom**

達成 strong guarantee 的經典手法，同時自然處理 self-assignment，copy 與 move assignment 一次搞定：

```cpp
Widget& operator=(Widget rhs) noexcept {  // by value：複製在進函式前完成
    swap(*this, rhs);                     // swap 不拋例外
    return *this;                         // 舊資源隨 rhs 解構釋放
}
```

### **4. 何時用例外、何時不用**

| 機制                          | 適用情境                                                |
| --------------------------- | --------------------------------------------------- |
| **例外 (exception)**           | 違反前置條件、無法在當地處理的錯誤、建構子失敗（建構子無法回傳錯誤碼）。   |
| **`std::optional<T>`**       | 表達「可能沒有值」這種非錯誤情況（如查表 miss）。              |
| **`std::expected<T, E>`** (C++23) | 需回傳「值或錯誤原因」且不想用例外時的型別安全選項，取代 out-param + error code。 |
| **error code**               | hot path 或 `noexcept` 邊界內，通常比例外合適。            |

> 原則：同一個 API 層級應風格一致，不要混用。

---

## 6️⃣ 並行與執行緒安全（Concurrency & Thread Safety）

### **1. `const` 的並行語意**

現代 C++ 中 `const` 成員函式**隱含「可被多執行緒並行呼叫而安全」**的承諾，標準函式庫即以此為假設。

當用 `mutable` 成員做快取（const 函式內部寫入），就破壞了這個承諾，必須自行補上同步：

```cpp
class ExpensiveQuery {
    mutable std::mutex mtx_;
    mutable std::optional<Result> cache_;
public:
    Result get() const {                    // 標示 const，故須對並行呼叫安全
        std::scoped_lock lock(mtx_);         // 保護 mutable 快取
        if (!cache_) cache_ = computeSlow();
        return *cache_;
    }
};
```

### **2. 鎖的使用守則**

* **鎖優先用 `std::scoped_lock`**（C++17），不要裸用 `lock()` / `unlock()` —— RAII 保證例外路徑也會解鎖，且可一次鎖多個 mutex 並避免死鎖。
* **不要在持鎖狀態下呼叫使用者 callback 或 virtual 函式**：對方若回頭存取你的物件即死鎖。把資料複製出臨界區後再呼叫。
* **臨界區越小越好**，只包住真正需要保護的共享狀態存取。
* 把同步機制**封裝在類別內**，不要讓呼叫端負責加鎖（呼叫端忘記加鎖是最常見的並行 bug，也違反封裝）。
* 唯讀為主、偶爾寫入的共享資料可用 `std::shared_mutex`；單純旗標或計數器用 `std::atomic` 取代 mutex。

---

## 7️⃣ 工程實踐：標頭檔與可測試性（Engineering Practices）

### **1. 標頭檔衛生（Header Hygiene）**

標頭檔設計直接影響**編譯時間**與**耦合度**，是大型專案維護成本的主要來源之一。

* **Include guard**：統一用 `#pragma once`；需嚴格可攜性才用傳統 `#ifndef` 巨集。
* **能 forward declare 就不要 include**：標頭中只用到指標或參考時，前向宣告即可。

```cpp
// widget.h
class Logger;            // 前向宣告就夠了
class Widget {
    Logger* logger_;     // 只用到指標，不需 Logger 的完整定義
public:
    void setLogger(Logger& l);
};
// Logger 的完整 #include 留到 widget.cpp
```

需要完整定義的時機：作為**值成員**、需呼叫其成員函式、繼承它、或 `sizeof` 它。
* **標頭檔絕不寫 `using namespace`**：會污染所有引入此標頭的檔案且無法收回。`using` 指令只能出現在 `.cpp` 或函式作用域內。
* **Include-what-you-use（IWYU）**：每個檔案直接 include 它自己用到的東西，不依賴遞移 include。
* **自我滿足性（self-containment）**：每個標頭應能單獨被 include 而成功編譯；慣例上把對應標頭放在 `.cpp` 的第一個 include 來驗證這點。

### **2. 可測試性（Testability）**

難以測試的類別，通常是相依過度耦合或職責不清的訊號 —— 與第 2 章「組合優於繼承」「單一職責」同一脈絡。

* **依賴注入（Dependency Injection）**：類別不該自己 `new` 出相依物件，而應透過建構子接收。

```cpp
// 難測試：相依硬寫死，測試時無法替換真實資料庫
class OrderService {
    Database db_;                          // 內部自建
};

// 易測試：相依由外部注入
class OrderService {
    IDatabase& db_;                        // 介面 + 注入
public:
    explicit OrderService(IDatabase& db) : db_(db) {}
};
```

* **抽出純粹的核心邏輯**：把計算/決策邏輯與 I/O、時間、亂數、全域狀態這些「難測試的邊界」分離。純函式最容易測試（呼應第 5 章 Value Semantics）。
* **避免單例與全域狀態**：它們讓測試互相污染、無法平行執行、難以設定初始條件。
* **不確定性來源也要可注入**：時鐘、UUID、亂數產生器都包成介面注入，測試才能餵入固定值得到可重現結果。
* **介面不要過胖**：小而專注的介面（對應單一職責）既容易 mock，也減少測試的脆弱性。

> 判準：**如果一個類別很難寫單元測試，那通常是設計需要重構的徵兆。**

---

## 8️⃣ 推薦學習資源（Recommended References）

| 類別        | 資源                                                                                                             |
| --------- | -------------------------------------------------------------------------------------------------------------- |
| 標準風格      | [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html)                                    |
| 現代 C++ 準則 | [C++ Core Guidelines (Stroustrup & Sutter)](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines.html) |
| 書籍        | Scott Meyers — *Effective Modern C++*、*More Effective C++*                                                     |


