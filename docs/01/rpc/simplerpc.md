# RPC

## 原理及解析

RPC:即远程过程调用

跨进程交互形式：Restful,webservice,HTTP,基于DB做数据交换，基于MQ做数据交换，以及RPC。



![image-20210207113447631](C:\Users\张威威\AppData\Roaming\Typora\typora-user-images\image-20210207113447631.png)

![image-20210207113529271](C:\Users\张威威\AppData\Roaming\Typora\typora-user-images\image-20210207113529271.png)

![image-20210207115459476](C:\Users\张威威\AppData\Roaming\Typora\typora-user-images\image-20210207115459476.png)

![image-20210207123404063](C:\Users\张威威\AppData\Roaming\Typora\typora-user-images\image-20210207123404063.png)



## 第一步:创建工程、制定协议、通用工具方法

![image-20210207141744833](C:\Users\张威威\AppData\Roaming\Typora\typora-user-images\image-20210207141744833.png)

### 1、项目配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.zww</groupId>
    <artifactId>Test-RPC</artifactId>
    <packaging>pom</packaging>
    <version>1.0-SNAPSHOT</version>
    <modules>
        <module>Test-RPC-common</module>
        <module>Test-RPC-proto</module>
        <module>Test-RPC-codec</module>
        <module>Test-RPC-transport</module>
        <module>Test-RPC-server</module>
        <module>Test-RPC-client</module>
    </modules>

    <properties>
        <java.version>1.8</java.version>
        <commons.version>2.5</commons.version>
        <jetty.version>9.4.19.v20190610</jetty.version>
        <fastjson.version>1.2.44</fastjson.version>
        <lombok.verison>1.18.8</lombok.verison>
        <slf4j.version>1.7.26</slf4j.version>
        <logback.version>1.2.3</logback.version>
        <junit.version>4.12</junit.version>
    </properties>


    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>commons-io</groupId>
                <artifactId>commons-io</artifactId>
                <version>${commons.version}</version>
            </dependency>
            <dependency>
                <groupId>org.eclipse.jetty</groupId>
                <artifactId>jetty-servlet</artifactId>
                <version>${jetty.version}</version>
            </dependency>
            <dependency>
                <groupId>com.alibaba</groupId>
                <artifactId>fastjson</artifactId>
                <version>${fastjson.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>${junit.version}</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.verison}</version>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>${slf4j.version}</version>
        </dependency>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>${logback.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
               <artifactId>maven-compiler-plugin</artifactId>
                <version>3.6.1</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### 2、协议代码

项目结构

<img src="C:\Users\张威威\AppData\Roaming\Typora\typora-user-images\image-20210207151558118.png" alt="image-20210207151558118" style="zoom:50%;" />

```java
/**
 * 表示网络传输的一个端点
 * @author zww
 * @date 2021/2/7 14:39
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Peer {
    private String host;
    private int port;
}
```

```java
/**
 * 表示RPC的一个请求
 * @author zww
 * @date 2021/2/7 14:42
 */

public class Request {
    private ServiceDescriptor service;
    private Object[] parameters;
}
```

```java
/**
 * 表示RPC的返回
 * @author zww
 * @date 2021/2/7 14:44
 */
@Data
public class Response {
    /*
    服务返回编码，0-成功，非0失败
     */
    private int code=0;
    private String message="ok";
    private Object data;
}
```

```java
/**
 * 表示服务
 * @author zww
 * @date 2021/2/7 14:40
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
@ToString
public class ServiceDescriptor {
    private String clazz;
    private String method;
    private String returnType;
    private String[] parameterTypes;
}
```

### 3、通用工具代码

```java
/**
 * 反射工具类
 * @author zww
 * @date 2021/2/7 14:46
 */

public class ReflectionUtils {
    /*
         根据class创建对象
     */
     public static <T> T newInstance(Class<T> clazz) {
         try {
             return clazz.newInstance();
         } catch (Exception e) {
          throw new IllegalStateException(e);
         }
     }

    /**
     * 获取某个class的公有方法
     * @param clazz
     * @return 当前类声明的共有方法
     */
     public static Method[] getPublicMethods(Class clazz){
         Method[] methods = clazz.getDeclaredMethods();
         List<Method> pmethods = new ArrayList<>();
         for (Method m : methods) {
             if(Modifier.isPublic(m.getModifiers())){
                 pmethods.add(m);
             }
         }
         return pmethods.toArray(new Method[0]);
     }

    /**
     * 调用指定对象的指定方法
     * @param obj 被调用方法的对象
     * @param method 被调用的方法
     * @param args 方法的参数
     * @return 返回结果
     */

     public static Object invoke(Object obj,
                                 Method method,
                                 Object... args){
         try {
             return method.invoke(obj,args);
         } catch (Exception e) {
           throw new IllegalStateException(e);
         }
     }
}
```

#### 3.1、junit4测试 

被测试类

```java
/**
 * @author zww
 * @date 2021/2/7 14:59
 */

public class TestClass {
    private String a(){
        return "a";
    }
    public   String b(){
        return "b";
    }
    protected String c(){
        return "c   ";
    }
}
```

```java
public class ReflectionUtilsTest {


    @Test
    public void newInstance() {
        TestClass t = ReflectionUtils.newInstance(TestClass.class);
        assertNotNull(t);
    }

    @Test
    public void getPiblicMethods() {
        Method[] methods = ReflectionUtils.getPublicMethods(TestClass.class);
        assertEquals(1,methods.length);
         String mname = methods[0].getName();
         assertEquals("b",mname);
    }

    @Test
    public void invoke() {
        Method[] methods = ReflectionUtils.getPublicMethods(TestClass.class);
        Method b = methods[0];
        
        TestClass t = new TestClass();
        Object invoke = ReflectionUtils.invoke(t, b);
        assertEquals("b",invoke);
    }
}
```

## 第二步:实现序列化模块

### 1、添加fastjson依赖

```xml
<dependencies>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
    </dependency>
</dependencies>
```

项目架构

<img src="C:\Users\张威威\AppData\Roaming\Typora\typora-user-images\image-20210207153325684.png" alt="image-20210207153325684" style="zoom: 50%;" />

### 2、序列化代码

```java
public interface Decoder {
        <T> T decode(byte[] bytes,Class<T> clazz);
}
/**
 * 基于json的反序列化实现
 * @author zww
 * @date 2021/2/7 15:20
 */

public class JSONDecoder implements Decoder {
    @Override
    public <T> T decode(byte[] bytes, Class<T> clazz) {
        return JSON.parseObject(bytes,clazz);
    }
}
```

```java
public interface Encoder {

        byte[] encode(Object obj);
}
/**
 * 基于json的反序列化
 * @author zww
 * @date 2021/2/7 15:14
 */

public class JSONEncoder implements Encoder {
    @Override
    public byte[] encode(Object obj) {
        return JSON.toJSONBytes(obj);
    }
}

```

### 3、测试代码

```java
@Test
public void decode() {
    Encoder encoder= new JSONEncoder();
    Decoder decoder = new JSONDecoder();
    TestBean bean = new TestBean();
    bean.setName("zww");
    bean.setAge(18);
    byte[] bytes = encoder.encode(bean);

    TestBean bean2 = decoder.decode(bytes,TestBean.class);
    assertEquals(bean.getName(),bean2.getName());
    assertEquals(bean.getAge(),bean2.getAge());
}
 @Test
    public void encdoe() {
        Encoder encoder = new JSONEncoder();
        TestBean bean = new TestBean();
        bean.setName("zww");
        bean.setAge(18);
        byte[] bytes = encoder.encode(bean);
        assertNotNull(bytes);
    }
    @Data
public class TestBean {
    private String name;
    private int age;

}

```

## 第三步:实现网络模块

### 1、添加依赖

```xml
<dependencies>
    <dependency>
        <groupId>commons-io</groupId>
        <artifactId>commons-io</artifactId>
      <version>2.5</version>
    </dependency>
    <dependency>
        <groupId>org.eclipse.jetty</groupId>
        <artifactId>jetty-servlet</artifactId>
    </dependency>
    <dependency>
        <groupId>com.zww</groupId>
        <artifactId>Test-RPC-proto</artifactId>
        <version>${project.version}</version>
    </dependency>
</dependencies>
```

### 2、网络代码

```java
public interface RequestHandler {
   void onRequest(InputStream recive, OutputStream toResp);
}
```

```java
/**
 * 1、创建链接
 * 2、发送数据，并且等待响应
 * 3、关闭链接
 * @author zww
 * @date 2021/2/7 15:38
 */

public interface TransportClient {
    void connext(Peer peer);
    InputStream write(InputStream data);
    void close();
}
```

```java
/**
 * 1、启动监听端口
 * 2、接收请求
 * 3、关闭监听
 * @author zww
 * @date 2021/2/7 15:38
 */

public interface TransportServer {
   void init(int port,RequestHandler handler);
    void start();
    void stop();
}
```

```java
/**
 * @author zww
 * @date 2021/2/7 15:45
 */
@Slf4j
public class HTTPTransportServer implements TransportServer {
    private RequestHandler handler;
    private Server server;

    @Override
    public void init(int port, RequestHandler handler) {
        this.server = new Server(port);
        this.handler = handler;
        
        //servlet接收请求
        ServletContextHandler ctx = new ServletContextHandler();
        server.setHandler(ctx);
        ServletHolder holder = new ServletHolder(new RequestServlet());
        ctx.addServlet(holder,"/*");
    }

    @Override
    public void start() {
        try {
            server.start();
            server.join();
        } catch (Exception e) {
           log.error(e.getMessage(),e);
        }
    }

    @Override
    public void stop() {
        try {
            server.stop();
        } catch (Exception e) {
            log.error(e.getMessage(),e);
        }
    }
    class RequestServlet extends HttpServlet{
        @Override
        protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
            log.info("client connect！");
          InputStream in = req.getInputStream();
          OutputStream out = resp.getOutputStream();

          if(handler!=null){
              handler.onRequest(in,out);
            }
          out.flush();
        }
    }
}
```

```java
/**
 * @author zww
 * @date 2021/2/7 15:45
 */

public class HTTPTransportClient implements TransportClient {
    private String url;
    @Override
    public void connext(Peer peer) {
        this.url = "http://" + peer.getHost() + ":"
                + peer.getPort();
    }

    @Override
    public InputStream write(InputStream data) {
        try {
            HttpURLConnection httpCnn = (HttpURLConnection) new URL(url).openConnection();
            httpCnn.setDoOutput(true);
            httpCnn.setDoInput(true);
            httpCnn.setUseCaches(false);
            httpCnn.setRequestMethod("POST");

            httpCnn.connect();
            IOUtils.copy(data,httpCnn.getOutputStream());

            int resultCode = httpCnn.getResponseCode();
            if ((resultCode == HttpURLConnection.HTTP_OK))
                return httpCnn.getInputStream();
            else
                return httpCnn.getErrorStream();
        } catch (IOException e) {
           throw new IllegalStateException(e);
        }
    }

    @Override
    public void close() {

    }
}
```

## 第四步:实现Server模块

### 1、添加依赖

```xml
<dependencies>
    <dependency>
        <groupId>com.zww</groupId>
        <artifactId>Test-RPC-codec</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>com.zww</groupId>
        <artifactId>Test-RPC-transport</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>com.zww</groupId>
        <artifactId>Test-RPC-proto</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>com.zww</groupId>
        <artifactId>Test-RPC-common</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>commons-io</groupId>
        <artifactId>commons-io</artifactId>
    </dependency>
</dependencies>
```

### 2、服务端代码

```java
@Data
@Slf4j
public class RpcServer {
    private RpcServerConfig config;
    private TransportServer net;
    private Encoder encoder;
    private Decoder decoder;
    private ServiceManager serviceManager;
    private ServiceInvoker serviceInvoker;

    private RequestHandler handler = new RequestHandler() {
        @Override
        public void onRequest(InputStream in, OutputStream out) {
            Response response = new Response();
            try {
                //byte[] bytes =IO
                byte[] bytes = IOUtils.readFully(in, in.available());
                Request request = decoder.decode(bytes, Request.class);
                log.info("get request: {}",request);
                ServiceInstance instance = serviceManager.lookup(request);
                Object data = serviceInvoker.invoke(instance, request);
                response.setData(data);
            } catch (Exception e) {
                e.printStackTrace();
                log.error(e.getMessage(), e);
                response.setCode(-1);
                response.setMessage("RpcServer error: " +
                        e.getClass().getName()+" "+e.getMessage());
            } finally {

                try {
                    byte[] bytes = encoder.encode(response);
                    out.write(bytes);
                    log.info("response client");
                } catch (IOException e) {
                    log.warn(e.getMessage(),e);
                    e.printStackTrace();
                }
            }
        }
    };

    public RpcServer() {
        this(new RpcServerConfig());
    }

    public RpcServer(RpcServerConfig config) {
        this.config = config;
        //net
        this.net = ReflectionUtils.newInstance(config.getTransportClass());
        this.net.init(config.getPort(), handler);
        //encode
        this.encoder = ReflectionUtils.newInstance(config.getEncoderClass());
        //decode
        this.decoder = ReflectionUtils.newInstance(config.getDecoderClass());
        //service
        this.serviceManager = new ServiceManager();
        this.serviceInvoker = new ServiceInvoker();
    }

    //注册服务
    public <T> void register(Class<T> interfaceClass, T bean) {
        serviceManager.register(interfaceClass, bean);
    }

    public void start() {
        this.net.start();
    }

    public void stop() {
        this.net.stop();
    }
}
```

```java
/**
 * server配置
 * @author zww
 * @date 2021/2/7 16:14
 */
@Data
public class RpcServerConfig {
    private Class<? extends TransportServer> transportClass = HTTPTransportServer.class;
    private Class<? extends Encoder> encoderClass = JSONEncoder.class;
    private Class<? extends Decoder> decoderClass = JSONDecoder.class;
    private int port = 3000;

}
```

```java
/**
 * 表示一个对象的具体服务
 * @author zww
 * @date 2021/2/7 16:42
 */
@Data
@AllArgsConstructor
public class ServiceInstance {
    private Object target;
    private Method method;
}
```

```java
/**
 * 调用具体服务
 * @author zww
 * @date 2021/2/7 17:06
 */

public class ServiceInvoker {
    public Object invoke(ServiceInstance service,
                         Request request) {
        return ReflectionUtils.invoke(
                service.getTarget(),
                service.getMethod(),
                request.getParameters());
    }
}
```

```java
/**
 * 管理rpc暴露的服务：注册服务，查找服务
 * @author zww
 * @date 2021/2/7 16:43
 */
@Slf4j
public class ServiceManager {
    private Map<ServiceDescriptor,ServiceInstance> services;

    public ServiceManager(){
        this.services = new ConcurrentHashMap<>();
    }
    //注册
    //扫描方法与bean绑定放在map中
    public <T> void register(Class<T> interfaceClass,T bean){
        Method[] methods = ReflectionUtils.getPublicMethods(interfaceClass);
        for (Method method : methods) {
            ServiceInstance sis = new ServiceInstance(bean,method);
            ServiceDescriptor sdp = ServiceDescriptor.from(interfaceClass,method);
            services.put(sdp,sis);
            log.info("register service:{}{}",sdp.getClazz(),sdp.getMethod());
        }
    }

    //查找
    public ServiceInstance lookup(Request request){
        ServiceDescriptor sdp = request.getService();
        return services.get(sdp);
    }
}
```

## 第五步:实现Client模块

### 1、添加依赖

```xml
<dependencies>
    <dependency>
        <groupId>com.zww</groupId>
        <artifactId>Test-RPC-codec</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>com.zww</groupId>
        <artifactId>Test-RPC-transport</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>com.zww</groupId>
        <artifactId>Test-RPC-proto</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>com.zww</groupId>
        <artifactId>Test-RPC-common</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>commons-io</groupId>
        <artifactId>commons-io</artifactId>
    </dependency>
</dependencies>
```

### 2、客户端代码

```java
/**
 * @author zww
 * @date 2021/2/21 14:19
 */
@Slf4j
public class RandomTransportSelector implements TransportSelector {
    /**
     * 已经连接好的client
     */
    private  List<TransportClient> clients;

    public RandomTransportSelector(){
        clients = new ArrayList<>();
    }


    @Override
    public  synchronized void init(List<Peer> peers,
                     int count,
                     Class<? extends TransportClient> clazz) {
        count=Math.max(count,1);
        for(Peer peer: peers){
            for (int i = 0; i < count; i++) {
                TransportClient client = ReflectionUtils.newInstance(clazz);
                client.connect(peer);
                clients.add(client);
            }
            log.info("connect server: {}",peer);
        }
    }

    @Override
    public synchronized TransportClient select() {
        int i = new Random().nextInt(clients.size());
        return clients.remove(i);
    }

    @Override
    public synchronized void release(TransportClient client) {
        clients.add(client);
    }

    @Override
    public  synchronized void close() {
        for(TransportClient client : clients){
            client.close();
        }
        clients.clear();
    }
}
```

```java
/**
 * 调用远程服务的代理类
 * @author zww
 * @date 2021/2/21 14:49
 */
@Slf4j
public class RemoteInvoker implements InvocationHandler {
    private Class clazz;
    private Encoder encoder;
    private Decoder decoder;
    private TransportSelector selector;

    RemoteInvoker(Class clazz, Encoder encoder, Decoder decoder,TransportSelector selector){
        this.clazz = clazz;
        this.decoder = decoder;
        this.encoder = encoder;
        this.selector = selector;
    }

    @Override
    public Object invoke(Object proxy,
                         Method method,
                         Object[] args) throws Throwable {
        Request request = new Request();
        request.setService(ServiceDescriptor.from(clazz, method));
        request.setParameters(args);

        Response resp = invokeRemote(request);
        if (resp == null || resp.getCode() != 0) {
            throw new IllegalStateException("fail to invoke remote: " + resp);
        }

        return resp.getData();
    }

    private Response invokeRemote(Request request) {
        Response resp = null;
        TransportClient client = null;

        try {
            client = selector.select();

            byte[] outBytes = encoder.encode(request);
            InputStream recive = client.write(new ByteArrayInputStream(outBytes));

            byte[] inBytes = IOUtils.readFully(recive, recive.available());
            resp = decoder.decode(inBytes, Response.class);

        } catch (IOException e) {
            log.warn(e.getMessage(), e);
            resp = new Response();
            resp.setCode(1);
            resp.setMessage("RpcClient got error: " + e.getClass() + ":" + e.getMessage());
        } finally {
            if (client != null) {
                selector.release(client);
            }
        }
        return resp;
    }
}
```

```java
public class RpcClient {
    private RpcClientConfig config;
    private Encoder encoder;
    private Decoder decoder;
    private TransportSelector selector;

    public RpcClient(){
        this(new RpcClientConfig());
    }

    public RpcClient(RpcClientConfig config){
        this.config=config;

        this.encoder = ReflectionUtils.newInstance(this.config.getEncoderClass());
        this.decoder = ReflectionUtils.newInstance(this.config.getDecoderClass());
        this.selector = ReflectionUtils.newInstance(this.config.getSelectorClass());

        this.selector.init( this.config.getServers(),
                this.config.getConnectCount(),
                this.config.getTransportClass());
    }
    public <T> T getProxy(Class<T> clazz){
        return (T) Proxy.newProxyInstance(
                getClass().getClassLoader(),
                new Class[]{clazz},
                new RemoteInvoker(clazz, encoder, decoder, selector)
        );
    }
}
```

```java
@Data
public class RpcClientConfig {
    private Class<? extends TransportClient> transportClass = HTTPTransportClient.class;
    private Class<? extends Encoder> encoderClass = JSONEncoder.class;
    private Class<? extends Decoder> decoderClass = JSONDecoder.class;
    private Class<? extends TransportSelector> selectorClass = RandomTransportSelector.class;
    private int connectCount = 1;
    private List<Peer> servers = Arrays.asList(
            new Peer("127.0.0.1", 3000)
    );
}
```

```java
public interface TransportSelector {
    /**
     * 初始化selector
     * @param peers  可以连接的server端点信息
     * @param count  client与server建立多少个连接
     * @param clazz client实现class
     */
    void init(List<Peer> peers, int count, Class<? extends TransportClient> clazz);
    /**
     * 选择一个ransport与server做交互
     * @return 网络client
     */
    TransportClient select();

    /**
     * 释放用完的client
     * @param client
     */
    void release(TransportClient client);

    void close();
}
```

## 第六步: gk-rpc使用案例

### 1、添加依赖

```xml
<dependencies>
    <dependency>
        <groupId>com.zww</groupId>
        <artifactId>Test-RPC-client</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>com.zww</groupId>
        <artifactId>Test-RPC-server</artifactId>
        <version>${project.version}</version>
    </dependency>
</dependencies>
```

### 2、测试代码

计算器功能代码：

```java
public interface CalcService {
    int add(int a,int b);
    int minus(int a,int b);
}
```

```java
public class CalcServiceImpl implements CalcService {
    @Override
    public int add(int a, int b) {
        return a+b;
    }
    @Override
    public int minus(int a, int b) {
        return a-b;
    }
}
```

```java
public class Client {
    public static void main(String[] args) {
        RpcClient client = new RpcClient();
        CalcService service = client.getProxy(CalcService.class);
        int r1 = service.add(1, 2);
        int r2 = service.minus(10, 8);
        System.out.println(r1);
        System.out.println(r2);
    }
}
```

```java
public class Server {
    public static void main(String[] args) {
        RpcServer server = new RpcServer();
        server.register(CalcService.class, new CalcServiceImpl());
        server.start();
    }
}
```