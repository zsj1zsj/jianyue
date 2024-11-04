> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/388501467) ![](https://pica.zhimg.com/v2-d0f7e4e7a539e4c0afe1c11fe0a75bc8_r.jpg)

> 欢迎大家关注公众号「JAVA 前线」查看更多精彩分享文章，主要包括源码分析、实际应用、架构思维、职场分享、产品思考等等，同时欢迎大家加我个人微信「java_front」一起交流学习

1 文章概述
------

责任链模式将请求发送和接收解耦，让多个接收对象都有机会处理这个请求。这些接收对象串成一条链路并沿着这条链路传递这个请求，直到链路上某个接收对象能够处理它。本文我们介绍责任链模式两种应用场景和四种代码实现方式，最后介绍了 DUBBO 如何应用责任链构建过滤器链路。

2 应用场景
------

2.1 命中立即中断
----------

我们实现一个关键词过滤功能。系统设置三个关键词过滤器，输入内容命中任何一个过滤器规则就返回校验不通过，链路立即中断无需继续进行。

(1) 实现方式一
---------

```
public interface ContentFilter {
    public boolean filter(String content);
}

public class AaaContentFilter implements ContentFilter {
    private final static String KEY_CONTENT = "aaa";

    @Override
    public boolean filter(String content) {
        boolean isValid = Boolean.FALSE;
        if (StringUtils.isEmpty(content)) {
            return isValid;
        }
        isValid = !content.contains(KEY_CONTENT);
        return isValid;
    }
}

public class BbbContentFilter implements ContentFilter {
    private final static String KEY_CONTENT = "bbb";

    @Override
    public boolean filter(String content) {
        boolean isValid = Boolean.FALSE;
        if (StringUtils.isEmpty(content)) {
            return isValid;
        }
        isValid = !content.contains(KEY_CONTENT);
        return isValid;
    }
}

public class CccContentFilter implements ContentFilter {
    private final static String KEY_CONTENT = "ccc";

    @Override
    public boolean filter(String content) {
        boolean isValid = Boolean.FALSE;
        if (StringUtils.isEmpty(content)) {
            return isValid;
        }
        isValid = !content.contains(KEY_CONTENT);
        return isValid;
    }
}

```

具体过滤器已经编写完成，接下来构造过滤器责任链路

```
@Service
public class FilterHandlerChain {
    private FilterHandler head = null;
    private FilterHandler tail = null;

    @PostConstruct
    public void init() {
        FilterHandler aaaHandler = new AaaContentFilterHandler();
        FilterHandler bbbHandler = new BbbContentFilterHandler();
        FilterHandler cccHandler = new CccContentFilterHandler();
        addHandler(aaaHandler);
        addHandler(bbbHandler);
        addHandler(cccHandler);
    }

    public void addHandler(FilterHandler handler) {
        if (head == null) {
            head = tail = handler;
        }
        /** 设置当前tail继任者 **/
        tail.setSuccessor(handler);

        /** 指针重新指向tail **/
        tail = handler;
    }

    public boolean filter(String content) {
        if (null == head) {
            throw new RuntimeException("FilterHandlerChain is empty");
        }
        /** head发起调用 **/
        return head.filter(content);
    }
}

public class Test {
    public static void main(String[] args) throws Exception {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[] { "classpath*:META-INF/chain/spring-core.xml" });
        FilterHandlerChain chain = (FilterHandlerChain) context.getBean("filterHandlerChain");
        System.out.println(context);
        boolean result1 = chain.filter("ccc");
        boolean result2 = chain.filter("ddd");
        System.out.println("校验结果1=" + result1);
        System.out.println("校验结果2=" + result2);
    }
}

```

(2) 实现方式二
---------

```
public abstract class FilterHandler {

    /** 下一个节点 **/
    protected FilterHandler successor = null;

    public void setSuccessor(FilterHandler successor) {
        this.successor = successor;
    }

    public final boolean filter(String content) {
        /** 执行自身方法 **/
        boolean isValid = doFilter(content);
        if (!isValid) {
            System.out.println("校验不通过");
            return isValid;
        }
        /** 执行下一个节点链路 **/
        if (successor != null && this != successor) {
            isValid = successor.filter(content);
        }
        return isValid;
    }
    /** 每个节点过滤方法 **/
    protected abstract boolean doFilter(String content);
}

public class AaaContentFilterHandler extends FilterHandler {
    private final static String KEY_CONTENT = "aaa";

    @Override
    protected boolean doFilter(String content) {
        boolean isValid = Boolean.FALSE;
        if (StringUtils.isEmpty(content)) {
            return isValid;
        }
        isValid = !content.contains(KEY_CONTENT);
        return isValid;
    }
}

// 省略其它过滤器代码

```

具体过滤器已经编写完成，接下来构造过滤器责任链路

```
@Service
public class FilterHandlerChain {
    private FilterHandler head = null;
    private FilterHandler tail = null;

    @PostConstruct
    public void init() {
        FilterHandler aaaHandler = new AaaContentFilterHandler();
        FilterHandler bbbHandler = new BbbContentFilterHandler();
        FilterHandler cccHandler = new CccContentFilterHandler();
        addHandler(aaaHandler);
        addHandler(bbbHandler);
        addHandler(cccHandler);
    }

    public void addHandler(FilterHandler handler) {
        if (head == null) {
            head = tail = handler;
        }
        /** 设置当前tail继任者 **/
        tail.setSuccessor(handler);

        /** 指针重新指向tail **/
        tail = handler;
    }

    public boolean filter(String content) {
        if (null == head) {
            throw new RuntimeException("FilterHandlerChain is empty");
        }
        /** head发起调用 **/
        return head.filter(content);
    }
}

public class Test {
    public static void main(String[] args) throws Exception {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[] { "classpath*:META-INF/chain/spring-core.xml" });
        FilterHandlerChain chain = (FilterHandlerChain) context.getBean("filterHandlerChain");
        System.out.println(context);
        boolean result1 = chain.filter("ccc");
        boolean result2 = chain.filter("ddd");
        System.out.println("校验结果1=" + result1);
        System.out.println("校验结果2=" + result2);
    }
}

```

2.2 全链路执行
---------

我们实现一个考题生成功能。在线考试系统根据不同年级生成不同考题。系统设置三个考题生成器，每个生成器都会执行，根据学生年级决定是否生成考题，无需生成则执行下一个生成器。

(1) 实现方式一
---------

```
public interface QuestionGenerator {
    public Question generateQuestion(String gradeInfo);
}

public class AaaQuestionGenerator implements QuestionGenerator {

    @Override
    public Question generateQuestion(String gradeInfo) {
        if (!gradeInfo.equals("一年级")) {
            return null;
        }
        Question question = new Question();
        question.setId("aaa");
        question.setScore(10);
        return question;
    }
}

// 省略其它生成器代码

```

具体生成器已经编写完成，接下来构造生成器责任链路

```
@Service
public class QuestionChain {
    private List<QuestionGenerator> generators = new ArrayList<QuestionGenerator>();

    @PostConstruct
    public void init() {
        QuestionGenerator aaaQuestionGenerator = new AaaQuestionGenerator();
        QuestionGenerator bbbQuestionGenerator = new BbbQuestionGenerator();
        QuestionGenerator cccQuestionGenerator = new CccQuestionGenerator();
        generators.add(aaaQuestionGenerator);
        generators.add(bbbQuestionGenerator);
        generators.add(cccQuestionGenerator);
    }

    public List<Question> generate(String gradeInfo) {
        if (CollectionUtils.isEmpty(generators)) {
            throw new RuntimeException("QuestionChain is empty");
        }
        List<Question> questions = new ArrayList<Question>();
        for (QuestionGenerator generator : generators) {
            Question question = generator.generateQuestion(gradeInfo);
            if (null == question) {
                continue;
            }
            questions.add(question);
        }
        return questions;
    }
}

public class Test {
    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[] { "classpath*:META-INF/chain/spring-core.xml" });
        System.out.println(context);
        QuestionChain chain = (QuestionChain) context.getBean("questionChain");
        List<Question> questions = chain.generate("一年级");
        System.out.println(questions);
    }
}

```

(2) 实现方式二
---------

```
public abstract class GenerateHandler {

    /** 下一个节点 **/
    protected GenerateHandler successor = null;

    public void setSuccessor(GenerateHandler successor) {
        this.successor = successor;
    }

    public final List<Question> generate(String gradeInfo) {
        List<Question> result = new ArrayList<Question>();

        /** 执行自身方法 **/
        Question question = doGenerate(gradeInfo);
        if (null != question) {
            result.add(question);
        }

        /** 执行下一个节点链路 **/
        if (successor != null && this != successor) {
            List<Question> successorQuestions = successor.generate(gradeInfo);
            if (null != successorQuestions) {
                result.addAll(successorQuestions);
            }
        }
        return result;
    }
    /** 每个节点生成方法 **/
    protected abstract Question doGenerate(String gradeInfo);
}

public class AaaGenerateHandler extends GenerateHandler {

    @Override
    protected Question doGenerate(String gradeInfo) {
        if (!gradeInfo.equals("一年级")) {
            return null;
        }
        Question question = new Question();
        question.setId("aaa");
        question.setScore(10);
        return question;
    }
}

// 省略其它生成器代码

```

具体生成器已经编写完成，接下来构造生成器责任链路

```
@Service
public class GenerateChain {
    private GenerateHandler head = null;
    private GenerateHandler tail = null;

    @PostConstruct
    public void init() {
        GenerateHandler aaaHandler = new AaaGenerateHandler();
        GenerateHandler bbbHandler = new BbbGenerateHandler();
        GenerateHandler cccHandler = new CccGenerateHandler();
        addHandler(aaaHandler);
        addHandler(bbbHandler);
        addHandler(cccHandler);
    }

    public void addHandler(GenerateHandler handler) {
        if (head == null) {
            head = tail = handler;
        }
        /** 设置当前tail继任者 **/
        tail.setSuccessor(handler);

        /** 指针重新指向tail **/
        tail = handler;
    }

    public List<Question> generate(String gradeInfo) {
        if (null == head) {
            throw new RuntimeException("GenerateChain is empty");
        }
        /** head发起调用 **/
        return head.generate(gradeInfo);
    }
}

public class Test {
    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[] { "classpath*:META-INF/chain/spring-core.xml" });
        GenerateChain chain = (GenerateChain) context.getBean("generateChain");
        System.out.println(context);
        List<Question> result = chain.generate("一年级");
        System.out.println(result);
    }
}

```

3 DUBBO 构建过滤器链
--------------

生产者和消费者最终执行对象都是过滤器链路最后一个节点，整个链路包含多个过滤器进行业务处理。我们看看生产者和消费者默认过滤器链路。

```
# 生产者过滤器链路
EchoFilter > ClassloaderFilter > GenericFilter > ContextFilter > 
TraceFilter > TimeoutFilter > MonitorFilter > ExceptionFilter > AbstractProxyInvoker

# 消费者过滤器链路
ConsumerContextFilter > FutureFilter > MonitorFilter > DubboInvoker

```

ProtocolFilterWrapper 作为链路生成核心通过匿名类方式构建过滤器链路，我们以消费者构建过滤器链路为例

```
public class ProtocolFilterWrapper implements Protocol {
    private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {

        // invoker = DubboInvoker
        Invoker<T> last = invoker;

        // 查询符合条件过滤器列表
        List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
        if (!filters.isEmpty()) {
            for (int i = filters.size() - 1; i >= 0; i--) {
                final Filter filter = filters.get(i);
                final Invoker<T> next = last;

                // 构造一个简化Invoker
                last = new Invoker<T>() {
                    @Override
                    public Class<T> getInterface() {
                        return invoker.getInterface();
                    }

                    @Override
                    public URL getUrl() {
                        return invoker.getUrl();
                    }

                    @Override
                    public boolean isAvailable() {
                        return invoker.isAvailable();
                    }

                    @Override
                    public Result invoke(Invocation invocation) throws RpcException {
                        // 构造过滤器链路
                        Result result = filter.invoke(next, invocation);
                        if (result instanceof AsyncRpcResult) {
                            AsyncRpcResult asyncResult = (AsyncRpcResult) result;
                            asyncResult.thenApplyWithContext(r -> filter.onResponse(r, invoker, invocation));
                            return asyncResult;
                        } else {
                            return filter.onResponse(result, invoker, invocation);
                        }
                    }

                    @Override
                    public void destroy() {
                        invoker.destroy();
                    }

                    @Override
                    public String toString() {
                        return invoker.toString();
                    }
                };
            }
        }
        return last;
    }

    @Override
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        // RegistryProtocol不构造过滤器链路
        if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
            return protocol.refer(type, url);
        }
        Invoker<T> invoker = protocol.refer(type, url);
        return buildInvokerChain(invoker, Constants.REFERENCE_FILTER_KEY, Constants.CONSUMER);
    }
}

```

4 文章总结
------

面向对象设计有一个重要原则：对扩展开放对修改关闭，我认为这是最重要的面向对象设计原则，责任链模式非常好得体现了这个原则，有助于代码维护和处理复杂场景。

本文我们分析了责任链模式两种场景和四种代码实现方式，最后介绍了 DUBBO 如何应用责任链构建过滤器链路。后续文章继续分析 DUBBO 设计模式实例请继续关注。

> 欢迎大家关注公众号「JAVA 前线」查看更多精彩分享文章，主要包括源码分析、实际应用、架构思维、职场分享、产品思考等等，同时欢迎大家加我个人微信「java_front」一起交流学习