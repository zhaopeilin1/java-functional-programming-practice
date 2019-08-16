参考https://www.baeldung.com/java-try-with-resources
先复习一下try with resources的知识点。
0、介绍：try with resources是java7的一个语法糖，可以让我们用try(){}语句块包围我们打开的资源，然后资源可以自动关闭。无论在花括号{}中的语句块是全部正常执行完，还是有异常抛出，资源都能自动关闭，声明的资源在try语句块外就是不可引用了的，当它不可引用时资源就已经自动关闭了。资源必须是声明实现了AutoCloseable 接口的类，而自动关闭资源是通过调用 AutoCloseable 具体实现类的close方法。

1、在try()里面打开资源，最简单写法

try (PrintWriter writer = new PrintWriter(new File("test.txt"))) {
    writer.println("Hello World");
}

2、用 try-with-resources 替换 try–catch-finally 

旧写法：

Scanner scanner = null;
try {
    scanner = new Scanner(new File("test.txt"));
    while (scanner.hasNext()) {
        System.out.println(scanner.nextLine());
    }
} catch (FileNotFoundException e) {
    e.printStackTrace();
} finally {
    if (scanner != null) {
        scanner.close();
    }
}

新写法：

try (Scanner scanner = new Scanner(new File("test.txt"))) {
    while (scanner.hasNext()) {
        System.out.println(scanner.nextLine());
    }
} catch (FileNotFoundException fnfe) {
    fnfe.printStackTrace();
}

3、try语句中打开多个资源，使用分号分隔

try (Scanner scanner = new Scanner(new File("testRead.txt"));
    PrintWriter writer = new PrintWriter(new File("testWrite.txt"))) {
    while (scanner.hasNext()) {
    writer.print(scanner.nextLine());
    }
}

4、编写自定义的可自动关闭的资源类

实现了Closeable(这个接口 extends AutoCloseable ) 或 AutoCloseable接口的类变量可以包在try的圆括号中。也就是説，我们可以自己实现这接口，来使我们的业务类可以实现try with resource这样免于手动关闭。

public class MyResource implements AutoCloseable {
    @Override
    public void close() throws Exception {
        System.out.println("Closed MyResource");
    }
}

5、打开多个资源时，资源关闭顺序

类似于栈的行为，先打开的资源，后关闭。try( A a = new A();B b= new B())这样关闭的时候，先调用 b.close(),再调a.close()。打开资源的顺序当然是代码的自然顺序，先a再b啦。

6、使用了try with resources，仍然可以有catch 块和finally块。finally块中的语句在资源关闭之后才执行。

public class A implements AutoCloseable {
    @Override
 public void close() throws Exception {
        System.out.println("a close");
 }
}
public class B implements AutoCloseable {
    @Override
 public void close() throws Exception {
        System.out.println("b close");
 }
}
public class TryWithResourcesTest {
    public static void main(String[] args) {
        try (A a = new A();B b = new B()){
            System.out.println("do some");
 }catch (Exception e){
            e.printStackTrace();
 }finally {
            System.out.println("finally");
 }
    }
}


RedisTools可以加锁和解锁
//加锁方法
public static boolean tryGetDistributedLock(RedisTemplate redisTemplate, String lockKey, String requestId, int expireTime)
//解锁方法

public static boolean releaseDistributedLock(RedisTemplate redisTemplate, String lockKey, String requestId)
通常的使用方法如下 

boolean getLock =RedisTools.tryGetDistributedLock(args...);
if(getLock){

try{

do some business....

} catch(Exception e ) {

} finally {

RedisTools.releaseDistributedLock(args...)
}

}

我用 grep -R -B4 -A4  tryGetDistributedLock  all.git 搜索我们的代码库，发现了很多这样的代码。俺寻思，可不可以优化一下。想来想去，可以使用try with resources来实现自动释放的锁，可以减少finally那段显式解锁的代码块。

1、新建一个RedisTools的内部类 public static Lock {}，实现AutoCloseable 接口，增加属性 

private boolean success;//是否成功获取锁
然后 RedisTools增加方法。这个方法和前面那个获取锁的方法不同只在于返回值不同，由于方法签名不同，但方法参数是相同的，只能改方法名，加个2字上去。

public static Lock tryGetDistributedLock2(RedisTemplate redisTemplate, String lockKey, String requestId, int expireTime)。函数定义出来了，函数体写法也就自然而然地确定下来了。

修改Lock类，增加属性。以及实现close方法

public static class Lock implements AutoCloseable {
    /**
 * 是否成功获取锁
 */
 private boolean success;
 /**
 * 真正关闭资源时需要的参数
 */
 private RedisTemplate redisTemplate;
 /**
 * 真正关闭资源时需要的参数
 */
 private String lockKey;
 /**
 * 真正关闭资源时需要的参数
 */
 private String requestId;

 Lock(boolean success,RedisTemplate redisTemplate, String lockKey, String requestId){
        this.success = success;
 this.redisTemplate = redisTemplate;
 this.lockKey = lockKey;
 this.requestId = requestId;
 }

    /**
 * 外部调用者通过这个方法判断是否获取到锁。
 * @return
 */
 public boolean isSuccess() {
        return success;
 }

    @Override
 public void close() {
        if(success){
            //调用释放锁的方法。
 releaseDistributedLock(redisTemplate, lockKey, requestId);
 }
    }
}
然后，实现获取锁的方法体

public static Lock tryGetDistributedLock2(RedisTemplate redisTemplate, String lockKey, String requestId, int expireTime) {
    boolean result = tryGetDistributedLock(redisTemplate, lockKey, requestId, expireTime);
 return new Lock(result,redisTemplate,lockKey,requestId);
}
最后，使用这个新的获取锁的方法

try (RedisTools.Lock lock= RedisTools.tryGetDistributedLock2(redisTemplate, redisKey, requestId, 2 * 60 * 1000)){
if(lock.isSuccess()){

//do some business...

}

}



总结： 我们可以充分使用学到的编程知识，融会贯通，编写更优雅的代码，充分地进行最佳实践。现在我们彻底掌握了try with resources并有了一个鲜活的应用安全，下一次再碰到类似的try的语句块，我们也完全可以做相应的代码优化改造了。




