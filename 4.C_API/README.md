# 4.C API
Lua能夠輕易的與C語言並用是其受歡迎重要的原因之一。此章我們會討論如何使用Lua的C語言API。

## 狀態機
Lua語言的實現為一儲存現在狀態之狀態機(State)。
Lua和C語言故無法直接對彼此的變數做存取，但可以透過狀態機的查詢及新增狀態來實現兩者互通。

那什麼是狀態機呢？
狀態機為一個儲存各階段狀態之堆疊，最初始的狀態疊在第一層，後面狀態改變時會把狀態一層一層疊上去。
最上層的狀態為現在狀態，而整個堆疊會成為狀態歷史紀錄。
當函式執行完時，會將現在的狀態返回，然後由垃圾回收機制將堆疊回收。

下面舉個例子，假設今天我們設計了一個函式`total(i,j)`，函式的功能是返回變數i和j之加總。
那在狀態機中可以如何實現呢？以下我們用`total(1, 2)`為範例演示狀態機各步驟之狀態：

* 首先Lua會先新增一狀態機，將Lua的基本函式庫載入進來，然後將兩個引數載入進狀態機堆疊。
```
.---.
| 2 | <- 現在狀態
|---|
| 1 |
|___|
```
* 然後我們將現在狀態之值設置為變數j並pop掉。
```
.---.
| 1 | <- 現在狀態
|___|
```
* 我們接著把現在狀態之值設置為變數i並pop掉。此時狀態機已經空了。
* 接著我們要將i和j之值相加並返回，於是我們將i和j之總和被放上狀態機堆疊中，然後函式結束返回了現在狀態
```
.---.
| 3 | <- 現在狀態
|___|
```
於是執行`total(1,2)`我們得到了輸出3。

## 在C語言中操作Lua狀態機
在上一節中我們嘗試敘述狀態機在一函式中的運作流程。
在下面我們會先接續上面的範例來演示如何在C底下操作狀態機>

首先在C語言的程式下要使用Lua之狀態機，要先導入Lua的函式庫：
```c
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
```

接著我們就可以開始實現上面的範例了：
```c
int main(){
    lua_State* L=lua_newstate();
    luaL_openlibs(L);
    
    int i = 1;
    int j = 2;

    luaL_loadstring(L, "total = function(i,j) return i+j end");
    lua_pcall(L, 0, 0, 0);

    lua_getfield(L, LUA_GLOBALSINDEX, "total");
    lua_pushnumber(L ,i);
    lua_pushnumber(L ,j);
    lua_pcall(2, 1, 0);
    printf("i+j= %d\n", luaL_checkinteger(L, 1));
    lua_close(L);
    return 1;
}
```
一開始先開啟新的狀態機叫L，第二行載入Lua的函式庫。
接著在第三段我們在狀態機中載入了函數total()，第四段我們呼叫了前面載入的函式total。
我們先呼叫了函式total()，然後把i和j兩個變數放進狀態機堆疊中，接著我們執行了`lua_pcall()`。
`lua_pcall()`的第一個引數為從狀態機堆疊中擷取的層數，且擷取後會將用了的堆疊層給pop掉。
第二個引數代表會返回的結果數，這些返回的結果會放到狀態機堆疊中。
`luaL_checkinteger(L, 1)`會將堆疊層從底下數來的第1層之值返回給C語言程式，於是我們收到了Lua狀態機的返回值。
最後我們把Lua狀態機關掉，完成了這個範例。