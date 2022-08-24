---
title: Lock-free programming
date: 2022-08-18 09:00:00
tags:
  - c++
  - multi-thread
  - lock-free
  - optimization
categories:
  - notes
keywords:
  - programming
---

## 前言

最近為了某個專案稍微複習了一下multi-thread的同步控制，除了使用簡單的spin-lock之外，還有更為強大的lockless演算法，如同前輩們所告誡的，看起來越是酷炫的東西，背後隱藏的陷阱也越多，本文是讀完[Lockless Programming Considerations for Xbox 360 and Microsoft Windows](http://msdn.microsoft.com/en-us/library/ee418650%28v=vs.85%29.aspx)所寫的心得，好文章當然得作點筆記以供日後參考

<!--more-->

如同大家所知道的，解決多緒程式同步問題最簡單的方法就是使用lock，例如有個thread之間共用的函式ManipulateSharedData()，一次只能由一個thread執行，我們可以簡單地用CRITICAL_SECTION達成：

``` C
// 初始化
CRITICAL_SECTION cs;
InitializeCriticalSection(&cs);

// 呼叫
void ManipulateSharedData()
{
  EnterCriticalSection(&cs);
  // 做該作的事...
  LeaveCriticalSection(&cs);
}

// 結束
DeleteCriticalSection(&cs);
```

非常直覺也很簡單易瞭，一眼就能看出邏輯的正確性，但是有隱藏的缺點；試想：假如有兩個thread試圖取得同樣兩個lock，但是以不同順序，例如：thread A取得了lock 1，而thread B取得了lock 2，這時死結就產生了，因為不論是thread A或是B都需要lock 1和2才能進入critical section，除此之外，如果因為某個thread持有lock的時間過長，也會令別的thread處於饑餓狀態，大大地降低整體程式執行的效能

除去以上問題，基本上critical section仍是處理多緒程式同步問題的好方法，但是對於效能執著而又有能力處理更為複雜情況的程式設計師而言，可以選擇lockless programming，對於選擇此路的程式設計師而言，主要面對的難題有兩項：`non-atomic操作`和`指令序重排`

## Non-atomic操作

首先定義一下什麼叫作atomic操作，一個atomic操作代表一個不能再被分割成更小步驟的程式碼，這保證一個thread絕對不會看到這個程式碼執行到一半時的狀態，atomic操作可以說是lockless programming的基礎，有了atomic操作才能確保thread不會去存取某個寫到一半的變數

在現今的CPU上你基本可以假設存取對齊其native定址寬度的資料是atomic操作，例如在32bit模式下存取某個在4-bytes-aligned位址的資料是atomic操作，在64bit模式下存取某個在8-bytes-aligned位址的資料是atomic操作，但是超過這個長度的資料，即使使用字串搬移指令或是SIMD指令操作16-bytes資料，那就不一定是atomic操作

對於atomic操作在 **Intel 64 and IA-32 Architectures Software Developer's Manual** 第三冊中有詳細解釋

對於Intel 486或更新的機器，以下操作保證為atomic：

* Reading or writing a byte
* Reading or writing a word aligned on a 16-bit boundary
* Reading or writing a doubleword aligned on a 32-bit boundary

對於Pentium或是更新的機器，以下操作保證為atomic：

* Reading or writing a quadword aligned on a 64-bit boundary
* 16-bit accesses to uncached memory locations that fit within a 32-bit data bus

對於P6或是更新的機器，以下操作保證為atomic：

* Unaligned 16-, 32-, and 64-bit accesses to cached memory that fit within a cache line

除了以上所列情況，存取非對齊於native定址寬度的資料有可能不是atomic操作，因為CPU可能需要對記憶體匯流排進行多次存取，所以thread有可能會在其過程中寫入／讀取到一半的資料。

一些組合式指令，所謂的 **讀取-修改-寫入** ，不一定會是atomic操作，在x86或是x64上一般是指 **inc** 或是 **dec** 這類指令，這些指令在單一CPU的系統上是atomic操作，但是在多CPU上則必需加上lock前置才會保證其它CPU不會同時間也進行這個指令

``` C
// 這個寫入不是atomic操作，因為它不是native aligned
UINT32 *pData = (UINT32 *)(pChar + 1);
*pData = 0;

// 這也不是atomic操作，因為它包含了三個動作 (讀取-修改-寫入)
++g_globalCounter;

// 這很明顯是atomic操作
g_alignedGlobal = 0;

// 這個也是atomic操作
UINT32 local = g_alignedGlobal;
```

## 如何保證指令的不可切割性

* 使用 *天然的* atomic操作 (例：單一讀取／寫入native-aligned位址的資料)
* 使用lock將一組非atomic操作包裝起來 (這不是又回到前面一開始的問題了嗎？)
* 使用作業系統提供的API (所以你多半只能選擇這條路啦！)

舉個例子，在Windows系統中提供了一組atomic版本的 **讀取-修改-寫入** API，就是有名的 **InterlockedXxx** 家族，如果你使用了這組API，即可保證你存取的thread共享變數是安全的

``` C
// 這不是atomic操作，理由同上
++g_globalCounter;

// 這樣就安全了，使用InterlockedXxx家族準沒錯
InterlockedIncrement(&g_globalCounter);
```

## 亂序執行

另一個令人困擾的問題就是指令的順序被改變。基於某些原因，記憶體的存取不一定是照你的程式所寫的順序進行，這常常會發生許多令人費解的問題。某些多緒程式演算法規定在寫入資料到目的地之後，要設定一個flag用來指示資料已經備妥，這有個術語叫作 **write-release** 。如果寫入動作的順序被重排，別的thread就有可能在資料備妥以前就讀到設定好的flag狀態。

同樣的，某thread在讀取資料前必需先讀取一個flag，該flag指示某thread該資料已經備妥，這個叫做 **read-acquire** ，如果讀取動作的順序被重排，則某thread就有可能讀到尚未更新完成的資料。

一個簡單例子：

``` C
// 在(x, y)產生一個新的sprite，然後設定它的'alive'旗標，如果
// (x, y)在完成寫入前，'alive'就先被寫入，程式就會產生問題
g_sprite[nextSprite].x = x;
g_sprite[nextSprite].y = y;
g_sprite[nextSprite].alive = true;
```

底下的code是另一個thread從g_sprite陣列中讀取資料：

``` C
// 畫出所有的sprite，如果(x, y)在'alive'被讀取前先被讀取，
// 則程式也會出問題
for (int i = 0; i < numSprite; ++i)
  if (g_sprite[i].alive)
    DrawSprite(g_sprite[i].x, g_sprite[i].y);

```

為了確保指令執行時順序的正確，我們需要先了解一下CPU和compiler

## x86和x64處理器的亂序執行

以寫入順序來說，在x86和x64的機器上，除了MTRR被設定為 **write-combined** 的區域，還有字串搬移指令和128-bits的SSE寫入指令之外，寫入的順序基本上是不會重新排序的

以讀取順序來說，除了字串搬移指令和128-bits的SSE讀取指令外，讀取的順序基本上也是不會被重排的

雖然讀取和讀取，寫入和寫入之間的順序不會被重排，但是讀取和寫入之間的順序是有可能被重排的，尤其是當讀取的來源和寫入的目標不一樣的時候，這種亂序執行會破壞某些演算法，像是著名的Dekker互斥鎖演算法。在Dekker演算法中，每一個thread都要先設定一個flag表示它想進入一個critical region，同時檢查另一個thread的flag，看它是否也在／正要進入critical region

``` C
volatile bool f0 = false;
volatile bool f1 = false;

void P0Acquire()
{
  // 表示正要進入critical region
  f0 = true;
  // 檢查其它thread是否正在或是正要進入critical region
  while (f1) {
    // 資源佔用中，看要等待或是清除flag離開
  }
  // 進入critical region
}

void P1Acquire()
{
  // 表示正要進入critical region
  f1 = true;
  // 檢查其它thread是否正在或是正要進入critical region
  while (f0) {
    // 資源佔用中，看要等待或是清除flag離開
  }
  // 進入critical region
}
```

這裡的問題在於，在P0Acquire()中，當f0被寫入前，f1就先被讀取，同一時間在P1Acquire()中，f1被寫入前，f0就先被讀取，所以最後的結果就是兩個thread都把自己的flag設成true，而且它們看到對方的flag都是false，所以兩個thread就一起進入critical region。由此可知Dekker演算法在沒有支援硬體memory barriers的機器上是不可行的。

在x86和x64的機器上，write不會重排序在上一個read之前(也就是說如果是先read再write，則這個write保證在read之後)，它只會將read重排序在上一個write之前(假如read和write的來源和目的地不一樣)。在PowerPC上則是read和write都可以互相重排序，只要來源和目的地不同

## Read-Acquire和Write-Release barriers

為了避免讀取和寫入的指令順序被重排，我們需要一種 **籓籬** 般的機制用以隔離關鍵的讀寫動作，如之前所述，read-acquire是一個thread在讀取某個共享資料之前先確認某個flag的狀態，這個動作可以視為thread對資料 **取得** 擁有權，如果在此指令 **之後** 加上一個barrier即可確保其餘的資料存取會在這個讀取之後；同樣的，一個write-release是某個thread設定一個flag，表示其對資料 **釋出** 所有權，如果在此指令 **之前** 加上一個barrier即可確保所有的資料存取在這個寫入前都已經完成

Herb Sutter曾經對此機制下過正式定義：

* 一個read-acquire必須在其thread中存取資料前執行，以使其之後的資料存取按照程式順序進行
* 一個write-release必須在其thread中存取資料後執行，以確保其之前的資料存取皆已按程式順序完成

一個簡單的例子說明：

``` C
// 測試一個flag (進行read-acquire)
if (g_flag) {
  // 這是一個barrier，確保所有的資料存取皆在
  // flag被讀取之後才按照程式順序進行
  BarrierOfSomeSort();

  // 現在可以安心存取共享資料
  int localVariable = sharedData.y;
  sharedData.x = 0;

  // 這是一個barrier，確保所有的資料存取皆已
  // 在flag被設定之前按照程式順序完成
  BarrierOfSomeSort();

  // 設定flag (write-release)
  g_flag = false;
}
```

關於x86和x64上的barrier實作，可以參考`lfence`和`sfence`指令，一般情況下可以使用作業系統或是編譯器所提供的API

## 防止編譯器重排指令順序

最大化的將程式原始碼最佳化是編譯器重要的任務之一，重新排列機器指令也是程式最佳化的方法之一，以C++來說，由於C++標準並沒有規範多緒程式，所以編譯器總是假設你的程式碼是單一執行緒，所以告知編譯器程式中的哪些部份指令是不能被重排的責任就落到寫程式的人身上了

在VC上可以透過使用編譯器intrinsic函式 **_ReadWriteBarrier()** 達成禁止編譯器重排指令的目的，只要在程式插入 **_ReadWriteBarrier()** ，不論是寫入或是讀取都不會跨越這道界線

``` C
#if _MSC_VER < 1400
  // 抱歉，如果你用的是VC2003，你得自行宣告_ReadWriteBarrier
  extern "C" void _ReadWriteBarrier();
#else
  // 如果是VC2005之後的版本，你可以直接include標頭檔intrin.h
  #include <intrin.h>
#endif
// 告訴編譯器這是一個intrinsic函式，不是一般函式
#pragma intrinsic(_ReadWriteBarrier)

// 創造一個新的sprite，設定初值
g_sprites[nextSprite].x = x;
g_sprites[nextSprite].y = y;

// 底下是一個write-release，在write-release之前設置一個barrier
// 這樣就可以保證在write-release之前所有對共享資源的存取皆會按照
// 程式邏輯順序完成
_ReadWriteBarrier();
g_sprites[nextSprite].alive = true;
```

底下的程式是另一個thread從sprite陣列中讀取資料

``` C
// 畫出所有的sprite
for (int i = 0; i < numSprites; ++i) {
  // read-acquire，接下來是一個barrier
  if (g_sprites[nextSprite].alive) {
    // 這個barrier可以確保所有接下來的資料存取都是在flag
    // 先被讀取之後才進行
    _ReadWriteBarrier();
    DrawSprite(g_sprites[i].x, g_sprites[i].y);
  }
}
```

這裡要注意的是 **_ReadWriteBarrier()** 並不會讓編譯器插入任何額外機器碼，因此也不能防止處理器進行亂序執行，它僅僅只是告訴編譯器不要在程式最佳化的時候將指令順序弄亂了，因此在x86或是x64的機器上 **_ReadWriteBarrier()** 已經足夠讓我們實作write-release barrier(因為x86或x64的機器不會將read-write順序重排，所以一般的write就已經可以安全release一個flag)，但是在許多情況下我們還是需要防止處理器對記憶體存取的指令進行亂序執行

你也可以使用 **_ReadBarrier()** 或是 **_WriteBarrier()** 進行更為精準的控制，編譯器不會讓read指令超過 **_ReadBarrier()** ，同樣的也不會讓write指令超過 **WriteBarrier()**

## 防止CPU將指令重排序

CPU的亂序執行比編譯器的指令重排序要更為複雜，你甚至不會直接看到它是怎麼發生的，你只會看到一些無法解釋的靈異現象，為了關閉硬體本身自有的功能，你只能透過硬體提供的方式，也就是memory barrier相關指令，在Windows上這些指令被包裝成一個巨集 -- **MemoryBarrier**

除了使用 **MemoryBarrier** 之外，利用指標也是個避免CPU進行亂序處理的技巧，具體做法是這樣的：如果你先讀取一個指標，然後再透過這個指標去存取資料，則處理器保證絕對不會先去存取資料再來讀取指標，所以如果你的flag是一個指標，而且你的共享資源也都是以指標形式進行讀取，那麼就可以不需要 **MemoryBarrier** ，從而大幅提升程式效能

``` C
Data *localPointer = g_sharedPointer;
if (localPointer)
{
  // 這裡不需要barrier，原因是底下的讀取是透過localPointer，而讀
  // 取localPointer絕對會先於透過它才能讀取其它資料的動作
  int localVariable = localPointer->y;
  // 這裡就需要一個memory barrier了，因為要防止讀取g_global的動
  // 作被排在讀取g_sharedPointer之前
  int localVariable2 = g_global;
}
```

**MemoryBarrier** 指令只針對使用快取的記憶體範圍有效，如果你allocate出來的記憶體是`PAGE_NOCACHE`或是`PAGE_WRITE_COMBINE`，那麼 **MemoryBarrier** 基本上是沒有效果的，大多數情況我們並不需要去同步化非快取的記憶體

## Interlocked函式和CPU的亂序執行

在Windows上通常可以使用 **InterlockedXxx** 之類的函式進行對資源存取的acquire或release， **InterlockedXxx** 家族非常安全，他們本身就被設計成內建read-acquire barrier或是write-release barrier

## volatile變數和亂序執行

C++標準說，讀取一個volatile變數是不可以從快取中取得的，寫入一個volatile變數則是不可被延遲的(non-write-back)，而且volatile變數的讀取和寫入順序是不可改變的，這在硬體和軟體的溝通上是足夠了，這也是volatile這個關鍵字被設計的用途，然而這在多緒程式設計上還是有所不足，因為C++並沒有規定編譯器不能重排non-volatile變數的存取和volatile變數的存取之間的順序，對於CPU的亂序執行也沒有提到

然而VC2005則自行對volatile語義加以延伸，使其對讀取一個volatile變數加上read-acquire語義，寫入一個volatile變數加上write-release語義，所以編譯器不會擅自在這些操作volatile變數時重排讀取或是寫入指令，這個功能只有在VC2005和之後的版本才有提供

## 範例：使用lock-free的data pipe

pipe是一種讓thread之間互相溝通的機制，一個lock-free版本的pipe可以成為一個thread間非常有效率的溝通方式，在DirectX SDK中就提供了一個 **LockFreePipe** ，一個單一讀取，單一寫入的pipe，宣告在DXUTLockFreePipe.h

**LockFreePipe** 可以用在當兩個thread間有生產者／消費者關係的時候，扮演生產者的thread可以在pipe的一端寫入資料，讓扮演消費者的thread在稍後處理，這個過程中不需要用到block機制，假如pipe滿了，寫入會失敗，生產者會晚點再嘗試寫入，這只會發生在生產者處理的速度超過消費者的時候；假如pipe空了，讀取會失敗，消費者會晚點再嘗試讀取，這只會發生在沒有生產者產生資料給消費者的時候，在運作良好的情況下，pipe會維持平衡狀態，資料會平順地增加和消耗而沒有任何延遲或阻塞

## Windows作業系統上的效能

在Windows上執行同步指令和函式的效能非常仰賴處理器的種類和設定，還有其餘正在執行的程式碼，多核心和多實體CPU的系統要花更久的時間進行同步指令，假如別的thread正擁有一個lock，取得lock會花更長的時間

一些簡單測試產生的數據：

* **MemoryBarrier** 大約需要20-90個週期
* **InterlockedIncrement** 大約需要36-90個週期
* 取得／釋放一個critical section大約需要40-100個週期
* 取得／釋放一個mutex大約需要750-2500個週期

這些數據來自在WindowsXP上使用各種CPU測試的結果，較少的時間來自單一處理器機器，較多的時間則是在多處理器機器上的結果，由數據可知取得／釋放locks比起使用lock-free programming的成本更加昂貴，但是最好的做法還是儘量減少共享資料吧

## 效能上的考量

最好可以自己實作一個critical section版本，因為使用回圈等待解鎖會非常浪費效能，可以試著組合 **memmory barrier** 和 **InterlockedXxx** 家族函式，加上遞迴檢查，如果有需要，使用mutex；如果一個critical section的衝突量很高，但是處理時間不會太長，則可以考慮使用 **InitializeCriticalSectionAndSpinCount** ，這樣如果critical section正被別的thread占用，作業系統會稍微進行spin loop(依照傳入的參數)等待，而不會立刻發出一個mutex，當然你得先自行衡量一下loop的次數要給多少比較合適

假如在thread間使用共享的heap，每一次的配置和釋放記憶體都會需要lock，如果thread的數量變多，或是配置的數量變多，效能終究會開始明顯下降，所以使用thread獨有heap，或是減少配置的次數，都可以降低使用lock造成的效能瓶頸

如果某個thread一直產生資料，而另一個一直消費這些資料，這會使兩個thread間的資料分享次數過於頻繁，例如一個thread從磁碟上讀取圖形資料，另一個thread則將圖形畫至螢幕上，如果繪圖的thread在每次繪圖函式被呼叫時就執行一次，lock造成的成本就會太高，像這種情況較好的辦法是讓資料只在畫格切換的時候才做同步
