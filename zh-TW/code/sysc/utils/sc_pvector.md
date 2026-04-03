# sc_pvector - 指標向量

## 概述

`sc_pvector` 是一個基於 `std::vector` 的簡單指標向量包裝，提供了一些額外的便利操作如排序、自動擴展等。它是 SystemC 內部使用的舊式容器，與 IEEE 1666 標準的 `sc_vector` 是完全不同的類別。

**來源檔案**：`sysc/utils/sc_pvector.h`（僅標頭檔）

## 生活比喻

想像一個可伸縮的名片夾：
- 可以在最後面插入新名片（`push_back`）
- 可以直接翻到第 N 張（`operator[]`）
- 如果你翻到第 100 張但名片夾只有 50 格，它會自動擴展到 101 格
- 可以整理排序（`sort`）
- 可以清空重來（`erase_all`）

## 類別介面

```cpp
template<class T>
class sc_pvector {
public:
    typedef const T* const_iterator;
    typedef T* iterator;

    sc_pvector();
    sc_pvector(const sc_pvector<T>& rhs);
    ~sc_pvector();

    std::size_t size() const;
    iterator begin();
    iterator end();
    const_iterator begin() const;
    const_iterator end() const;

    T& operator[](unsigned int i);       // 自動擴展
    const T& operator[](unsigned int i) const;

    T& fetch(int i);                     // 不擴展
    const T& fetch(int i) const;

    T* raw_data();                       // 取得底層陣列指標
    const T* raw_data() const;

    void push_back(T item);
    void erase_all();
    void sort(CFT compar);              // C 風格排序

    void put(T item, int i);            // 直接設定第 i 個元素
    void decr_count();                  // 移除最後一個元素
    void decr_count(int k);             // 移除最後 k 個元素

protected:
    mutable std::vector<T> m_vector;
};
```

## 設計特點

### 自動擴展的 operator[]

```cpp
T& operator[](unsigned int i) {
    if (i >= m_vector.size()) m_vector.resize(i+1);
    return m_vector[i];
}
```

與標準 `std::vector::operator[]` 不同，`sc_pvector` 會自動擴展到所需大小。這避免了越界錯誤，但也可能隱藏了邏輯 bug。

### C 風格排序

```cpp
typedef int (*CFT)(const void*, const void*);
void sort(CFT compar) {
    qsort(&m_vector[0], m_vector.size(), sizeof(void*), compar);
}
```

使用 C 標準庫的 `qsort`，因為這個類別主要用於指標類型的排序。

### mutable 內部向量

```cpp
mutable std::vector<T> m_vector;
```

宣告為 `mutable` 是因為 `const` 版本的 `operator[]` 也可能觸發 `resize`（自動擴展語意）。

### 迭代器是裸指標

```cpp
typedef const T* const_iterator;
typedef T* iterator;
```

迭代器直接使用指向底層 `std::vector` 資料的裸指標，因為 `std::vector` 保證連續記憶體配置。

## 與 sc_vector 的差異

| 特性 | `sc_pvector` | `sc_vector` |
|------|-------------|-------------|
| 標準支援 | 內部使用 | IEEE 1666 標準 |
| 繼承 `sc_object` | 否 | 是 |
| 自動命名 | 否 | 是 |
| 批次綁定 | 否 | 是 |
| 型別 | 通用 | 必須是 `sc_object` 子類別 |

## 相關檔案

- [sc_vector.md](sc_vector.md) — IEEE 1666 標準的具名物件向量
- [sc_list.md](sc_list.md) — 鏈結串列容器
