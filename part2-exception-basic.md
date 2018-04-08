# Part2 例外處理的基本觀念
## 1. 強健性的威脅潛伏
* 影響強健度的威脅:
    1. fault(缺陷)
    1. error(錯誤)
    1. failure(失效, 失敗)
    1. exception(例外, 異常)
* failure: 當一個軟體元件(function, method or service)所提供的服務與其功能規格相符則稱為正確服務(correct service), 反之則稱為服務失效(service failure)或失效(failure). 例如:
    1. 1+5若回傳6以外的答案, 則函數失效
    1. 網路購物線上扣款要求在60秒內完成, 若超過則視為扣款功能失效
* error: 指軟體元件內部處於錯誤狀態(error status), 此狀態可能使元件執行失敗, 導致failure
* fault: 是假設或經過判斷後所確認導致error發生的原因
    * 依據其**存在時間長短**可分為下列三類:
        1. 暫態缺陷(transient fault): 出現一次之後就消失, 若重新執行一次失效的操作則會執行正常
        1. 間歇缺陷(intermittent fault): 出現之後消失, 然後又重複出現. 有可能是硬體所導致, 也有可能是資源洩漏(resource leak), 一般來說重新啟動可解決, 但系統使用一陣子後問題又會出現
        1. 永久缺陷(permanent fault): 發生之後除非將有問題的元件換掉, 否則元件會一直處在failure的狀態
    * 依據其**產生原因**可分為兩類:
        1. 設計缺陷(default fault): 就是bug, 屬於永久缺陷
        1. 元件缺陷(component fault): 硬體元件失效所產生的錯誤, 或延伸為軟體元件之間或與硬體的互動時產生不正常情況, 有可能是暫態, 間歇或永久缺陷
    * Avizienis將fault分為下列三類:
        1. 開發缺陷(development fault): 開發過程中所注入在產品中的缺陷, 如需求錯誤, 程式撰寫錯誤
        1. 實體缺陷(physical fault): 硬體設備缺陷
        1. 互動缺陷(interaction fault): 由內部互動所引起的問題, 如使用者操作錯誤, 錯誤的系統設定等
    * 本書對於fault分類是依產生原因
        1. default fault: 相當於development fault, 即程式設計不良而出現bug
        1. component fault: 相當於physical fault或interaction fault, 個別元件的設計和實作都沒問題, 但元件互動之後產生問題
    * 觸發流程:
        1. fault發生(default or component)
        1. 活化error
        1. 傳播後造成failure
* Exception是程式語言用來表達error和failure的概念, 例如A函數發生IOException
    1. A函數內部代表error(目前可能處在不正確的狀態)
    1. 呼叫者代表failure(被呼叫的函數辦事不力)
* 找不到資料是不是異常? 是一種常見狀況不應該丟出例外
    * 可以回傳null
    * 也可以使用null object設計模式

## 2. 例外處理的四種脈絡
* 例外處理的四種不同context(脈絡, 情境):
    1. 例外脈絡(exception context)
    1. 局部脈絡(local context)
    1. 物件脈絡(object context)
    1. 架構脈絡(architecture context)
* exception context: 當例外發生時, 例外處理程序(exception handler, handler)可以從例外物件取得相關資訊
    * 可由例外產生者(signaler)傳入, 例如使用例外串接(exception chaining, 由catch抓住後再丟出例外), 執行後可以觀察console中的Cause by
    * Throwable建構子只有下面四種, 除非在Java中自行設計例外, 否則例外產生者傳遞給例外處理程序的資訊很有限
        1. Throwable()
        1. Throwable(String message)
        1. Throwable(String message, Throwable cause)
        1. Throwable(Throwable cause)
* object context: OOP中, object context表示在一個物件裡面可以存取到的資訊, 包含物件的資料成員(data member)及全域變數
    * 請看下面程式碼中catch捕捉例外後帶入了資料成員的資訊:
    ```
    public class ObjectContext {
        private Connection conn;
        private String userName;
        private String password;
        //...
        public void connect() throws SQLException {
            //...
        } 
        public void readData() {
            try {
                connect();
                //...
            } catch (SQLException e) {
                //handle the exception may need the object context
            } finally {
                //...
            }
        }

    }
    ```
* local context: 本書以一個函數當作一個local context的範圍(廣義來說任何程式區塊所形成的scope便可算是local context)
    * 例外處理的設計方法:
        1. 什麼都不做
        1. 丟出另一個錯誤語意比較清楚的InvalidPacketException
* architecture context: 若前三種方法仍無法決定例外處理的設計方法, 需要將範圍擴大, 從軟體架構的角度來評估例外處理策略, 請參考下列範例:
    * 背景: 網路遊戲軟體架構:
        1. presentation layer(AppWin): 顯示遊戲畫面
        1. application layer(GameServer): 負責遊戲邏輯
        1. service layer(Acceptor): 監聽某個TCP port
    * 情境: 假設Acceptor要監聽某個port卻發現已被其他人占用, 因此收到IOException, 請問Acceptor該如何處理?
    * 解答:
        1. 特定情境的最佳化例外處理(如果收到錯誤就改port號): 不好, 因為是底層元件故重複利用率高, 若針對特定情境設計例外處理則該元件在其他情境裡面的重複使用機率會下降, 且通常底層元件沒有足夠的architecture context, 所以不知道自己有可能會被用在那些情境之中
        1. 把例外直接向上層(GameServer或AppWin)回報: 正確的做法, 不要客製化處理
        1. GameServer vs AppWin: 對於哪一層處理沒有標準答案, 而是要處理時所獲得architecture context是否足夠一般化. 以本例來說, GameServer應該就有足夠資訊可以處理
## 3. 物件導向語言的例外處理機制
* 設計例外處理機制所需考量的10個因素(Garcia):
    1. Representation: 如何表達一個例外
    1. Declaration: 往外傳遞的例外是否需要宣告
    1. Signaling: 如何產生一個例外的實例(instance)
    1. Propagation: 如何傳遞一個例外
    1. Attachment: 例外處理程序可綁定到何種程式區塊
    1. Resolution: 針對一個例外, 如何找到可以處理它的例外處理程序
    1. Continuation: 例外發生後, 程式的控制流程該如何進行
    1. Cleanup: 無論是否發生例外, 如何讓程式清理資源以維持在正確狀態
    1. Reliability Check: 因為例外處理機制所造成的問題, 程式語言提供何種檢查
    1. Concurrency: 程式語言對於並行處理程式提供多少例外處理的支援
1. Representation: 如何表達一個例外
    1. 符號(symbol): 以字串或整數來表達一種例外狀況
    1. 資料物件(data object): 用物件來表示或儲存錯誤訊息
    1. 完整物件(full object): 用物件來儲存之外, 連如何產生一個例外實例, 和例外發生後的控制流等全部定義在例外類別之中, 如BETA的例外
1. Declaration: 往外傳遞的例外是否需要宣告
    1. 強制式: 所有往外傳遞的例外一定要宣告在函數介面上, 如Java的checked exception, 若違反規則的函數視為編譯錯誤
    1. 選擇式: 宣不宣告皆可, 如Java的unchecked exception
    1. 不支援: 根本不支援在介面上宣告例外, 如C#
    1. 混合式: 同時支援上述一種以上的方式, 如Java支援強制式和選擇式
1. Signaling: 把例外類別的實例傳遞給例外接收者的這個動作稱為signaling, throwing, raising或triggering
    * **例外產生的方式**可分為兩種:
        1. 同步例外(synchronous exception): 因程式中呼叫的指令或函數執行失敗所產生的例外
        1. 非同步例外(asynchronous exception): 由執行環境主(JVM)動丟出的例外, 例如記憶體不足的例外
        * 大致上我們討論如何處理同步例外, 若發生非同步例外最好就是結束程式執行, 因為執行環境已經產生嚴重錯誤
    * 從例外**是否可傳遞出軟體元件**的介面, 可分為兩類: 
        1. 內部例外(internal exception): 軟體元件內部所引發的例外, 主要的目的是將程式執行到內部定義的例外處理程序. 內部例外無法傳遞出軟體元件
        1. 外部例外(external exception): 軟體元件內部無法處理的例外, 需要將例外往外傳遞
        * 以Java為例並沒有區分內外部例外, 如果一個例外被一個函數給捕捉且沒有往外傳, 則是內部例外, 若有拋出則是外部例外, 但無論哪一種都是使用throw這個指令來丟出例外
1. Propagation: 討論例外傳遞的問題
    1. 外顯式(explicit): 接收到例外的人如果不想處理這個例外, 必須要明白指出要把這個未被處理的例外往外丟, 如Java的checked exception
    1. 內隱式(implicit): 又稱為自動傳遞方式, 接收到例外的人, 如果不想處理這個例外, 在預設情況下將直接往外傳遞, 如Java的unchecked exception
1. Attachment: 例外處理程序可附加到哪些受保護的程式區塊
    1. 敘述(statement)或區塊(block): 將例外處理程序與程式敘述或式區塊綁定在一起, 如try-catch就是catch定義的例外處理程序和try定義的程式敘述綁定在一起
    1. 函數: Eiffle
    1. 物件: BETA
    1. 類別: BETA, Ada95等
    1. 例外: BETA
1. Resolution: 針對一個例外, 如何找到可以處理它的例外處理程序
1. Continuation: 例外發生後, 程式的控制流程該如何進行
1. Cleanup: 無論是否發生例外, 如何讓程式清理資源以維持在正確狀態
1. Reliability Check: 因為例外處理機制所造成的問題, 程式語言提供何種檢查
1. Concurrency: 程式語言對於並行處理程式提供多少例外處理的支援

