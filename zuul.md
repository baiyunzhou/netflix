# zuul
## monitoring
### Tracer.java

```java
package com.netflix.zuul.monitoring;

/**
 * 跟踪器
 */
public interface Tracer {

    void stopAndLog();

    void setName(String name);

}
```

### TracerFactory.java

```java

package com.netflix.zuul.monitoring;

/**
 * 跟踪器工厂
 */
public abstract class TracerFactory {

    private static TracerFactory INSTANCE;

    public static final void initialize(TracerFactory f) {
        INSTANCE = f;
    }

    public static final TracerFactory instance() {
        if(INSTANCE == null) throw new IllegalStateException(String.format("%s not initialized", TracerFactory.class.getSimpleName()));
        return INSTANCE;
    }

    public abstract Tracer startMicroTracer(String name);

}
```

### CounterFactory.java

```java
package com.netflix.zuul.monitoring;

/**
 * 计数器工厂
 */
public abstract class CounterFactory {

    private static CounterFactory INSTANCE;

    public static final void initialize(CounterFactory f) {
        INSTANCE = f;
    }

    public static final CounterFactory instance() {
        if(INSTANCE == null) throw new IllegalStateException(String.format("%s not initialized", CounterFactory.class.getSimpleName()));
        return INSTANCE;
    }

    public abstract void increment(String name);

}
```

### MonitoringHelper.java

```java
package com.netflix.zuul.monitoring;

/**
 * 监控帮助类,虚实现
 */
public class MonitoringHelper {

    public static final void initMocks() {
        CounterFactory.initialize(new CounterFactoryImpl());
        TracerFactory.initialize(new TracerFactoryImpl());
    }

    private static final class CounterFactoryImpl extends CounterFactory {
        @Override
        public void increment(String name) {}
    }

    private static final class TracerFactoryImpl extends TracerFactory {
        @Override
        public Tracer startMicroTracer(String name) {
            return new TracerImpl();
        }
    }

    private static final class TracerImpl implements Tracer {
        @Override
        public void setName(String name) {}

        @Override
        public void stopAndLog() {}
    }

}
```

## ExecutionStatus.java

```java
package com.netflix.zuul;
/**
 * 执行状态
 */
public enum ExecutionStatus {

    SUCCESS (1), SKIPPED(-1), DISABLED(-2), FAILED(-3);
    
    private int status;

    ExecutionStatus(int status) {
        this.status = status;
    }
}
```

## ZuulFilterResult.jaa

```java
package com.netflix.zuul;
/**
 * zuul过滤器执行结果
 */
public final class ZuulFilterResult {
        
    private Object result;
    private Throwable exception;
    private ExecutionStatus status;
    
    public ZuulFilterResult(Object result, ExecutionStatus status) {
        this.result = result;
        this.status = status;
    }
    
    public ZuulFilterResult(ExecutionStatus status) {
        this.status = status;
    }

    public ZuulFilterResult() {
        this.status = ExecutionStatus.DISABLED;
    }
  
}

```

## IZuulFilter.java

```java
package com.netflix.zuul;

import com.netflix.zuul.exception.ZuulException;

/**
 * zuul过滤器基础接口
 */
public interface IZuulFilter {

    boolean shouldFilter();

    Object run() throws ZuulException;

}
```

### ZuulFilter.java

```java
package com.netflix.zuul;
/**
 * zuul过滤器基本抽象实现
 */
public abstract class ZuulFilter implements IZuulFilter, Comparable<ZuulFilter> {

    private final AtomicReference<DynamicBooleanProperty> filterDisabledRef = new AtomicReference<>();

    abstract public String filterType();

    abstract public int filterOrder();

    public boolean isStaticFilter() {
        return true;
    }

    public String disablePropertyName() {
        return "zuul." + this.getClass().getSimpleName() + "." + filterType() + ".disable";
    }

    public boolean isFilterDisabled() {
        filterDisabledRef.compareAndSet(null, DynamicPropertyFactory.getInstance().getBooleanProperty(disablePropertyName(), false));
        return filterDisabledRef.get().get();
    }

    public ZuulFilterResult runFilter() {
        ZuulFilterResult zr = new ZuulFilterResult();
        if (!isFilterDisabled()) {
            if (shouldFilter()) {
                Tracer t = TracerFactory.instance().startMicroTracer("ZUUL::" + this.getClass().getSimpleName());
                try {
                    Object res = run();
                    zr = new ZuulFilterResult(res, ExecutionStatus.SUCCESS);
                } catch (Throwable e) {
                    t.setName("ZUUL::" + this.getClass().getSimpleName() + " failed");
                    zr = new ZuulFilterResult(ExecutionStatus.FAILED);
                    zr.setException(e);
                } finally {
                    t.stopAndLog();
                }
            } else {
                zr = new ZuulFilterResult(ExecutionStatus.SKIPPED);
            }
        }
        return zr;
    }
	//默认使用order判断相等
    public int compareTo(ZuulFilter filter) {
        return Integer.compare(this.filterOrder(), filter.filterOrder());
    }
}
```

#### StaticResponseFilter.java

```java
package com.netflix.zuul.filters
/**
 * 响应静态数据的过滤器，执行这个会跳过route过滤器
 */
public abstract class StaticResponseFilter extends ZuulFilter {

    /**
     * Define a URI eg /static/content/path or List of URIs for this filter to return a static response.
     * @return String URI or java.util.List of URIs
     */
    abstract def uri()

    abstract String responseBody()

    @Override
    String filterType() {
        return "static"
    }

    @Override
    int filterOrder() {
        return 0
    }

    boolean shouldFilter() {
        String path = RequestContext.currentContext.getRequest().getRequestURI()
        if (checkPath(path)) return true
        if (checkPath("/" + path)) return true
        return false
    }

    boolean checkPath(String path) {
        def uri = uri()
        if (uri instanceof String) {
            return uri.equals(path)
        } else if (uri instanceof List) {
            return uri.contains(path)
        } else if (uri instanceof Pattern) {
            return uri.matcher(path).matches();
        }
        return false;
    }

    @Override
    Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        // Set the default response code for static filters to be 200
        ctx.setResponseStatusCode(HttpServletResponse.SC_OK)
        // first StaticResponseFilter instance to match wins, others do not set body and/or status
        if (ctx.getResponseBody() == null) {
            ctx.setResponseBody(responseBody())
            ctx.sendZuulResponse = false;
        }
    }

}

```

#### SurgicalDebugFilter.java

```java
package com.netflix.zuul.filters
/**
 * 跟eureka相关的debug过滤器
 */
public abstract class SurgicalDebugFilter extends ZuulFilter {

    abstract def boolean patternMatches()

    @Override
    String filterType() {
        return "pre"
    }

    @Override
    int filterOrder() {
        return 99
    }

    boolean shouldFilter() {

        DynamicBooleanProperty debugFilterShutoff = DynamicPropertyFactory.getInstance().getBooleanProperty(ZuulConstants.ZUUL_DEBUGFILTERS_DISABLED, false);

        if (debugFilterShutoff.get()) return false;

        if (isFilterDisabled()) return false;


        String isSurgicalFilterRequest = RequestContext.currentContext.getRequest().getHeader(ZuulHeaders.X_ZUUL_SURGICAL_FILTER);
        if ("true".equals(isSurgicalFilterRequest)) return false; // dont' apply filter if it was already applied
        return patternMatches();
    }


    @Override
    Object run() {
        DynamicStringProperty routeVip = DynamicPropertyFactory.getInstance().getStringProperty(ZuulConstants.ZUUL_DEBUG_VIP, null);
        DynamicStringProperty routeHost = DynamicPropertyFactory.getInstance().getStringProperty(ZuulConstants.ZUUL_DEBUG_HOST, null);

        if (routeVip.get() != null || routeHost.get() != null) {
            //RequestContext.currentContext.routeHost = routeHost.get();
            RequestContext.getCurrentContext().setRouteHost( new URL(routeHost.get()));
            //RequestContext.currentContext.routeVIP = routeVip.get();
            RequestContext.getCurrentContext().setRouteHost( new URL(routeVip.get()));
            RequestContext.currentContext.addZuulRequestHeader(ZuulHeaders.X_ZUUL_SURGICAL_FILTER, "true");
            if (HTTPRequestUtils.getInstance().getQueryParams() == null) {
                RequestContext.getCurrentContext().setRequestQueryParams(new HashMap<String, List<String>>());
            }
            HTTPRequestUtils.getInstance().getQueryParams().put("debugRequest", ["true"])
            RequestContext.currentContext.setDebugRequest(true)
            RequestContext.getCurrentContext().zuulToZuul = true

        }
    }
}
```

## Debug.java

```java
package com.netflix.zuul.context;
/**
 * Simple wrapper class around the RequestContext for setting and managing Request level Debug data.
 * @author Mikey Cohen
 * Date: 1/25/12
 * Time: 2:26 PM
 */
public class Debug {

    public static void setDebugRequest(boolean bDebug) {
        RequestContext.getCurrentContext().setDebugRequest(bDebug);
    }

    public static void setDebugRequestHeadersOnly(boolean bHeadersOnly) {
        RequestContext.getCurrentContext().setDebugRequestHeadersOnly(bHeadersOnly);
    }

    public static boolean debugRequestHeadersOnly() {
        return RequestContext.getCurrentContext().debugRequestHeadersOnly();
    }


    public static void setDebugRouting(boolean bDebug) {
        RequestContext.getCurrentContext().setDebugRouting(bDebug);
    }


    public static boolean debugRequest() {
        return RequestContext.getCurrentContext().debugRequest();
    }

    public static boolean debugRouting() {
        return RequestContext.getCurrentContext().debugRouting();
    }

    public static void addRoutingDebug(String line) {
        List<String> rd = getRoutingDebug();
        rd.add(line);
    }

    /**
     *
     * @return Returns the list of routiong debug messages
     */
    public static List<String> getRoutingDebug() {
        List<String> rd = (List<String>) RequestContext.getCurrentContext().get("routingDebug");
        if (rd == null) {
            rd = new ArrayList<String>();
            RequestContext.getCurrentContext().set("routingDebug", rd);
        }
        return rd;
    }

    /**
     * Adds a line to the  Request debug messages
     * @param line
     */
    public static void addRequestDebug(String line) {
        List<String> rd = getRequestDebug();
        rd.add(line);
    }

    /**
     *
     * @return returns the list of request debug messages
     */
    public static List<String> getRequestDebug() {
        List<String> rd = (List<String>) RequestContext.getCurrentContext().get("requestDebug");
        if (rd == null) {
            rd = new ArrayList<String>();
            RequestContext.getCurrentContext().set("requestDebug", rd);
        }
        return rd;
    }


    /**
     * Adds debug details about changes that a given filter made to the request context.
     * @param filterName
     * @param copy
     */
    public static void compareContextState(String filterName, RequestContext copy) {
        RequestContext context = RequestContext.getCurrentContext();
        Iterator<String> it = context.keySet().iterator();
        String key = it.next();
        while (key != null) {
            if ((!key.equals("routingDebug") && !key.equals("requestDebug"))) {
                Object newValue = context.get(key);
                Object oldValue = copy.get(key);
                if (oldValue == null && newValue != null) {
                    addRoutingDebug("{" + filterName + "} added " + key + "=" + newValue.toString());
                } else if (oldValue != null && newValue != null) {
                    if (!(oldValue.equals(newValue))) {
                        addRoutingDebug("{" +filterName + "} changed " + key + "=" + newValue.toString());
                    }
                }
            }
            if (it.hasNext()) {
                key = it.next();
            } else {
                key = null;
            }
        }

    }
}
```

## RequestContext.java

```java
package com.netflix.zuul.context;
/**
 * 请求上下文
 */
public class RequestContext extends ConcurrentHashMap<String, Object> {

    private static final Logger LOG = LoggerFactory.getLogger(RequestContext.class);

    protected static Class<? extends RequestContext> contextClass = RequestContext.class;

    private static RequestContext testContext = null;

    protected static final ThreadLocal<? extends RequestContext> threadLocal = new ThreadLocal<RequestContext>() {
        @Override
        protected RequestContext initialValue() {
            try {
                return contextClass.newInstance();
            } catch (Throwable e) {
                throw new RuntimeException(e);
            }
        }
    };


    public RequestContext() {
        super();
    }

    public static void setContextClass(Class<? extends RequestContext> clazz) {
        contextClass = clazz;
    }

    public static void testSetCurrentContext(RequestContext context) {
        testContext = context;
    }

    public static RequestContext getCurrentContext() {
        if (testContext != null) return testContext;

        RequestContext context = threadLocal.get();
        return context;
    }

    public boolean getBoolean(String key) {
        return getBoolean(key, false);
    }

    public boolean getBoolean(String key, boolean defaultResponse) {
        Boolean b = (Boolean) get(key);
        if (b != null) {
            return b.booleanValue();
        }
        return defaultResponse;
    }

    public void set(String key) {
        put(key, Boolean.TRUE);
    }

    public void set(String key, Object value) {
        if (value != null) put(key, value);
        else remove(key);
    }

    public boolean getZuulEngineRan() {
        return getBoolean("zuulEngineRan");
    }

    public void setZuulEngineRan() {
        put("zuulEngineRan", true);
    }

    public HttpServletRequest getRequest() {
        return (HttpServletRequest) get("request");
    }

    public void setRequest(HttpServletRequest request) {
        put("request", request);
    }

    public HttpServletResponse getResponse() {
        return (HttpServletResponse) get("response");
    }

    public void setResponse(HttpServletResponse response) {
        set("response", response);
    }

    public Throwable getThrowable() {
        return (Throwable) get("throwable");

    }

    public void setThrowable(Throwable th) {
        put("throwable", th);

    }

    public void setDebugRouting(boolean bDebug) {
        set("debugRouting", bDebug);
    }

    public boolean debugRouting() {
        return getBoolean("debugRouting");
    }

    public void setDebugRequestHeadersOnly(boolean bHeadersOnly) {
        set("debugRequestHeadersOnly", bHeadersOnly);

    }

    public boolean debugRequestHeadersOnly() {
        return getBoolean("debugRequestHeadersOnly");
    }

    public void setDebugRequest(boolean bDebug) {
        set("debugRequest", bDebug);
    }

    public boolean debugRequest() {
        return getBoolean("debugRequest");
    }

    public void removeRouteHost() {
        remove("routeHost");
    }

    public void setRouteHost(URL routeHost) {
        set("routeHost", routeHost);
    }

    public URL getRouteHost() {
        return (URL) get("routeHost");
    }

    public void addFilterExecutionSummary(String name, String status, long time) {
            StringBuilder sb = getFilterExecutionSummary();
            if (sb.length() > 0) sb.append(", ");
            sb.append(name).append('[').append(status).append(']').append('[').append(time).append("ms]");
    }

    public StringBuilder getFilterExecutionSummary() {
        if (get("executedFilters") == null) {
            putIfAbsent("executedFilters", new StringBuilder());
        }
        return (StringBuilder) get("executedFilters");
    }

    public void setResponseBody(String body) {
        set("responseBody", body);
    }

    public String getResponseBody() {
        return (String) get("responseBody");
    }

    public void setResponseDataStream(InputStream responseDataStream) {
        set("responseDataStream", responseDataStream);
    }

    public void setResponseGZipped(boolean gzipped) {
        put("responseGZipped", gzipped);
    }

    public boolean getResponseGZipped() {
        return getBoolean("responseGZipped", true);
    }

    public InputStream getResponseDataStream() {
        return (InputStream) get("responseDataStream");
    }

    public boolean sendZuulResponse() {
        return getBoolean("sendZuulResponse", true);
    }

    public void setSendZuulResponse(boolean bSend) {
        set("sendZuulResponse", Boolean.valueOf(bSend));
    }

    public int getResponseStatusCode() {
        return get("responseStatusCode") != null ? (Integer) get("responseStatusCode") : 500;
    }

    public void setResponseStatusCode(int nStatusCode) {
        getResponse().setStatus(nStatusCode);
        set("responseStatusCode", nStatusCode);
    }

    public void addZuulRequestHeader(String name, String value) {
        getZuulRequestHeaders().put(name.toLowerCase(), value);
    }

    public Map<String, String> getZuulRequestHeaders() {
        if (get("zuulRequestHeaders") == null) {
            HashMap<String, String> zuulRequestHeaders = new HashMap<String, String>();
            putIfAbsent("zuulRequestHeaders", zuulRequestHeaders);
        }
        return (Map<String, String>) get("zuulRequestHeaders");
    }

    public void addZuulResponseHeader(String name, String value) {
        getZuulResponseHeaders().add(new Pair<String, String>(name, value));
    }

    public List<Pair<String, String>> getZuulResponseHeaders() {
        if (get("zuulResponseHeaders") == null) {
            List<Pair<String, String>> zuulRequestHeaders = new ArrayList<Pair<String, String>>();
            putIfAbsent("zuulResponseHeaders", zuulRequestHeaders);
        }
        return (List<Pair<String, String>>) get("zuulResponseHeaders");
    }

    public List<Pair<String, String>> getOriginResponseHeaders() {
        if (get("originResponseHeaders") == null) {
            List<Pair<String, String>> originResponseHeaders = new ArrayList<Pair<String, String>>();
            putIfAbsent("originResponseHeaders", originResponseHeaders);
        }
        return (List<Pair<String, String>>) get("originResponseHeaders");
    }

    public void addOriginResponseHeader(String name, String value) {
        getOriginResponseHeaders().add(new Pair<String, String>(name, value));
    }

    public Long getOriginContentLength() {
        return (Long) get("originContentLength");
    }

    public void setOriginContentLength(Long v) {
        set("originContentLength", v);
    }

    public void setOriginContentLength(String v) {
        try {
            final Long i = Long.valueOf(v);
            set("originContentLength", i);
        } catch (NumberFormatException e) {
            LOG.warn("error parsing origin content length", e);
        }
    }

    public boolean isChunkedRequestBody() {
        final Object v = get("chunkedRequestBody");
        return (v != null) ? (Boolean) v : false;
    }

    public void setChunkedRequestBody() {
        this.set("chunkedRequestBody", Boolean.TRUE);
    }

    public boolean isGzipRequested() {
        final String requestEncoding = this.getRequest().getHeader(ZuulHeaders.ACCEPT_ENCODING);
        return requestEncoding != null && requestEncoding.toLowerCase().contains("gzip");
    }

    public void unset() {
        threadLocal.remove();
    }

    public RequestContext copy() {
        RequestContext copy = new RequestContext();
        Iterator<String> it = keySet().iterator();
        String key = it.next();
        while (key != null) {
            Object orig = get(key);
            try {
                Object copyValue = DeepCopy.copy(orig);
                if (copyValue != null) {
                    copy.set(key, copyValue);
                } else {
                    copy.set(key, orig);
                }
            } catch (NotSerializableException e) {
                copy.set(key, orig);
            }
            if (it.hasNext()) {
                key = it.next();
            } else {
                key = null;
            }
        }
        return copy;
    }

    public Map<String, List<String>> getRequestQueryParams() {
        return (Map<String, List<String>>) get("requestQueryParams");
    }

    public void setRequestQueryParams(Map<String, List<String>> qp) {
        put("requestQueryParams", qp);
    }
}
```



## FilterUsageNotifier.java

```java
package com.netflix.zuul;

/**
 * 过滤器使用通知器
 */
public interface FilterUsageNotifier {
    public void notify(ZuulFilter filter, ExecutionStatus status);
}

```

### FilterProcessor.BasicFilterUsageNotifier.java

```java
/**
 * 记录过滤器使用次数
 */
public static class BasicFilterUsageNotifier implements FilterUsageNotifier {
        private static final String METRIC_PREFIX = "zuul.filter-";

        @Override
        public void notify(ZuulFilter filter, ExecutionStatus status) {
            DynamicCounter.increment(METRIC_PREFIX + filter.getClass().getSimpleName(), "status", status.name(), "filtertype", filter.filterType());
        }
    }
```

## DynamicCodeCompiler.java

```java
package com.netflix.zuul;
/**
 * 动态代码编译器
 */
public interface DynamicCodeCompiler {
    Class compile(String sCode, String sName) throws Exception;

    Class compile(File file) throws Exception;
}

```

### GroovyCompiler.java

```java
package com.netflix.zuul.groovy;
/**
 * Groovy代码编译器
 */
public class GroovyCompiler implements DynamicCodeCompiler {

    private static final Logger LOG = LoggerFactory.getLogger(GroovyCompiler.class);

    @Override
    public Class compile(String sCode, String sName) {
        GroovyClassLoader loader = getGroovyClassLoader();
        LOG.warn("Compiling filter: " + sName);
        Class groovyClass = loader.parseClass(sCode, sName);
        return groovyClass;
    }

    GroovyClassLoader getGroovyClassLoader() {
        return new GroovyClassLoader();
    }

    @Override
    public Class compile(File file) throws IOException {
        GroovyClassLoader loader = getGroovyClassLoader();
        Class groovyClass = loader.parseClass(file);
        return groovyClass;
    }

}
```

## GroovyFileFilter.java

```java
package com.netflix.zuul.groovy;
/**
 * 文件过滤器，只获取.groovy结尾的文件
 */
public class GroovyFileFilter implements FilenameFilter {

    @Override
    public boolean accept(File dir, String name) {
        return name.endsWith(".groovy");
    }
}
```

## FilterRegistry.java

```java
package com.netflix.zuul.filters;
/**
 * ZuulFilter注册器
 */
public class FilterRegistry {

    private static final FilterRegistry INSTANCE = new FilterRegistry();

    public static final FilterRegistry instance() {
        return INSTANCE;
    }

    private final ConcurrentHashMap<String, ZuulFilter> filters = new ConcurrentHashMap<String, ZuulFilter>();

    private FilterRegistry() {
    }

    public ZuulFilter remove(String key) {
        return this.filters.remove(key);
    }

    public ZuulFilter get(String key) {
        return this.filters.get(key);
    }

    public void put(String key, ZuulFilter filter) {
        this.filters.putIfAbsent(key, filter);
    }

    public int size() {
        return this.filters.size();
    }

    public Collection<ZuulFilter> getAllFilters() {
        return this.filters.values();
    }

}

```
## FilterFactory.class.java

```java
package com.netflix.zuul;

/**
 * zuul过滤器工厂
 */
public interface FilterFactory {
    
    public ZuulFilter newInstance(Class clazz) throws Exception;
}

```

### DefaultFilterFactory.java

```java
package com.netflix.zuul;

/**
 * 默认实现
 */
public class DefaultFilterFactory implements FilterFactory {

    @Override
    public ZuulFilter newInstance(Class clazz) throws InstantiationException, IllegalAccessException {
        return (ZuulFilter) clazz.newInstance();
    }

}
```

## FilterLoader.java

```java
package com.netflix.zuul;
/**
 * ZuulFilter加载器
 */
public class FilterLoader {
    final static FilterLoader INSTANCE = new FilterLoader();

    private static final Logger LOG = LoggerFactory.getLogger(FilterLoader.class);

    private final ConcurrentHashMap<String, Long> filterClassLastModified = new ConcurrentHashMap<String, Long>();
    private final ConcurrentHashMap<String, String> filterClassCode = new ConcurrentHashMap<String, String>();
    private final ConcurrentHashMap<String, String> filterCheck = new ConcurrentHashMap<String, String>();
    private final ConcurrentHashMap<String, List<ZuulFilter>> hashFiltersByType = new ConcurrentHashMap<String, List<ZuulFilter>>();

    private FilterRegistry filterRegistry = FilterRegistry.instance();

    static DynamicCodeCompiler COMPILER;
    
    static FilterFactory FILTER_FACTORY = new DefaultFilterFactory();

    public void setCompiler(DynamicCodeCompiler compiler) {
        COMPILER = compiler;
    }

    public void setFilterRegistry(FilterRegistry r) {
        this.filterRegistry = r;
    }

    public void setFilterFactory(FilterFactory factory) {
        FILTER_FACTORY = factory;
    }

    public static FilterLoader getInstance() {
        return INSTANCE;
    }

    public ZuulFilter getFilter(String sCode, String sName) throws Exception {

        if (filterCheck.get(sName) == null) {
            filterCheck.putIfAbsent(sName, sName);
            if (!sCode.equals(filterClassCode.get(sName))) {
                LOG.info("reloading code " + sName);
                filterRegistry.remove(sName);
            }
        }
        ZuulFilter filter = filterRegistry.get(sName);
        if (filter == null) {
            Class clazz = COMPILER.compile(sCode, sName);
            if (!Modifier.isAbstract(clazz.getModifiers())) {
                filter = (ZuulFilter) FILTER_FACTORY.newInstance(clazz);
            }
        }
        return filter;

    }

    public int filterInstanceMapSize() {
        return filterRegistry.size();
    }

    public boolean putFilter(File file) throws Exception {
        String sName = file.getAbsolutePath() + file.getName();
        if (filterClassLastModified.get(sName) != null && (file.lastModified() != filterClassLastModified.get(sName))) {
            LOG.debug("reloading filter " + sName);
            filterRegistry.remove(sName);
        }
        ZuulFilter filter = filterRegistry.get(sName);
        if (filter == null) {
            Class clazz = COMPILER.compile(file);
            if (!Modifier.isAbstract(clazz.getModifiers())) {
                filter = (ZuulFilter) FILTER_FACTORY.newInstance(clazz);
                List<ZuulFilter> list = hashFiltersByType.get(filter.filterType());
                if (list != null) {
                    hashFiltersByType.remove(filter.filterType()); //rebuild this list
                }
                filterRegistry.put(file.getAbsolutePath() + file.getName(), filter);
                filterClassLastModified.put(sName, file.lastModified());
                return true;
            }
        }

        return false;
    }

    public List<ZuulFilter> getFiltersByType(String filterType) {

        List<ZuulFilter> list = hashFiltersByType.get(filterType);
        if (list != null) return list;

        list = new ArrayList<ZuulFilter>();

        Collection<ZuulFilter> filters = filterRegistry.getAllFilters();
        for (Iterator<ZuulFilter> iterator = filters.iterator(); iterator.hasNext(); ) {
            ZuulFilter filter = iterator.next();
            if (filter.filterType().equals(filterType)) {
                list.add(filter);
            }
        }
        Collections.sort(list); // sort by priority

        hashFiltersByType.putIfAbsent(filterType, list);
        return list;
    }
}
```



## FilterProcessor.java

```java
package com.netflix.zuul;

/**
 * 过滤器执行器
 */
public class FilterProcessor {

    static FilterProcessor INSTANCE = new FilterProcessor();
    protected static final Logger logger = LoggerFactory.getLogger(FilterProcessor.class);

    private FilterUsageNotifier usageNotifier;


    public FilterProcessor() {
        usageNotifier = new BasicFilterUsageNotifier();
    }

    public static FilterProcessor getInstance() {
        return INSTANCE;
    }

    public static void setProcessor(FilterProcessor processor) {
        INSTANCE = processor;
    }

    public void setFilterUsageNotifier(FilterUsageNotifier notifier) {
        this.usageNotifier = notifier;
    }

    public void postRoute() throws ZuulException {
        try {
            runFilters("post");
        } catch (ZuulException e) {
            throw e;
        } catch (Throwable e) {
            throw new ZuulException(e, 500, "UNCAUGHT_EXCEPTION_IN_POST_FILTER_" + e.getClass().getName());
        }
    }

    public void error() {
        try {
            runFilters("error");
        } catch (Throwable e) {
            logger.error(e.getMessage(), e);
        }
    }

    public void route() throws ZuulException {
        try {
            runFilters("route");
        } catch (ZuulException e) {
            throw e;
        } catch (Throwable e) {
            throw new ZuulException(e, 500, "UNCAUGHT_EXCEPTION_IN_ROUTE_FILTER_" + e.getClass().getName());
        }
    }

    public void preRoute() throws ZuulException {
        try {
            runFilters("pre");
        } catch (ZuulException e) {
            throw e;
        } catch (Throwable e) {
            throw new ZuulException(e, 500, "UNCAUGHT_EXCEPTION_IN_PRE_FILTER_" + e.getClass().getName());
        }
    }

    public Object runFilters(String sType) throws Throwable {
        if (RequestContext.getCurrentContext().debugRouting()) {
            Debug.addRoutingDebug("Invoking {" + sType + "} type filters");
        }
        boolean bResult = false;
        List<ZuulFilter> list = FilterLoader.getInstance().getFiltersByType(sType);
        if (list != null) {
            for (int i = 0; i < list.size(); i++) {
                ZuulFilter zuulFilter = list.get(i);
                Object result = processZuulFilter(zuulFilter);
                if (result != null && result instanceof Boolean) {
                    bResult |= ((Boolean) result);
                }
            }
        }
        return bResult;
    }

    public Object processZuulFilter(ZuulFilter filter) throws ZuulException {

        RequestContext ctx = RequestContext.getCurrentContext();
        boolean bDebug = ctx.debugRouting();
        final String metricPrefix = "zuul.filter-";
        long execTime = 0;
        String filterName = "";
        try {
            long ltime = System.currentTimeMillis();
            filterName = filter.getClass().getSimpleName();
            
            RequestContext copy = null;
            Object o = null;
            Throwable t = null;

            if (bDebug) {
                Debug.addRoutingDebug("Filter " + filter.filterType() + " " + filter.filterOrder() + " " + filterName);
                copy = ctx.copy();
            }
            
            ZuulFilterResult result = filter.runFilter();
            ExecutionStatus s = result.getStatus();
            execTime = System.currentTimeMillis() - ltime;

            switch (s) {
                case FAILED:
                    t = result.getException();
                    ctx.addFilterExecutionSummary(filterName, ExecutionStatus.FAILED.name(), execTime);
                    break;
                case SUCCESS:
                    o = result.getResult();
                    ctx.addFilterExecutionSummary(filterName, ExecutionStatus.SUCCESS.name(), execTime);
                    if (bDebug) {
                        Debug.addRoutingDebug("Filter {" + filterName + " TYPE:" + filter.filterType() + " ORDER:" + filter.filterOrder() + "} Execution time = " + execTime + "ms");
                        Debug.compareContextState(filterName, copy);
                    }
                    break;
                default:
                    break;
            }
            
            if (t != null) throw t;

            usageNotifier.notify(filter, s);
            return o;

        } catch (Throwable e) {
            if (bDebug) {
                Debug.addRoutingDebug("Running Filter failed " + filterName + " type:" + filter.filterType() + " order:" + filter.filterOrder() + " " + e.getMessage());
            }
            usageNotifier.notify(filter, ExecutionStatus.FAILED);
            if (e instanceof ZuulException) {
                throw (ZuulException) e;
            } else {
                ZuulException ex = new ZuulException(e, "Filter threw Exception", 500, filter.filterType() + ":" + filterName);
                ctx.addFilterExecutionSummary(filterName, ExecutionStatus.FAILED.name(), execTime);
                throw ex;
            }
        }
    }

}
```

## ZuulRunner.java

```java
package com.netflix.zuul;
/**
 * This class initializes servlet requests and responses into the RequestContext and wraps the FilterProcessor calls
 * to preRoute(), route(),  postRoute(), and error() methods
 */
public class ZuulRunner {

    private boolean bufferRequests;

    public ZuulRunner() {
        this.bufferRequests = true;
    }

    public ZuulRunner(boolean bufferRequests) {
        this.bufferRequests = bufferRequests;
    }

    public void init(HttpServletRequest servletRequest, HttpServletResponse servletResponse) {

        RequestContext ctx = RequestContext.getCurrentContext();
        if (bufferRequests) {
            ctx.setRequest(new HttpServletRequestWrapper(servletRequest));
        } else {
            ctx.setRequest(servletRequest);
        }

        ctx.setResponse(new HttpServletResponseWrapper(servletResponse));
    }

    public void postRoute() throws ZuulException {
        FilterProcessor.getInstance().postRoute();
    }

    public void route() throws ZuulException {
        FilterProcessor.getInstance().route();
    }

    public void preRoute() throws ZuulException {
        FilterProcessor.getInstance().preRoute();
    }

    public void error() {
        FilterProcessor.getInstance().error();
    }
}
```

## ZuulServletFilter.java

```java
package com.netflix.zuul.filters;

/**
 * Zuul Servlet filter
 */
public class ZuulServletFilter implements Filter {

    private ZuulRunner zuulRunner;

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

        String bufferReqsStr = filterConfig.getInitParameter("buffer-requests");
        boolean bufferReqs = bufferReqsStr != null && bufferReqsStr.equals("true") ? true : false;

        zuulRunner = new ZuulRunner(bufferReqs);
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        try {
            init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);
            try {
                preRouting();
            } catch (ZuulException e) {
                error(e);
                postRouting();
                return;
            }
            
            // Only forward onto to the chain if a zuul response is not being sent
            if (!RequestContext.getCurrentContext().sendZuulResponse()) {
                filterChain.doFilter(servletRequest, servletResponse);
                return;
            }
            
            try {
                routing();
            } catch (ZuulException e) {
                error(e);
                postRouting();
                return;
            }
            try {
                postRouting();
            } catch (ZuulException e) {
                error(e);
                return;
            }
        } catch (Throwable e) {
            error(new ZuulException(e, 500, "UNCAUGHT_EXCEPTION_FROM_FILTER_" + e.getClass().getName()));
        } finally {
            RequestContext.getCurrentContext().unset();
        }
    }

    void postRouting() throws ZuulException {
        zuulRunner.postRoute();
    }

    void routing() throws ZuulException {
        zuulRunner.route();
    }

    void preRouting() throws ZuulException {
        zuulRunner.preRoute();
    }

    void init(HttpServletRequest servletRequest, HttpServletResponse servletResponse) {
        zuulRunner.init(servletRequest, servletResponse);
    }

    void error(ZuulException e) {
        RequestContext.getCurrentContext().setThrowable(e);
        zuulRunner.error();
    }

    @Override
    public void destroy() {
    }
}
```

## ZuulServlet.java

```java
package com.netflix.zuul.http;


/**
 * Core Zuul servlet which intializes and orchestrates zuulFilter execution
 */
public class ZuulServlet extends HttpServlet {

    private static final long serialVersionUID = -3374242278843351500L;
    private ZuulRunner zuulRunner;
    @Override
    public void init(ServletConfig config) throws ServletException {
        super.init(config);

        String bufferReqsStr = config.getInitParameter("buffer-requests");
        boolean bufferReqs = bufferReqsStr != null && bufferReqsStr.equals("true") ? true : false;

        zuulRunner = new ZuulRunner(bufferReqs);
    }

    @Override
    public void service(javax.servlet.ServletRequest servletRequest, javax.servlet.ServletResponse servletResponse) throws ServletException, IOException {
        try {
            init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);

            // Marks this request as having passed through the "Zuul engine", as opposed to servlets
            // explicitly bound in web.xml, for which requests will not have the same data attached
            RequestContext context = RequestContext.getCurrentContext();
            context.setZuulEngineRan();

            try {
                preRoute();
            } catch (ZuulException e) {
                error(e);
                postRoute();
                return;
            }
            try {
                route();
            } catch (ZuulException e) {
                error(e);
                postRoute();
                return;
            }
            try {
                postRoute();
            } catch (ZuulException e) {
                error(e);
                return;
            }

        } catch (Throwable e) {
            error(new ZuulException(e, 500, "UNHANDLED_EXCEPTION_" + e.getClass().getName()));
        } finally {
            RequestContext.getCurrentContext().unset();
        }
    }

    void postRoute() throws ZuulException {
        zuulRunner.postRoute();
    }

    void route() throws ZuulException {
        zuulRunner.route();
    }

    void preRoute() throws ZuulException {
        zuulRunner.preRoute();
    }

    void init(HttpServletRequest servletRequest, HttpServletResponse servletResponse) {
        zuulRunner.init(servletRequest, servletResponse);
    }

    void error(ZuulException e) {
        RequestContext.getCurrentContext().setThrowable(e);
        zuulRunner.error();
    

}
```

