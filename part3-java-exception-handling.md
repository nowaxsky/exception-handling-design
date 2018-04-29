# Part3 Java語言的例外處理機制
## 1. Java的Try, Catch, Finally
* 例外類別繼承架構:

Checked | Unchecked
--- | ---
Throwable | --
Exception | Error
IOException, SQLException | RuntimeException
...... | IndexOutOfBoundsException, NullPointerException

* 一般來說Error發生大部分會導致程式終止, 因此一般最後catch捕捉到Exception而非Throwable
* 越上層的例外類別應該越晚捕捉, 如果一開始就catch Exception將會導致catch IOException沒有機會被執行, 正確的順序如下範例:
```
public void tryCatch() {
    try{
        throwIOException();
        throwSQLException();
    } catch (IOException e) {
        //
    } catch (SQLException e) {
        //
    } catch (Exception e) {
        //
    }
}

```
* **finally block一定會被執行, 即使在try中最後使用了return**
* **但如果在try block或catch block中JVM被程式終止時, finally block不會被執行**
```
public static void main(String[] args) {
    try {
        System.out.println("execute try block");
        System.exit(0);
    } finally {
        System.out.println("execute finally block");
    }
}
```
* Java7之後多了multi-catch exception, try-with-resources, rethrow三種特性
    1. multi-catch exception: 可在一個catch中同時捕捉多種例外, 可以讓這些例外都使用相同的處理方式, 如下例:
        ```
        public void tryCatch() {
            try {
                //
            } catch (ParseException | IOException e) {
                //
            }
        }
        ``` 
    1. try-with-resources: Java7之前要釋放資源需要寫在finally裡面, Java7之後可以把產生資源的程式寫在try()中, 若該類別有實作AutoCloseable則可在結束時自動關閉, 不需要再另外寫finally block來釋放資源, 如下例:
        ```
        public void tryCatch() {
            try(FileInputStream fis = new FileInputStream(new File("FileName"))) {
                //
            } 
        }
        ``` 
    1. rethrow: 如下面的例子, 過去是無法通過編譯的, 因為在catch例外後往外丟, 編譯器會認為宣告的例外會是捕捉的例外Exception, Java7後編譯器可以辨認其實是catch後又丟出一個IOException, 所以宣告寫throws IOExeption而非Exception:
        ```
        public void rethrow() throws IOException {
            try {
                throw new IOException("error");
            } catch (Exception e) {
                throw e;
            }
        }
        ``` 
## 2. Finally Block覆蓋問題
* finally block丟出的例外會覆蓋掉try blcok和catch block所丟出的例外, 如下例:
    ```
    public class Test {

        public static void finallyException() throws IOException {
            try {
                throw new IOException();    
            } finally {
                try {
                    throw new IOException();
                } catch (IOException e) {
                    throw e;
                }
            }
        }
    
        public static void main(String[] args) {
            try {
                finallyException();
            } catch (Exception e) {
                e.printStackTrace();
                Throwable [] t = e.getSuppressed();
                System.err.println("suppressed exception size = " + t.length);
            }
        }
    }    
    ```
* 執行結果如下(第12行是第二個IOException):
    ```
    java.io.IOException
        at org.cpm.zwl.util.Test.finallyException(Test.java:12)
        at org.cpm.zwl.util.Test.main(Test.java:21)
    suppressed exception size = 0
    ```
* 兩個IOException代表的錯誤意義:
    1. 功能失效(function failure): 可以retry, 或考慮呼叫其他函數來取代失敗的函數, 如果是design fault則需要修改程式
    1. 清理失效(cleanup failure): 資源釋放失敗, 一般為design fault, 需要修改程式
* 實務上, 會保留function failure(第一個錯誤), 因為cleanup failure(第二個錯誤)發生的機率比較低, 且就算發生系統依然能再執行一陣子(如果沒有釋放記憶體也不會立刻導致系統問題), 所以在finally產生例外的話, 需要把例外記錄到log中
* 但是理想上2個錯誤都應該要保留, 而且在Java SE7之後支援try-with-resources, 因為沒有finally block, 也無法保存錯誤. 解決方法為suppressed exception, 請參考下一節

## 3. Suppressed Exception
* Java SE7在Throwable中加入了getSuppressed()來取得suppressed exception, 範例如下:
    ```
    public class MyOutputStream implements AutoCloseable {

        @Override
        public void close() throws Exception {
            throw new IOException();    
        }

    }

    public class Test {

        public static void tryWithResource() throws Exception {

            try(MyOutputStream mos = new MyOutputStream()) {
                throw new IOException("Function failure");
            } 
        }
    
        public static void main(String[] args) {
            try {
                tryWithResource();
            } catch (Exception e) {
                e.printStackTrace();
                Throwable [] t = e.getSuppressed();
                System.err.println("suppressed exception size = " + t.length);
            }
        }
    }
    ```
* 執行的結果如下:
    ```
    java.io.IOException: Function failure
        at org.cpm.zwl.util.Test.tryWithResource(Test.java:10)
        at org.cpm.zwl.util.Test.main(Test.java:16)
        Suppressed: java.io.IOException
            at org.cpm.zwl.util.MyOutputStream.close(MyOutputStream.java:9)
            at org.cpm.zwl.util.Test.tryWithResource(Test.java:11)
            ... 1 more
    suppressed exception size = 1
    ```
* 由執行結果可以得知, main(呼叫者)所捕捉到的例外是tryWithResource所丟出來的IOException(即function failure), 在close中所丟出的IOException(cleanup failure)變成了第一個錯誤的suppressed exception
* **總結: 在使用try-with-resources時丟出的例外是function failure, 若function failure附帶suppressed exception, 則表示還發生cleanup failure**
* 若是沒有任何的function failure, 只有cleanup failure, 則suppressed exception不會出現(嘗試把上面的例子中, try裡面的```throw new IOException("Function failure");```這行刪掉), 或者其他suppressed exception中(依堆疊順序反向出現), 如下例:
    ```
    public class MyConnection implements AutoCloseable {

        @Override
        public void close() throws Exception {
            throw new SQLException();
        }
    }

    public class MyInputStream implements AutoCloseable {

        @Override
        public void close() throws Exception {
            throw new FileNotFoundException();
        }
    }

    public class Test {

        public static void tryWithResource() throws Exception {

            try (MyOutputStream mos = new MyOutputStream();
                MyConnection mc = new MyConnection();
                MyInputStream mis = new MyInputStream()) {
            }
        }

        public static void main(String[] args) {
            try {
                tryWithResource();
            } catch (Exception e) {
                e.printStackTrace();
                Throwable[] t = e.getSuppressed();
                System.err.println("suppressed exception size = " + t.length);
            }
        }
    }
    ```
* 執行後, 由於MyInputStream最先關閉所以先拋出FileNotFoundException, 接著依序為MyConnection和MyOutputStream, 形成的例外結構為最先出現的錯誤會顯示, suppressed exception依序為SQLException和IOException, 結果如下:
    ```
    java.io.FileNotFoundException
        at org.cpm.zwl.util.MyInputStream.close(MyInputStream.java:9)
        at org.cpm.zwl.util.Test.tryWithResource(Test.java:10)
        at org.cpm.zwl.util.Test.main(Test.java:15)
        Suppressed: java.sql.SQLException
            at org.cpm.zwl.util.MyConnection.close(MyConnection.java:9)
            ... 2 more
        Suppressed: java.io.IOException
            at org.cpm.zwl.util.MyOutputStream.close(MyOutputStream.java:9)
            ... 2 more
    suppressed exception size = 2
    ```
* 由上面的例子可以知道例外的結構有多種可能, 有可能全部是cleanup failure, 也有可能是1個function failure, 其他是cleanup failure, 若只出現一個exception甚至也無法判斷是cleanup failure還是function failure, 所以我們需要更清楚的例外類別設計

## 4. 清楚的Cleanup Failure語意
* 要有清楚的cleanup failure, 作者建議自行定義CleanupException用來代表cleanup failure:
    ```
    public class CleanupException extends RuntimeException {

        public CleanupException(Exception e) {
            super(e);
        }
    }
    ```
* 繼承自RuntimeException(是一個unchecked exception)的理由:
    1. 任何釋放資源的地方都有可能丟出CleanupException, 如果設計成checked exception則每次都要處理這個較次要的錯誤, 反而容易導致例外反射性地被忽略而下降系統強健度
    1. 發生cleanup failure的機率其實不高, 若是真的發生再將這個例外交由系統最上層寫入log, 或立即顯示讓使用者得知
* 修正cleanup failure(實作AutoCloseable)相關的類別, 使其接到錯誤後拋出CleanupException, 如下例:
    ```
    public class MyConnection implements AutoCloseable {

        @Override
        public void close() throws Exception {
            try {
                throw new SQLException();
            } catch (Exception e) {
                throw new CleanupException(e);
            }
        }
    }
    ```
* 將MyInputStream和MyOutputStream都加入CleanupException後, 重新執行一次:
    ```
    org.cpm.zwl.util.CleanupException: java.io.FileNotFoundException
        at org.cpm.zwl.util.MyInputStream.close(MyInputStream.java:12)
        at org.cpm.zwl.util.Test.tryWithResource(Test.java:13)
        at org.cpm.zwl.util.Test.main(Test.java:18)
        Suppressed: org.cpm.zwl.util.CleanupException: java.sql.SQLException
            at org.cpm.zwl.util.MyConnection.close(MyConnection.java:12)
            ... 2 more
        Caused by: java.sql.SQLException
            at org.cpm.zwl.util.MyConnection.close(MyConnection.java:10)
            ... 2 more
        Suppressed: org.cpm.zwl.util.CleanupException: java.io.IOException
            at org.cpm.zwl.util.MyOutputStream.close(MyOutputStream.java:12)
            ... 2 more
        Caused by: java.io.IOException
            at org.cpm.zwl.util.MyOutputStream.close(MyOutputStream.java:10)
            ... 2 more
    Caused by: java.io.FileNotFoundException
        at org.cpm.zwl.util.MyInputStream.close(MyInputStream.java:10)
        ... 2 more
    suppressed exception size = 2
    ```
* 如果在try中加入function failure:
    ```
    public class Test {

        public static void tryWithResource() throws Exception {

            try (MyOutputStream mos = new MyOutputStream();
                MyConnection mc = new MyConnection();
                MyInputStream mis = new MyInputStream()) {
                
                // add this line
                throw new IOException();
            }
        }

        public static void main(String[] args) {
            try {
                tryWithResource();
            } catch (Exception e) {
                e.printStackTrace();
                Throwable[] t = e.getSuppressed();
                System.err.println("suppressed exception size = " + t.length);
            }
        }
    }
    ```
* 結果如下:
    ```
    java.io.IOException
        at org.cpm.zwl.util.Test.tryWithResource(Test.java:12)
        at org.cpm.zwl.util.Test.main(Test.java:18)
        Suppressed: org.cpm.zwl.util.CleanupException: java.io.FileNotFoundException
            at org.cpm.zwl.util.MyInputStream.close(MyInputStream.java:12)
            at org.cpm.zwl.util.Test.tryWithResource(Test.java:13)
            ... 1 more
        Caused by: java.io.FileNotFoundException
            at org.cpm.zwl.util.MyInputStream.close(MyInputStream.java:10)
            ... 2 more
        Suppressed: org.cpm.zwl.util.CleanupException: java.sql.SQLException
            at org.cpm.zwl.util.MyConnection.close(MyConnection.java:12)
            at org.cpm.zwl.util.Test.tryWithResource(Test.java:13)
            ... 1 more
        Caused by: java.sql.SQLException
            at org.cpm.zwl.util.MyConnection.close(MyConnection.java:10)
            ... 2 more
        Suppressed: org.cpm.zwl.util.CleanupException: java.io.IOException
            at org.cpm.zwl.util.MyOutputStream.close(MyOutputStream.java:12)
            at org.cpm.zwl.util.Test.tryWithResource(Test.java:13)
            ... 1 more
        Caused by: java.io.IOException
            at org.cpm.zwl.util.MyOutputStream.close(MyOutputStream.java:10)
            ... 2 more
    suppressed exception size = 3
    ```
* 如此我們可以清楚區分是function failure或是cleanup failure了, 但還有2點問題待改善:
    1. 在實作AutoCloseable類別時, close方法只能拋出Exception, 不能拋出CleanupException
    1. 使用try-with-resources時JVM會幫忙產生suppressed exception, 但傳統finally寫法卻不會, 依然發生finally拋出的例外覆蓋掉try或catch例外的狀況

## 5.自行製作Suppressed Exception
* 本節主要解決不能拋出CleanupException和不使用try-with-resources時的覆蓋問題
* 先將MyInputStream, MyConnection, 和MyOutputStream改成只拋出原來的例外FileNotFoundException, SQLException, IOException, 這邊的目的是模擬Java內建的資源物件
* 設計一個Cleaner類別如下:
    ```
    public class Cleaner {

        private Stack<AutoCloseable> stack;
        private Throwable leadException;
        private CleanupException outgoingCleanupException;

        private Cleaner() {
            this.stack = new Stack<>();
            this.leadException = null;
            this.outgoingCleanupException = null;
        }

        static public Cleaner newInstance() {
            return new Cleaner();
        }

        public <T extends AutoCloseable> T push(T obj) {
            return (T) stack.push(obj);
        }

        public void setLeadException(Throwable lead) {
            this.leadException = lead;
        }

        public void clear() {
            while (!stack.isEmpty()) {
                AutoCloseable auto = stack.pop();
                try {
                    if (null != auto) auto.close();
                } catch (Exception e) {
                    if (hasLeadException()) {
                        leadException.addSuppressed(new CleanupException(e));
                    } else if (!hasOutgoingCleanupException()) {
                        outgoingCleanupException = new CleanupException(e);
                    } else {
                        outgoingCleanupException.addSuppressed(new CleanupException(e));
                    }
                }
            }
            if (hasOutgoingCleanupException()) throw outgoingCleanupException;
        }

        public boolean hasLeadException() {
            return (null != leadException) ? true : false;
        }

        public boolean hasOutgoingCleanupException() {
            return (null != outgoingCleanupException) ? true : false;
        }

    }

    ```
* 接著使用finally來顯示上節的範例:
    ```
    public class Test {

        public static void failureException() throws Exception {

            MyOutputStream mos = null;
            MyConnection mc = null;
            MyInputStream mis = null;
            Cleaner cln = Cleaner.newInstance();
            try {
                mos = cln.push(new MyOutputStream());
                mc = cln.push(new MyConnection());
                mis = cln.push(new MyInputStream());
                throw new IOException("Function failure");
            } catch (Exception e) {
                cln.setLeadException(e);
                throw e;
            } finally {
                cln.clear();
            }

        }

        public static void main(String[] args) {
            try {
                failureException();
            } catch (Exception e) {
                e.printStackTrace();
                Throwable[] t = e.getSuppressed();
                System.err.println("suppressed exception size = " + t.length);
            }
        }
    }
    ```
* 這邊的做法就是模擬try-with-resources, 先將類別放入Cleaner中, 若在try中發生錯誤, 則會進入自定義的leadException裡(即function failure), 接著cleanup failure會是leadException的suppressed exception, 若try中並未發生錯誤, 則第一個cleanup failure會成為自定義的outgoingCleanupException, 接著其他cleanup failure會是outgoingCleanupException的suppressed exception, 執行結果如下:
    ```
    java.io.IOException: Function failure
        at org.cpm.zwl.util.Test.failureException(Test.java:17)
        at org.cpm.zwl.util.Test.main(Test.java:29)
        Suppressed: org.cpm.zwl.util.CleanupException: java.io.FileNotFoundException
            at org.cpm.zwl.util.Cleaner.clear(Cleaner.java:37)
            at org.cpm.zwl.util.Test.failureException(Test.java:22)
            ... 1 more
        Caused by: java.io.FileNotFoundException
            at org.cpm.zwl.util.MyInputStream.close(MyInputStream.java:9)
            at org.cpm.zwl.util.Cleaner.clear(Cleaner.java:34)
            ... 2 more
        Suppressed: org.cpm.zwl.util.CleanupException: java.sql.SQLException
            at org.cpm.zwl.util.Cleaner.clear(Cleaner.java:37)
            at org.cpm.zwl.util.Test.failureException(Test.java:22)
            ... 1 more
        Caused by: java.sql.SQLException
            at org.cpm.zwl.util.MyConnection.close(MyConnection.java:9)
            at org.cpm.zwl.util.Cleaner.clear(Cleaner.java:34)
            ... 2 more
        Suppressed: org.cpm.zwl.util.CleanupException: java.io.IOException
            at org.cpm.zwl.util.Cleaner.clear(Cleaner.java:37)
            at org.cpm.zwl.util.Test.failureException(Test.java:22)
            ... 1 more
        Caused by: java.io.IOException
            at org.cpm.zwl.util.MyOutputStream.close(MyOutputStream.java:9)
            at org.cpm.zwl.util.Cleaner.clear(Cleaner.java:34)
            ... 2 more
    suppressed exception size = 3
    ```
* 將try中的例外拿掉結果如下:
    ```
    org.cpm.zwl.util.CleanupException: java.io.FileNotFoundException
        at org.cpm.zwl.util.Cleaner.clear(Cleaner.java:39)
        at org.cpm.zwl.util.Test.failureException(Test.java:22)
        at org.cpm.zwl.util.Test.main(Test.java:29)
        Suppressed: org.cpm.zwl.util.CleanupException: java.sql.SQLException
            at org.cpm.zwl.util.Cleaner.clear(Cleaner.java:41)
            ... 2 more
        Caused by: java.sql.SQLException
            at org.cpm.zwl.util.MyConnection.close(MyConnection.java:9)
            at org.cpm.zwl.util.Cleaner.clear(Cleaner.java:34)
            ... 2 more
        Suppressed: org.cpm.zwl.util.CleanupException: java.io.IOException
            at org.cpm.zwl.util.Cleaner.clear(Cleaner.java:41)
            ... 2 more
        Caused by: java.io.IOException
            at org.cpm.zwl.util.MyOutputStream.close(MyOutputStream.java:9)
            at org.cpm.zwl.util.Cleaner.clear(Cleaner.java:34)
            ... 2 more
    Caused by: java.io.FileNotFoundException
        at org.cpm.zwl.util.MyInputStream.close(MyInputStream.java:9)
        at org.cpm.zwl.util.Cleaner.clear(Cleaner.java:34)
        ... 2 more
    suppressed exception size = 2
    ```
## 6. Try, Catch, Finally的責任分擔