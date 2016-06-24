title: Tomcat 架构探索
date: 2016-06-25 01:13:06
tags: ["Tomcat","Java"]
---
## 前言

花了一个礼拜的时间阅读了 `how tomcat works`，本文基于此书，整理了一下`Tomcat 5`的基本架构，其实也没什么多复杂的东西，无非是解析`Http`请求，然后调用相应的`Servlet`。另推荐看`CSAPP`的网络编程那一章

本文不以别人能看懂为目的，仅作自己备忘笔记只用。

顺便问问有无需要`暑假实习`的？坐标`杭州`，`Java后台`方向。有意可联系我 `threezj@hotmail.com`

## 基本架构

`Tomcat`由两个模块协同合作

- `connector`
- `container`

`connector` 负责解析处理`HTTP`请求，比如说`请求头`,`查询字符串`,`请求参数`之类的。生成`HttpRequest`和`HttpResponse`
之后交给`container`，由它负责调用相应的`Servlet`。
<!--more-->
## Connector

`Tomcat`默认的`Connector`为`HttpConnector`。作为`Connector`必须要实现`Connector`这个接口。

`Tomcat`启动以后会开启一个线程，做一个死循环，通过`ServerSocket`来等待请求。一旦得到请求，生成`Socket`，注意这里`HttpConnector`并不会自己处理`Socket`，而是把它交给`HttpProcessor`。详细看下面代码，这里我只保留了关键代码。

```java
public void run() {
        // Loop until we receive a shutdown command
        while (!stopped) {
            Socket socket = null;
            try {
                socket = serverSocket.accept(); //等待链接
            } catch (AccessControlException ace) {
                log("socket accept security exception", ace);
                continue;
            }
            // Hand this socket off to an appropriate processor
            HttpProcessor processor = createProcessor();
            processor.assign(socket);  //这里是立刻返回的
            // The processor will recycle itself when it finishes
        }
    }
```
注意一点，上面的`processor.assign(socket);`是立刻返回的，并不会阻塞在那里等待。因为Tomcat不可能一次只能处理一个请求，所以是异步的，每个`processor`处理都是一个单独的线程。

### HttpProcessor

上面的代码并没有显示调用`HttpProcessor`的`process`方法，那这个方法是怎么调用的呢？我们来看一下`HttpProcessor`的`run`方法。

```java
public void run() {
        // Process requests until we receive a shutdown signal
        while (!stopped) {
            // Wait for the next socket to be assigned
            Socket socket = await(); 
            if (socket == null)
                continue;
            // Process the request from this socket
            try {
                process(socket);
            } catch (Throwable t) {
                log("process.invoke", t);
            }
            // Finish up this request
            connector.recycle(this);
        }
    }
```
我们发现他是调用`await`方法来阻塞等待获得`socket`方法。而之前`Connector`是调用`assign`分配的，这是什么原因？
下面仔细看`await`和`assign`方法。这两个方法协同合作，当`assign`获取`socket`时会通知`await`然后返回`socket`。
```java
    synchronized void assign(Socket socket) {
        // Wait for the Processor to get the previous Socket
        while (available) {
            try {
                wait();
            } catch (InterruptedException e) {
            }
        }
        // Store the newly available Socket and notify our thread
        this.socket = socket;
        available = true;
        notifyAll();
    }
    private synchronized Socket await() {
        // Wait for the Connector to provide a new Socket
        while (!available) {
            try {
                wait();
            } catch (InterruptedException e) {
            }
        }
        // Notify the Connector that we have received this Socket
        Socket socket = this.socket;
        available = false;
        notifyAll();
        return (socket);
    }
```
默认`available`为`false`。

接下来就是剩下的事情就是解析请求，填充`HttpRequest`和`HttpResponse`对象，然后交给`container`负责。
这里我不过多赘述如何解析
```java
private void process(Socket socket) {
    //parse
    ....
    connector.getContainer().invoke(request, response);
    ....
}
```
## Container

> A Container is an object that can execute requests received from a client, and return responses based on those requests 

`Container`是一个接口，实现了这个接口的类的实例，可以处理接收的请求，调用对应的`Servlet`。

总共有四类`Container`，这四个`Container`之间并不是平行关系，而是父子关系

- `Engine` - 最顶层的容器，可以包含多个`Host`
- `Host` - 代表一个虚拟主机，可以包含多个`Context`
- `Context` - 代表一个`web应用`，也就是`ServletContext`，可以包含多个`Wrappers`
- `Wrapper` - 代表一个`Servlet`,不能包含别的容器了，这是最底层

### Container的调用

容器好比是一个加工厂，加工接受的`request`，加工方式和流水线也很像，但又有点区别。这里会用到一个叫做`Pipeline`的 东西，中文翻译为`管道`，`request`就放在管道里顺序加工，进行加工的工具叫做`Valve`，好比手术刀，`Pipeline`可添加多个`Valve`,最后加工的工具称为`BaseValve`

上面可能讲的比较抽象，接下来我们来看代码。`Engine`是顶层容器，所以上面`invoke`，执行的就是`Engine`的方法。`StandardEngine`是`Engine`的默认实现，注意它也同时实现了`Pipeline`接口，且包含了`Pipeline`。

它的构造方法同时指定了`baseValve`,也就是管道最后一个调用的`Valve`
```java
public StandardEngine() {
    super();
    pipeline.setBasic(new StandardEngineValve());
}
```

好，接着我们看`invoke`,这个方法是继承自`ContainerBase`。只有一行，之间交给`pipeline`，进行加工。
```java
public void invoke(Request request, Response response)
        throws IOException, ServletException {
        pipeline.invoke(request, response);
}
```

下面是`StandardPipeline`的`invoke`实现，也就是默认的`pipeline`实现。
```java
public void invoke(Request request, Response response)
        throws IOException, ServletException {
        // Invoke the first Valve in this pipeline for this request
        (new StandardPipelineValveContext()).invokeNext(request, response);
}
```
也只有一行！调用`StandardPipelineValveContext`的`invokeNext`方法，这是一个`pipeline`的内部类。让我们来看
具体代码
```java
public void invokeNext(Request request, Response response)
            throws IOException, ServletException {
            int subscript = stage;
            stage = stage + 1;
            // Invoke the requested Valve for the current request thread
            if (subscript < valves.length) {
                valves[subscript].invoke(request, response, this);  //加工
            } else if ((subscript == valves.length) && (basic != null)) {
                basic.invoke(request, response, this);
            } else {
                throw new ServletException
                    (sm.getString("standardPipeline.noValve"));
            }
}
```

它调用了`pipeline`所用的`Valve`来对`request`做加工，当Valve执行完，会调用`BaseValve`,也就是上面的`StandardEngineValve`，
我们再来看看它的`invoke`方法
```java
    // Select the Host to be used for this Request
    StandardEngine engine = (StandardEngine) getContainer();
    Host host = (Host) engine.map(request, true);
    if (host == null) {
        ((HttpServletResponse) response.getResponse()).sendError
            (HttpServletResponse.SC_BAD_REQUEST,
                sm.getString("standardEngine.noHost",
                            request.getRequest().getServerName()));
        return;
    }
    // Ask this Host to process this request
    host.invoke(request, response);
```
它通过`(Host) engine.map(request, true);`获取所对应的`Host`,然后进入到下一层容器中继续执行。后面的执行顺序
和`Engine`相同，我不过多赘述

#### 执行顺序小结

经过一长串的`invoke`终于讲完了第一层容器的执行顺序。估计你们看的有点晕，我这里小结一下。

>Connector -> HttpProcessor.process() -> StandardEngine.invoke() -> StandardPipeline.invoke() -> 
StandardPipelineValveContext.invokeNext() -> valves.invoke() -> StandardEngineValve.invoke() ->
StandardHost.invoke()

到这里位置`Engine`这一层结束。接下来进行`Host`，步骤完全一致

>StandardHost.invoke() -> StandardPipeline.invoke() -> 
StandardPipelineValveContext.invokeNext() -> valves.invoke() -> StandardHostValve.invoke() ->
StandardContext.invoke()

然后再进行`Context`这一层的处理，到最后选择对应的`Wrapping`执行。

## Wrapper

`Wrapper`相当于一个`Servlet`实例，`StandardContext`会更根据的`request`来选择对应的`Wrapper`调用。我们直接来看看
`Wrapper`的`basevalve`是如果调用`Servlet`的`service`方法的。下面是`StandardWrapperValve`的`invoke`方法，我省略了很多，
只看关键。

```java
public void invoke(Request request, Response response,
                       ValveContext valveContext)
        throws IOException, ServletException {
        // Allocate a servlet instance to process this request
        if (!unavailable) {
            servlet = wrapper.allocate();
        }
        // Create the filter chain for this request
        ApplicationFilterChain filterChain =
            createFilterChain(request, servlet);
        // Call the filter chain for this request
        // NOTE: This also calls the servlet's service() method
        String jspFile = wrapper.getJspFile();  //是否是jsp
        if (jspFile != null)
            sreq.setAttribute(Globals.JSP_FILE_ATTR, jspFile);
        else
            sreq.removeAttribute(Globals.JSP_FILE_ATTR);
        if ((servlet != null) && (filterChain != null)) {
            filterChain.doFilter(sreq, sres);
        }
        sreq.removeAttribute(Globals.JSP_FILE_ATTR);
    }
```

首先调用`wrapper.allocate()`,这个方法很关键，它会通过`反射`找到对应`servlet`的`class`文件，构造出实例返回给我们。然后创建一个`FilterChain`，熟悉`j2ee`的各位应该对这个不陌生把？这就是我们在开发`web app`时使用的`filter`。然后就执行`doFilter`方法了，它又会调用`internalDoFilter`，我们来看这个方法

```java
 private void internalDoFilter(ServletRequest request, ServletResponse response)
        throws IOException, ServletException {
        // Call the next filter if there is one
        if (this.iterator.hasNext()) {
            ApplicationFilterConfig filterConfig =
              (ApplicationFilterConfig) iterator.next();
            Filter filter = null;
            
            filter = filterConfig.getFilter();
            filter.doFilter(request, response, this);
            return;
        }
        // We fell off the end of the chain -- call the servlet instance
        if ((request instanceof HttpServletRequest) &&
            (response instanceof HttpServletResponse)) {
            servlet.service((HttpServletRequest) request,
                            (HttpServletResponse) response);
        } else {
            servlet.service(request, response);
        }
}
```

终于，在这个方法里看到了`service`方法，现在你知道在使用`filter`的时候如果不执行`doFilter`，`service`就不会执行的原因了把。

## 小结

`Tomcat`的重要过程应该都在这里了，还值得一提的是`LifeCycle`接口，这里所有类几乎都实现了`LifeCycle`，`Tomcat`通过它来统一管理容器的生命流程，大量运用观察者模式。有兴趣的同学可以自己看书

## Referance

`How Tomcat works`