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
* finally block丟出的例外會覆蓋掉try blcok和catch block所丟出的例外

## 3. 物件導向語言的例外處理機制

## 4. 例外處理vs容錯設計

