release time :2022-03-25 11:24

2 asynchronous examples
* void has no return value
* has a return value


Spring boot comes with @Async annotation, just add it to the method you want to be asynchronous. There is a small pit, that is, it is only a synchronous service, and the @EnableAsync annotation needs to be added to the main method.

```java
@SpringBootApplication
@EnableAsync
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

# Async with no return value
service
```java
@Service
public class AsyncNoReturnImpl implements AsyncNoReturn {

    @Async
    @Override
    public void execAsync1(){
        try{
            Thread.sleep(2000);
            System.out.println("甲睡了2000ms");
        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @Async
    @Override
    public void execAsync2(){
        try{
            Thread.sleep(4000);
            System.out.println("乙睡了4000ms");
        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @Async
    @Override
    public void execAsync3(){
        try{
            Thread.sleep(3000);
            System.out.println("丙睡了3000ms");
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

controller
```java
@GetMapping("/async-no-return")
    public ResponseEntity<String> getAsyncNoReturn() {
        long s = System.currentTimeMillis();
        asyncNoReturn.execAsync1();
        asyncNoReturn.execAsync2();
        asyncNoReturn.execAsync3();
        System.out.println("我执行结束了");
        long costTime = System.currentTimeMillis() - s;
        System.out.println("service async-no-return cost time: " + costTime);
        return ResponseEntity.status(200).body("good");
    }
```

test
```bash
## async-no-return
GET http://localhost:8080/async-no-return

我执行结束了
service async-no-return cost time: 3
甲睡了2000ms
丙睡了3000ms
乙睡了4000ms
```

Because the three services are all asynchronous, the interface returns directly, and it takes almost no time, 3ms. The three services are executed in parallel, and all of them are executed after 4 seconds.

# async with return value
service
```java
@Service
public class AsyncHasReturnImpl implements AsyncHasReturn {

    @Async
    public Future<String> execAsync1(){
        try{
            Thread.sleep(2000);
            System.out.println("甲睡了2000ms");
            return new AsyncResult<String>("甲执行成功了");
        }catch (Exception e){
            e.printStackTrace();
        }
        return null;
    }
    @Async
    public Future<String> execAsync2(){
        try{
            Thread.sleep(4000);
            System.out.println("乙睡了4000ms");
            return new AsyncResult<String>("乙执行成功了");
        }catch (Exception e){
            e.printStackTrace();
        }
        return null;
    }
    @Async
    public Future<String> execAsync3(){
        try{
            Thread.sleep(3000);
            System.out.println("丙睡了3000ms");
            return new  AsyncResult<String>("丙执行结束了");
        }catch (Exception e){
            e.printStackTrace();
        }
        return null;
    }

}
```

controller
```java
@GetMapping("/async-has-return")
    public ResponseEntity<String> getAsyncHasReturn() throws ExecutionException, InterruptedException {
        long s = System.currentTimeMillis();
        Future<String> stringFuture1 = asyncHasReturn.execAsync1();
        Future<String> stringFuture2 = asyncHasReturn.execAsync2();
        Future<String> stringFuture3 = asyncHasReturn.execAsync3();
        long costTime = System.currentTimeMillis() - s;
        System.out.println("执行到代码中间了，cost time: " + costTime);
        System.out.println("我执行结束了" + stringFuture1.get() + stringFuture2.get() + stringFuture3.get());
        costTime = System.currentTimeMillis() - s;
        System.out.println("service async-has-return cost time: " + costTime);
        return ResponseEntity.status(200).body("good");
    }
```

test
```bash
## async-has-return
GET http://localhost:8080/async-has-return

执行到代码中间了，cost time: 1
甲睡了2000ms
丙睡了3000ms
乙睡了4000ms
我执行结束了甲执行成功了乙执行成功了丙执行结束了
service async-has-return cost time: 4006
```

It can be seen that it does not take time to assign an asynchronous service to a Future, and there is no need to wait for the current asynchronous thread to finish executing. Only the get() method needs to wait for the current asynchronous thread to finish executing, and several asynchronous threads are executed in parallel. Similar to the get() method is isDone(), because get() already includes isDone(), so there is no need to use isDone() to make a judgment.

> All the code is at: https://github.com/backendcloud/example/tree/master/spring-boot/async/demo/

