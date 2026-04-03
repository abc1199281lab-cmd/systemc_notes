# sc_iostream - 可攜式 I/O 串流標頭

## 概述

`sc_iostream.h` 是一個簡單的 I/O 串流標頭包裝檔案，統一引入常用的 C++ 標準 I/O 標頭。在 SystemC 早期，不同編譯器對 C++ 標準函式庫的支援程度各異，這個檔案的作用是抽象化這些差異。

**來源檔案**：`sysc/utils/sc_iostream.h`（僅標頭檔）

## 現況

這個檔案目前已被標記為 **deprecated**（棄用），因為所有現代 C++ 編譯器都已正確支援標準 I/O 標頭。它現在只是簡單地引入以下標頭：

```cpp
#include <iostream>
#include <sstream>
#include <fstream>
#include <cstddef>
#include <cstring>
```

## 生活比喻

想像在不同國家的插座轉接頭：早年各國插座規格不同，你需要一個萬用轉接頭。現在大多數國家都支援 USB-C，轉接頭變得不再必要，但為了相容舊設備還是保留著。

`sc_iostream.h` 就是這樣的「轉接頭」——曾經很重要，現在只是為了不破壞舊程式碼而保留。

## 相關檔案

- [sc_string.md](sc_string.md) — 使用此標頭的字串工具
