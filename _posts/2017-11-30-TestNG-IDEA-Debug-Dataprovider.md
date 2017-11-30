---
layout: post
title:  IDEA TestNG DataProvider问题调试
date:   2017-11-30 23:20:57 +0800
categories: Dev
tag: Java
---

* content
{:toc}

背景
----------------
**记录一次IntelliJ IDEA基于TestNG测试框架使用DataProvider碰到问题的调试过程.**

代码
----------------

注:以下代码经过简化

    @DataProvider(name = "getUrls")
    public Object[][] getUrls() {
        return new Object[][]{
                {"www.baidu.com"},
                {"www.google.com"},
        };
    }

    @Test(dataProvider = "getUrls")
    public void getUrl(String url) {
        System.out.println("GET " + url);
    }

    @DataProvider(name = "postUrls")
    public Object[][] postUrls() {
        return new Object[][]{
                {"www.baidu.com"},
                {"www.google.com"},
        };
    }

    @Test(dataProvider = "postUrls")
    public void postUrl(String url) {
        System.out.println("POST " + url);
    }

问题
----------------

在IDEA中点击绿色三角箭头运行getUrl测试方法， 期望输出

    GET www.baidu.com
    GET www.google.com

实际只输出

    GET www.baidu.com

定位
----------------
1. 代码中有另外一个测试方法postUrl使用dataProvider却能正常输出（也是通过IDEA点击绿色三角箭头运行)， 对比两者实现并无不同.
2. 考虑到getUrls的DataProvider返回的Object[][]可能出问题
   
   - 尝试将getUrl方法对应DataProvider指定到postUrls，即改成
    ```
        @Test(dataProvider = "postUrls")
        public void getUrl(String url)
        {
            System.out.println("GET "+url);
        }
    ```
   - 结果仍然只输出`GET www.baidu.com`
   
   - 继续尝试将postUrl方法对应DataProvider指定到getUrls，即改成
    ```	
        @Test(dataProvider = "getUrls")
        public void postUrl(String url)
        {
            System.out.println("POST "+url);
        }
    ```
   
   - 结果正常输出
    ```	
        POST www.baidu.com
        POST www.google.com
    ```

   - 说明DataProvider的定义没有问题
   
3. 继续将有问题的getUrl向正常的postUrl做趋近修改，将getUrl命名改成postUrl，再次运行发现正常.
将名字修改成其他的， 运行后依然能正常输出.
    - 猜测getUrl名字有问题
4. 直接在命令行中指定getUrl方法单独运行，能正常输出.
    - 推测IDEA工具对getUrl处理问题

调试
----------------

开始对运行测试过程进行调试

环境： `Testng 6.8.8`  `IDEA 2017.2.6` `Java 1.8`

首先由于运行的是测试方法， 在`testng`代码中找到`invokeTestMethods`函数,该函数在`TestMethodWorker.run()`中被调用.

在该函数入口加断点， debug运行， 进入其实现中.

在`invokeTestMethods`函数定位到调用单个方法`invokeTestMethod`函数代码

    while (allParameterValues.hasNext()) {
    …
    invokeTestMethod
    …}
    
查看其循环条件中`allParameterValues`为Object[]类型迭代器，
Object[][] `allParameterValues.m_objects`内只存放了`www.baidu.com`. 所以循环只执行一次，只输出了
`GET www.baidu.com`.
 
定位到`allParameterValues`赋值处

    ParameterBag bag = createParameters(testMethod,
        parameters, allParameterNames, null, suite, testContext, instances[0],
        null);
    ........
    Iterator<Object[]> allParameterValues = bag.parameterHolder.parameters;

Debug `createParameters`函数一层层进入直到`handleParameters`函数，该函数负责填充方法，其注释
      
      /**
       * If the method has parameters, fill them in. Either by using a @DataProvider
       * if any was provided, or by looking up <parameters> in testng.xml
       * @return An Iterator over the values for each parameter of this
       * method.
       */
       
从注释中可以确认其使用`DataProvider`解析参数.
      
      parameters = MethodInvocationHelper.invokeDataProvider(
          instance, /* a test instance or null if the dataprovider is static*/
          dataProviderHolder.method,
          testMethod,
          methodParams.context,
          fedInstance,
          annotationFinder);
      Iterator<Object[]> filteredParameters = filterParameters(parameters,
          testMethod.getInvocationNumbers());

继续step调试， 经过`invokeDataProvider`函数解析后返回的`parameters`是包含了预期参数的.

问题出现在后面的`filterParameters`函数调用，执行后`filteredParameters`只剩下一组参数了.

开始分析`filteredParameters`函数的实现， 其传入的第二个参数为`List<Integer> list`， 即元素为整型的列表， 其使用如下：
  
      /**
       * If numbers is empty, return parameters, otherwise, return a subset of parameters
       * whose ordinal number match these found in numbers.
       */
      static private Iterator<Object[]> filterParameters(Iterator<Object[]> parameters,
          List<Integer> list) {
        if (list.isEmpty()) {
          return parameters;
        } else {
          List<Object[]> result = Lists.newArrayList();
          int i = 0;
          while (parameters.hasNext()) {
            Object[] next = parameters.next();
            if (list.contains(i)) {
              result.add(next);
            }
            i++;
          }
          return new ArrayIterator(result.toArray(new Object[list.size()][]));
        }
      }
      
分析`if(list.contains(i))`代码部分及结合注释可以知道， 该函数使用第二个参数对参数组进行过滤，如果`list`包含数字`i` ,则过滤出参数组索引为`i`的，在该问题环境代码中，`list`只包含一个元素`0`，导致第0组参数`www.baidu.com`被过滤出来，最终只执行了该参数的方法.

对比正常情况`list`应该为`null`， 接下来debug `list`在何处被错误设置的.

`list`为`testMethod.getInvocationNumbers()`返回的`m_invocationNumbers`，在代码中搜索定位到

      public XmlInclude(String n, List<Integer> list, int index) {
        m_name = n;
        m_invocationNumbers = list;
        m_index = index;
      }
      
在该函数中加入断点重新debug，在第二次执行到该处时其被赋值为只包含一个元素`0`的`list`，此时调用栈如下

    "main@1" prio=5 tid=0x1 nid=NA runnable
      java.lang.Thread.State: RUNNABLE
          at org.testng.xml.XmlInclude.<init>(XmlInclude.java:37)
          at org.testng.IDEARemoteTestNG.run(IDEARemoteTestNG.java:59)
    	  at org.testng.RemoteTestNGStarter.main(RemoteTestNGStarter.java:123)

本地没有IDEA的代码，在网上搜索，找到`IDEARemoteTestNG.java`

    for (XmlInclude include : aClass.getIncludedMethods()) {	
        includes.add(new XmlInclude(include.getName(), Collections.singletonList(Integer.parseInt(myParam)), 0));
	}

`XmlInclue`函数在该处调用，并且传入参数`Collections.singletonList`为`list`,`myParam`值为`0` `(通过IDEA在debugger窗口定位到org.testng.IDEARemoteTestNG.run(IDEARemoteTestNG.java:59)后加入myParam监视获得）`

`myParam`赋值

    private final String myParam;	
        public IDEARemoteTestNG(String param) {
        myParam = param;
    }

调用处在`RemoteTestNGStarter.java中`
	
    final IDEARemoteTestNG testNG = new IDEARemoteTestNG(param);

查看`param`赋值处如下

    public static void main(String[] args) throws Exception {	
        int i = 0;
        String param = null;
        String commandFileName = null;
        String workingDirs = null;
        Vector resultArgs = new Vector();
        for (; i < args.length; i++) {
        String arg = args[i];
        if (arg.startsWith("@name")) {
        param = arg.substring(5);
        continue;
        }

`param`通过传入参数`"@name0"`解析出来的.

打开IDEA Console界面发现命令运行打印信息：
    
    ……… org.testng.RemoteTestNGStarter @name0 ……… 

那么问题来了，这个参数是从哪里来的?

打开`IDEA Run/Debug Configuration`界面中`getUrl`测试方法对应窗口， 在`JDK Settings`标签中的`Test runner params`发现写入了个`0`. 
估计是运行测试过程中不小心敲入的:)

**将`0`删除后运行正常,至此问题得到修复**

Test runner params
----------------
从IDEA官网上查看该参数的说明：`Arguments to be passed to the test runner`  

还想知道在哪里给0加上`@name`前缀的.

在IDEA代码中搜索`@name`，定位到`JavaTestFrameworkRunnableState.java`

    protected List<String> getNamedParams(String parameters) {	
        return Collections.singletonList("@name" + parameters);
	}

该函数`getNamedParams`添加的`@name`前缀，而其被调用处`testng/configuration/TestNGRunnableState.java`

      @Override
      protected List<String> getNamedParams(String parameters) {
        try {
          Integer.parseInt(parameters);
          return super.getNamedParams(parameters); //添加@name
        }
        catch (NumberFormatException e) {
          return Arrays.asList(parameters.split(" "));
        }
      }

可知当`Test runner params`参数为数字`${num}`时，传入参数`@name${num}`.不为数字时，直接传参.

在以后使用中，可以通过设置`Test runner params`为数字，使其单独运行DataProvider某一组参数.

扩展
----------------
查看`xmlInclude`函数代码

      private void xmlInclude(boolean start, Attributes attributes) {
        if (start) {
          m_locations.push(Location.INCLUDE);
          m_currentInclude = new Include(attributes.getValue("name"),
              attributes.getValue("invocation-numbers"));
        } else {
          String name = m_currentInclude.name;
          if (null != m_currentIncludedMethods) {
            String in = m_currentInclude.invocationNumbers;
            XmlInclude include;
            if (!Utils.isStringEmpty(in)) {
              include = new XmlInclude(name, stringToList(in), m_currentIncludeIndex++);

可知`invocationNumbers`变量对应到`testng.xml`文件中为`invocation-numbers`，即
`Test runner params`参数为数字时相当于在`testng.xml`中设置`invocation-numbers`，所以也可以通过修改`testng.xml`中`<include name="getUrl"/>`为`<include name="getUrl" invocation-numbers="0"/>`来使用.或者一次多个`<include name="getUrl" invocation-numbers="0 2 5"/>`.

在网上搜索`invocation-numbers`作用

    运行DataProvider某项失败时，在`testng-failed.xml`记录各项失败索引，
    使得你可以无需运行整个测试就可以快速重新运行失败的测试，或者无需运行所有参数，与以上分析一致.

最后附上`IDEA`自动生成的`testng.xml`

    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE suite SYSTEM "http://testng.org/testng-1.0.dtd">
    <suite name="Default Suite">
      <test name="jpaas_web">
        <classes>
          <class name="com.jd.jr.jpaas.qa.springcloud.IntegrationTests.URLsTest">
            <methods>
              <include name="getUrl"/>
            </methods>
          </class> <!-- com.jd.jr.jpaas.qa.springcloud.IntegrationTests.URLsTest -->
        </classes>
      </test> <!-- jpaas_web -->
    </suite> <!-- Default Suite -->

参考
----------
[Run/Debug Configuration: TestNG](https://www.jetbrains.com/help/idea/run-debug-configuration-testng.html)

[Markdown 语法手册 （完整整理版)](http://blog.leanote.com/post/freewalk/Markdown-语法手册)

[TestNG参数化测试-数据提供程序 @DataProvider方式](http://blog.csdn.net/wangxh_haha/article/details/52250649)
