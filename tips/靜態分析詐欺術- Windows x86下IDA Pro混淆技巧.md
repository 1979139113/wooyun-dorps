# 靜態分析詐欺術: Windows x86下IDA Pro混淆技巧

**馬聖豪aaaddress1@gmail.com**

0x00 前言
=====

此文為我在台灣駭客年會2015社群場中以TDOHacker社群名義發表的「欺騙IDA Pro Hex Rays插件！讓逆向分析者看見完全不同的結果 」（[http://hitcon.org/2015/CMT/agenda](http://hitcon.org/2015/CMT/agenda)）這邊把我在HITCON上發表的一些觀念、技巧整理成簡單入門的文章方便以後的人想學比較好入門，拓展了些有趣的觀念。

以前分析惡意軟體、破解企業產品、找產品弱點⋯⋯總是得動態分析追組合語言指令並且對系統架構、API有很深的概念才能深入分析出一支軟體架構、功能、手法；但近日各式靜態分析工具（大名鼎鼎如IDA Pro）崛起，使數位鑑識、軟體破解、逆向工程⋯⋯等的門檻降低許多（令許多破解者舒適的F5功能）

但其實靜態分析工具雖是依照真實程序內組合語言狀況來做反組譯「推論」出原本C語言狀況下可能是如何開發的，那麼既然是透過「推論」來還原原始程式碼狀況，那必定靜態分析工具會對一些組合語言排列狀況的「特徵」或者「性質」來判定編譯器編譯特徵，例如說VC編譯的迴圈會往下跳執行一段程式碼然後回跳，中間插入一句判斷句來離開向上跳的行為，這就是一種VC標準迴圈的特徵，當然還有更多種特徵，不一一舉例。

說其實這類手花詐欺靜態工具已經是老伎倆了，這邊為初心者入門提供了一些簡易入門的手花技巧，如何簡易的做到讓IDA Pro這類靜態反推原始碼的工具不好做分析、程序開發者如何自行插入一些手花在程式碼來擾亂這類靜態工具（而非只是去花錢買一些Themida、VMP之類的殼來幫程序保護，我們也能自己做些小手段做擾亂分析者）最後目標是做到讓破解者以靜態工具去分析程序原始程式碼和實質執行狀況不同的一些小伎倆，欺騙那些只會用靜態工具做分析的猴子們吧！

附註：此篇所有手花伎倆都是建立於詐欺IDA Pro為基礎上、並不保證對所有類型靜態分析工具皆有效果而測試的IDA Pro版本號為6.6版本，編譯環境為Visual C++ 2013（並且保留pdb完整資訊給IDA Pro做參考做分析）0x01 試探IDA Pro靜態分析（邏輯順序）

0x01 試探IDA Pro靜態分析(邏輯順序)
=====

在一開始我們完全對IDA Pro靜態分析的模式完全沒有任何認識的情況下，我用VC寫了一個簡單的程序並透過逐步加一些手花來測試IDA Pro反組譯後的反應如何，首先我寫了一個大家都能看明白的Hello World程序：

![](http://drops.javaweb.org/uploads/images/70e262e4212fc1713fe494759857e1e82ace8795.jpg)

接著可想而知IDA Pro一定能準確的分析出原始編寫狀況：

![](http://drops.javaweb.org/uploads/images/d6e8ffd0faa59483f0dc1761d8aaa49c08bc53d6.jpg)

那麼接著可以試著嘗試做一些簡易的改變，例如說簡易的跳轉但是Jmp太簡單了，我試著改成push地址與return，看看IDA Pro反組譯的反應如何

![](http://drops.javaweb.org/uploads/images/6bf97409ddfde13afb7037655e64916740704202.jpg)

狀況一如預期的直接無視了push+return的組合（就跟Jmp一樣）反組譯過程中IDA Pro直接將跳躍後的狀況解析進來了，忽視掉跳轉指令

![](http://drops.javaweb.org/uploads/images/280f63e1cf4d67d321626312433ea59706d6f3fb.jpg)

好吧，沒關係 那我們可以再試試看別的！例如說我們知道push會先在堆棧上申請一個DWORD的空間並且寫入你要push的資料，意思是例如push 00可拆解為sub esp,04 + mov [esp], 00，於是我們可以把上述跳轉的手法再改寫：

![](http://drops.javaweb.org/uploads/images/eae8b40fdc2fed0cdd27e7c9b7ee682e4c7c32b7.jpg)

不過IDA Pro反組譯起來的結果令人感到有些失望：

![](http://drops.javaweb.org/uploads/images/280f63e1cf4d67d321626312433ea59706d6f3fb.jpg)

不過到此我們可以得知的是：1. IDA Pro反組譯會根據跳轉順序做一定的推敲 2.除了基礎的跳轉（jmp、Jl、Je...）也會對ret與堆疊上最後返回的地址做相當的推敲來根據順序拼湊出原始編寫程式碼的狀況；那如果我們在做些改變呢？例如往下跳轉後在做記憶體的往上回跳，是否IDA Pro會誤會是一個Loop來處理呢？於是把程式碼改寫一下：

![](http://drops.javaweb.org/uploads/images/2911868e477997490dc60a2fa39b380baf933476.jpg)

這時候會看到我們已經成功阻止IDA Pro做邏輯跳轉的進階解析了（至於為什麼可以顯示出Next01，編譯時候的pdb資訊是有給IDA Pro參考的）如果點擊Next01標籤，IDA Pro是無法進階跳轉進去的（會被阻止進階解析進入）看來IDA Pro認為這原本應該會是一個迴圈結構但是找不到判斷句跳出所以就當作跳入不再出來了（這也是為什麼反組譯結果僅顯示returun &Next01）

![](http://drops.javaweb.org/uploads/images/0029695084bcabd29a8e90cab2c2f4abcca49c86.jpg)

咦，不過就有人會想“既然不過就是詐欺IDA Pro誤判為迴圈嗎？那為什麼不直接這麼寫就好？”於是撰寫了以下程式碼：

![](http://drops.javaweb.org/uploads/images/ec7ade2e81ae8e521ee19d7ef0b1d995eac93958.jpg)

不過很可惜的是IDA Pro遇到push+return的組合就直接作為Jmp處理了，所以這方面解析是完全沒有問題的不會有任何誤判的，到這邊我們可以得到一些結論如下：

*   IDA Pro對於Jmp的跳轉是完全沒有任何問題的（廢話）
*   IDA Pro可識別push+return的組合等價於Jmp指令
*   IDA Pro對於純暫存器搬移處理[ESP]在做條件/return返回的分析能力是很弱的

邏輯跳轉大致上用的手法如上介紹，當然不一定要把跳轉寫在函數內也可寫在外部函數，如下：

這邊在入口做了一個簡單的長程跳躍，而跳躍後的函數地址保存於全域變數中

![](http://drops.javaweb.org/uploads/images/b75b333cbaa47c37bf4732a3e197493dd27ad418.jpg)

而效果也是令人感到不錯滿意的，僅顯示一個return Label

![](http://drops.javaweb.org/uploads/images/6362d84e2c8170db192bd987dc5ecf06db7becec.jpg)

如果點擊了_next()該標籤做解析，此時IDA Pro就會告知以下資訊：

![](http://drops.javaweb.org/uploads/images/4d135baeefbb12c8cef19a8bf3309c78a9f3f1b1.jpg)

僅告知它是全域變數，偏移地址為多少，至於這個手法雖然不能很有效的避免分析（可透過人工閱讀組合語言程式碼來做分析跳轉位置）但在一些自動化的加殼、混淆工具上做保護，「邏輯混淆」便也是依此方式做大量的來回跳轉來擾亂靜態分析工具的追蹤。

0x02 舉個栗子
=====

這種手法可以用在哪裡？例如說有一個小型的CTF — AIS3中有個Crypto考題這麼出的：

![](http://drops.javaweb.org/uploads/images/89f1e3d6e6eab67c1380f15c6976285db7e29270.jpg)

圖中可以看到在入口處（public start）可以看到從入口處有個紅色箭頭往底下指了下去，但是指下去後又跳回了loc_4000CF的位置上（灰色箭頭返回處）而透過IDA Pro的反組譯功能觀察到的程式碼如下：

![](http://drops.javaweb.org/uploads/images/7f41422618c1851ea18f35065dc9aaa5a5ec3447.jpg)

這邊就用了這樣一個很有趣的做法來增加Crypto考題的難度，避免參加CTF比賽者完全沒有閱讀組合語言的能力，僅透過反組譯工具來靜態閱讀、分析，考出了「組合語言閱讀能力」，到這個題目上可以看到它用的跳轉是Jmp，由文章開頭到此我們可以對IDA Pro的「函數結尾」判定解析有了一些了解：

*   正常情況下，遇到ret視為函數結束
*   若找查不到ret則以Jmp當作結尾
*   Jmp在函數內來回跳可做解析
*   Jmp往下跳則無法解析回C（但可點擊至該地址）
*   Jmp往下跳後又往上跳（往回跳）則無法做出明確反組譯

![](http://drops.javaweb.org/uploads/images/d58d9841e74e1299f4710384d4b6e86323a48e80.jpg)

![](http://drops.javaweb.org/uploads/images/09286fecd1454e7c92505c9961536d9226c53071.jpg)

![](http://drops.javaweb.org/uploads/images/ef0784b11c80df89f34c30d51d99a948e82d3b7f.jpg)

接著講到此，順道提一下由知名資安廠商出的CTF — Flare-On，解完第一題後通關會獲得第二道題目，第二道題目中也不巧用了這個手法，所以就順道寫進來介紹了XD

![](http://drops.javaweb.org/uploads/images/1c99a525368ddb9cec3a7f43d2fbe73b5af39526.jpg)

首先我們可將此道題目透過IDA Pro分析，可看到此程序有三個函數，一個標準入口點還有兩個自定函數，接著我們一一稍微檢閱一下它們：

![](http://drops.javaweb.org/uploads/images/1d5633730ef30617826bf2c048f8ac50216e305a.jpg)

首先看見Sub_401000，點擊做進階解析是無法做解析的，IDA Pro會提示該函數的堆棧空間壞掉了（也許是函數頭、申請空間、對堆棧空間釋放、使用...等狀況）導致IDA Pro無法做精確的解析反組譯回標準C的狀態，好的 那麼先放著，看下一個。

![](http://drops.javaweb.org/uploads/images/4b4c376ff4caf48f99e3f1963a36f90e5f9b7d08.jpg)

接著看到sub_401084上，IDA Pro可很輕易的反組譯推敲出原始狀態，看這模樣大概是KeyGen一部份吧，先放著不理它，來看看入口點怎麼做。

![](http://drops.javaweb.org/uploads/images/e741380a004421094fba80e43220127f622082b7.jpg)

看到入口處可看見主要拆成紅區、橘區、綠區三區，紅區做了一個函數呼叫然後丟給橘區做運算處理接著綠區就用到了我們前面提到的邏輯順序置換的手法，跳到入口的組合語言觀察一下：

![](http://drops.javaweb.org/uploads/images/87979f83c1e77c91fdf665e7805a922339881e37.jpg)

這邊可以看到有趣的是綠區的jmp反應了前面反組譯時看到的回跳，應證了前面我們實驗時所見的，IDA Pro反組譯對於「回跳」到前面的記憶體模塊這個行為，會當作函數尾端來處理的（視同return一般作為解析的尾巴）

![](http://drops.javaweb.org/uploads/images/44da26104888d9297b964eb14990dada7b17997a.jpg)

好的，接著那麼我們可以來觀察一下紅區的函數呼叫怎麼做的，我們回到sub_401000的組合語言來觀察一下！（也就是一開始我們發現IDA Pro可辨識出有三個函數，但是不能做反組譯、函數堆棧壞掉的那個函數）

![](http://drops.javaweb.org/uploads/images/ba56a3e79b2a2620c916d8e6e13b1ccb5b36772b.jpg)

接著可以發現，原來在標準函數頭初始化（也就是橘區那段）之前被插入了一個pop eax！接著橘區做的就是標準函數頭會做的初始化函數，然後底下開始做題目的顯示、解析Key正確與否...等KeyGen的行為。（原來就是該pop eax導致此函數被IDA Pro認為函數堆棧壞掉啦！）

![](http://drops.javaweb.org/uploads/images/507a15909bca7892a06bfce19a0af1845423bbce.jpg)

所以回到函數頭可以發現call sub_401000這個行為可以拆解成push 0x004010E4 接著Jmp sub_401000，但又因sub_401000的函數第一個指令就做了pop eax，故前面push的0x004010E8原本位在堆棧第一個指標就被釋放掉了！於是我們會發現這個入口點其實底下紅區處是完全不會被執行到的，事實上入口處進來就直接jmp到sub_401000做事情了，參賽者完全不必分析底下多餘的程式碼，這是一個很漂亮的IDA Pro反組譯詐欺手法呀！

最後，透過VC我實作了這題的詐欺手法原始是怎麼撰寫出來的，寫法如下：

![](http://drops.javaweb.org/uploads/images/2bf58580486494c84061be499e095cb000d8cc2d.jpg)

編譯後將程序交給IDA Pro反組譯分析，可看見：

![](http://drops.javaweb.org/uploads/images/83bf280a62a92f08c9ffb4c58995af3fc4f1364e.jpg)

嘿嘿，與Flare-On第二題完全一樣的伎倆被我們學會怎麼編寫了！

以上內容提及了手花做混淆如何影響靜態分析工具與一些實際應用這些技巧的例子，不過我們並不滿足於此，我們終極目標是撰寫出讓分析者用靜態工具觀察到A程式碼而實際執行的是B行為呢！那我們該如何做到這種感覺是「不可能的任務」的手法呢？

0x03 IDA Pro反組譯分析模式觀察
=====

靜態分析工具在做反組譯理論上應以「組合語言程式碼做了多少事情，就應該推論出多少原始碼」為主，所以只有前面0x01~0x02的主題做單純的邏輯置換，顯然對於我們要做到「隱藏要做的程式碼」與「顯示出欺騙的程式碼」的目標還很遙遠呢！

對此我們需要有更進階對靜態分析工具的反組譯推敲模式有更進階理解才行！這邊我透過內遷組合語言的方式插入了一堆「純暫存器操作」的行為，意圖確認IDA Pro在做反組譯時主要的基準是以什麼為基準點，這邊我插入了大量對純暫存器的操作（紅區）意圖擾亂靜態分析工具的反組譯行為，接著橘區做我想做的程式碼，最後恢復一下堆棧（因為前面用了push 00）然後離開這個入口函數。

![](http://drops.javaweb.org/uploads/images/9ec3fc0ef2e3c368b0aaea44ffcf69eb1b532c91.jpg)

![](http://drops.javaweb.org/uploads/images/9b0fbb32b272396fa670b81e55b76e0661a4ab41.jpg)

而在IDA Pro解析了編譯出的程序後，可見組合語言方面我做擾亂的紅區部分沒有因為VC編譯器優化而被移除，橘區要做的程式碼也都還在，那麼我們來看看IDA Pro反組譯的結果：

![](http://drops.javaweb.org/uploads/images/280f63e1cf4d67d321626312433ea59706d6f3fb.jpg)

這時候反組譯之後看到的程式碼卻只還原出橘區的程式碼，前面紅區那些針對純暫存器的操作通通被IDA Pro認為無用途而移除不做解釋了！這邊可以發現IDA Pro在反組譯解析時的特性：

1.  針對一個個函數為基礎單位做解析
2.  僅針對函數內區域變數（或者操作到全域變數）的部分做解釋回C程式碼
3.  由第二點，對於其餘純暫存器操作是不做任何解析的（被忽視掉）
4.  第三點唯一的例外為esp暫存器，用於確認該函數是否損毀、實際返回狀況 （透過esp返回狀況，IDA Pro才能畫出流程圖）

這時候可以想想看，有什麼辦法是可以做到：

1.  跳轉到其他函數、地址上
2.  不操作到任何區域變數、全域變數
3.  不使用Jmp、Jl、Je等跳轉指令（只要出現IDA Pro便能畫出圖）
4.  純暫存器操作

0x04 SEH（Structured Exception Handling）
=====

![](http://drops.javaweb.org/uploads/images/42ec0f988ab821d73176f325369a23b65cfc4048.jpg)

舉例到這邊，對Windows PE結構熟的朋友立刻就能想到：這不就是Windows異常處理機制SEH嗎？是的！有用過SEH形式的異常處理並且逆向過就可以得知SEH組合語言寫法上如下

```
push Handler
mov eax,fs:[0]
push eax
mov fs:[0],esp

```

上面是一個標準的SEH式的try在做「異常機制註冊」的程式碼，fs:[0]指向了Windows的Thread遇到異常出錯時，要去找誰解決，這時候fs:[0]保存著一群幫忙解決異常問題的「Handler」的清單（至於Handler是什麼？例如說你的try中的catch要做的處理，就是Handler在做的處理，只是編譯器會幫你封裝為Handler的函數型態）

所以註冊時會先把Handler先push入堆棧，用堆棧空間來紀錄Handler的地址接著把原始fs:[0]清單上原始Handler也push推到堆棧做保存，接著把esp寫入到fs:[0]之內，如此一來fs:[0]內清單Handler就會是：原始Handler -> 接著是你的catch的handler -> 剩餘的其他Handler …依此接續下去，如果Thread在運作時發生任何異常，就會調閱出fs:[0]的清單來找Handler協助解決問題。

那麼，我們既然做了push兩次，勢必做完try包起來的事情必須幫忙try做恢復，避免堆棧被我們玩壞掉呢，有註冊就有解除註冊的機制，其實也就是把前面那段註冊的程式碼倒著寫

```
mov eax,[esp]
mov fs:[0],eax
add eap,0x08

```

到這邊你會發現，SEH異常處理機制不就恰恰滿足我們想要的「純暫存器處理」、「不動用程序中的變數」了嗎？簡直為我們這次目的量身打造的設計呀！

0x05 詐欺IDA Pro：反組譯與行為完全不同的伎倆
=====

首先，將VC的專案中C/C++的例外處理機制改為「是，但有SEH例外狀況」來處理（預設為「是（/EHsc) ）如此一來我們就可以不用深入理解SEH結構體的運作寫法以純組合語言方式來操作SEH異常處理寫法了（不過有興趣的童鞋還是可以Google一下的）

![](http://drops.javaweb.org/uploads/images/76d783e9aec7f773ca79ebd58541376e8da5df70.jpg)

![](http://drops.javaweb.org/uploads/images/c68f70bb6839530eb889bbaa432d3c13aff01bfd.jpg)

我們可以寫一個很簡單的try，在try內寫上印出“ADR Is Handsome!”而在發生異常時顯示出“Hello World”，那麼我們來看看IDA Pro分析狀況：

![](http://drops.javaweb.org/uploads/images/df91201c6e8ce318d81113ce10de1b2e608ab6ba.jpg)

在0x004113D5 ~ 0x004113E0的部分與底下0x00411408就是一整個SEH的註冊。 （push了Handler並且重新連結fs:[0]中的Handler順序）

![](http://drops.javaweb.org/uploads/images/0f31ecc96f8f1a27a17264dd56c9009f458e1d87.jpg)

而到底下可看見前面我們發現的0x00411408就是最後一句完成SEH註冊的部分，底下做了我們寫好的透過printf打印出“ADR is Handsome!”，如果打印沒出任何狀況，底下jmp loc_41144E就會做恢復SEH註冊並且離開函數。這裡我們可以發現，咦？他居然有了jmp，所以IDA Pro應該會把此Jmp視為此函數的結尾，並把catch部分的程式碼忘記才對！接著我們透過IDA Pro反組譯看看是否如我們預期一樣：

![](http://drops.javaweb.org/uploads/images/06122089840bf316a017d71b8fcf5ba05b12cea7.jpg)

看起來效果相當顯著啊，肯定是電系神奇寶貝遇上了水系神奇寶貝了（？）這裡看見我們已經可以透過try的方式來隱藏我們不希望別人看到的程式碼，不過目前執行的程式碼的確就是執行印出“ADR is Handsome!”那麼我們該如何改變程序執行流程，來跳到Handler的部分而不執行原始程式碼呢？這問題簡單呀，SEH本來就是用於作異常機制處理，那我們只要再執行顯示”ADR is Handsome!“以前引發一個異常就可以了嘛！

![](http://drops.javaweb.org/uploads/images/003909449d0cd40c2ae41302268ff19fdaca1777.jpg)

所以我們可以寫出像這樣子的程式碼，在printf出”ADR is Handsome!“以前先透過xor eax,eax清空eax後再去存取[0]的值為多少，就會引發記憶體不存在的錯誤而跳轉到Hello World顯示了！編譯好後交給IDA Pro反組譯狀況如下：

![](http://drops.javaweb.org/uploads/images/06122089840bf316a017d71b8fcf5ba05b12cea7.jpg)

效果顯著啊，那麼執行起來呢？

![](http://drops.javaweb.org/uploads/images/11a2eb661fab0b6225a2e512eb0b202cde2f709c.jpg)

成功做出了執行與顯示是兩回事，嘴上說不要身體卻很實在的程序呀！

0x06 延伸有趣的解析伎倆
=====

前面從0x01 ~ 0x05章節我們一路透過IDA Pro的解析特性來引導IDA Pro反組譯時解析到我們故意提供的錯誤資訊，那麼還有什麼有趣的伎倆呢？例如說：

![](http://drops.javaweb.org/uploads/images/dcaa8b9febdc998d853945a6cdab0b606c4b7b53.jpg)

我們前面透過SEH類型的Try然後引發異常錯誤來引導IDA Pro解析到與執行是不同的程式碼段，那麼這樣還有什麼有趣的呢？既然我們知道IDA Pro會由函數頭一路解析至Try內包的所有程式碼結束，那麼有趣的事情來了，在標準C++編譯下的函數，例如說有3個參數，那麼因為函數頭部分會去申請3個DWORD空間來存放參數，所以return時候必須釋放掉來平衡堆棧，一般來說返回會寫成：ret 0x0C （0x0C = 12 = 3* sizeof(DWORD) = 3 * 4）

所以假設我們故意在IDA Pro解析時給予ret 0xFF呢？是否會被視為參數有63幾個參數呢？

把編譯好的程序交給IDA Pro做反組譯得到：

![](http://drops.javaweb.org/uploads/images/521927dd5e93feb72a774fd399a1a0d32ceb5f2c.jpg)

嘿嘿，真的成功引起IDA Pro反組譯對參數需求的辨識錯誤啦！

最後，還有沒有什麼跳轉手法是可以引起IDA Pro反組譯解析錯誤的？事實上還有許多的，只要能達成純暫存器處理與不動用變數即可！這邊做個簡單好吃的小栗子！

![](http://drops.javaweb.org/uploads/images/d9c9eeb257d0bceb52e5a68be456e65ed758fa37.jpg)

這邊我把實際執行的程式碼放在H3llo()函數內，在入口處想辦法Jmp過去就對了！不過前面我們推論出不管是用push + ret或者Jmp系列跳轉指令，IDA Pro反組譯都可推論出正確的ESP並且進階追蹤下去，無法做到完全隱藏的行為，那麼真的就這樣放棄嗎？

這邊我做了一個純暫存器處理與不動用變數的例子，透過push函數地址，再push隨意一個垃圾上去，最後想辦法去把那個垃圾從堆棧上釋放掉（例如這邊我用了lea指令來實現ESP+4的行為）在做ret，此時IDA Pro反組譯就無法推論出正確的ESP做持續追蹤，出現了以下的反組譯狀況：

![](http://drops.javaweb.org/uploads/images/a9814e2e4bde027069b02cc1248d6601c775a3f1.jpg)

![](http://drops.javaweb.org/uploads/images/bfb4ec183399fb45cfe185c80970ce8c4d150c6a.jpg)

又是一個可以做到實際執行與反組譯結果完全不同的做法囉！

0x07 總結
=====

玩手花是相當好玩、有創意的！這邊內容總結了許多靜態反組譯工具的一些解析都是因為分析時太過依賴一些特徵才導致可以被惡意利用，撰寫出造成靜態分析工具分析錯誤的一些小伎倆；而現在靜態分析工具越來越進步下，許多惡意軟體鑑識人員漸漸仰賴靜態分析工具而少開了動態分析工具，這邊提及了一些簡單的詐欺手法怎麼做混淆、手花，也藉此提醒分析人員應以動態分析為主而靜態分析為輔，在動態分析時遇到難以解決或者看不懂的部分才以靜態分析工具做查詢，才是正確的分析王道！如有任何意見聯絡或者聊天都可私信我，特別是女網友！