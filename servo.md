# servo

## Tag.java

```java
package com.netflix.servo.tag;

/**
 * 与指标关联的键-值对.
 */
public interface Tag {

  String getKey();

  String getValue();

  String tagString();
}

```

### BasicTag.java

```java
package com.netflix.servo.tag;

import com.netflix.servo.util.Preconditions;

/**
 * 不可变tag
 */
public final class BasicTag implements Tag {

  private final String key;
  private final String value;

  public BasicTag(String key, String value) {
    this.key = checkNotEmpty(key, "key");
    this.value = checkNotEmpty(value, "value");
  }

  private static String checkNotEmpty(String v, String name) {
    Preconditions.checkNotNull(v, name);
    Preconditions.checkArgument(!v.isEmpty(), "%s cannot be empty", name);
    return v;
  }

  public String getKey() {
    return key;
  }

  public String getValue() {
    return value;
  }

  @Override
  public boolean equals(Object o) {
    if (this == o) {
      return true;
    } else if (o instanceof Tag) {
      Tag t = (Tag) o;
      return key.equals(t.getKey()) && value.equals(t.getValue());
    } else {
      return false;
    }
  }

  @Override
  public int hashCode() {
    int result = key.hashCode();
    result = 31 * result + value.hashCode();
    return result;
  }

  @Override
  public String toString() {
    return key + "=" + value;
  }

  public static BasicTag parseTag(String tagString) {
    return (BasicTag) Tags.parseTag(tagString);
  }

  @Override
  public String tagString() {
    return toString();
  }
}

```

### DataSourceType.java

```java
package com.netflix.servo.annotations;
/**
 * Indicates the type of value that is annotated to determine how it will be
 * measured.
 */
public enum DataSourceType implements Tag {
  /**
   * GAUGE用于不需要修改就可以采样的数值。度量标准的例子应该是当前温度、打开连接的数量、磁盘使用情况等。
   */
  GAUGE,

  /**
   * COUNTER用于在某些事件发生时递增的数值。COUNTER将被采样并转换为每秒的变化率。COUNTER值应单调递增，即，则该值不应减少。
   */
  COUNTER,

  /**
   * RATE用于表示每秒速率的数值。
   */
  RATE,

  /**
   * NORMALIZED每秒的标准化速率。对于根据步骤边界报告值的计数器。
   */
  NORMALIZED,

  /**
   * 信息性属性用于可能对调试有用的值，但不会作为监视目的的指标收集。这些值在JMX中可用。
   */
  INFORMATIONAL;

  /**
   * 用于数据源类型标记的键名，可通过servlet .datasourcetype配置。关键的系统属性
   */
  public static final String KEY = System.getProperty("servo.datasourcetype.key", "type");

  public String getKey() {
    return KEY;
  }

  public String getValue() {
    return name();
  }

  public String tagString() {
    return getKey() + "=" + getValue();
  }
}

```



## PublishingPolicy.java

```java
package com.netflix.servo.monitor;

/**
 * 发布策略允许我们定制不同观察者的行为。
 */
public interface PublishingPolicy {
}

```

### DefaultPublishingPolicy.java

```java
package com.netflix.servo.monitor;

/**
 * 默认发布策略。当与监视器关联的MonitorConfig使用此策略时，观察者必须遵循默认行为。
 */
public final class DefaultPublishingPolicy implements PublishingPolicy {
  private static final DefaultPublishingPolicy INSTANCE = new DefaultPublishingPolicy();

  private DefaultPublishingPolicy() {
  }

  public static DefaultPublishingPolicy getInstance() {
    return INSTANCE;
  }

  @Override
  public String toString() {
    return "DefaultPublishingPolicy";
  }
}

```

## MonitorConfig.java

```java
package com.netflix.servo.monitor;
/**
 * Configuration settings associated with a monitor. A config consists of a name that is required
 * and an optional set of tags.
 */
public final class MonitorConfig {

  public static class Builder {
    private final String name;
    private SmallTagMap.Builder tagsBuilder = SmallTagMap.builder();
    private PublishingPolicy policy = DefaultPublishingPolicy.getInstance();

    public Builder(MonitorConfig config) {
      this(config.getName());
      withTags(config.getTags());
      withPublishingPolicy(config.getPublishingPolicy());
    }

    public Builder(String name) {
      this.name = name;
    }

    public Builder withTag(String key, String val) {
      tagsBuilder.add(Tags.newTag(key, val));
      return this;
    }

    public Builder withTag(Tag tag) {
      tagsBuilder.add(tag);
      return this;
    }

    public Builder withTags(TagList tagList) {
      if (tagList != null) {
        for (Tag t : tagList) {
          tagsBuilder.add(t);
        }
      }
      return this;
    }

    public Builder withTags(Collection<Tag> tagCollection) {
      tagsBuilder.addAll(tagCollection);
      return this;
    }

    public Builder withTags(SmallTagMap.Builder tagsBuilder) {
      this.tagsBuilder = tagsBuilder;
      return this;
    }

    public Builder withPublishingPolicy(PublishingPolicy policy) {
      this.policy = policy;
      return this;
    }

    public MonitorConfig build() {
      return new MonitorConfig(this);
    }

    public String getName() {
      return name;
    }

    public List<Tag> getTags() {
      return UnmodifiableList.copyOf(tagsBuilder.result());
    }

    public PublishingPolicy getPublishingPolicy() {
      return policy;
    }
  }

  public static Builder builder(String name) {
    return new Builder(name);
  }

  private final String name;
  private final TagList tags;
  private final PublishingPolicy policy;

  private final AtomicInteger cachedHashCode = new AtomicInteger(0);

  private MonitorConfig(Builder builder) {
    this.name = Preconditions.checkNotNull(builder.name, "name");
    this.tags = (builder.tagsBuilder.isEmpty())
        ? BasicTagList.EMPTY
        : new BasicTagList(builder.tagsBuilder.result());
    this.policy = builder.policy;
  }

  public String getName() {
    return name;
  }

  public TagList getTags() {
    return tags;
  }

  public PublishingPolicy getPublishingPolicy() {
    return policy;
  }

  @Override
  public boolean equals(Object obj) {
    if (this == obj) {
      return true;
    }
    if (obj == null || !(obj instanceof MonitorConfig)) {
      return false;
    }
    MonitorConfig m = (MonitorConfig) obj;
    return name.equals(m.getName())
        && tags.equals(m.getTags())
        && policy.equals(m.getPublishingPolicy());
  }

  @Override
  public int hashCode() {
    int hash = cachedHashCode.get();
    if (hash == 0) {
      hash = name.hashCode();
      hash = 31 * hash + tags.hashCode();
      hash = 31 * hash + policy.hashCode();
      cachedHashCode.set(hash);
    }
    return hash;
  }

  @Override
  public String toString() {
    return "MonitorConfig{name=" + name + ", tags=" + tags + ", policy=" + policy + '}';
  }

  private MonitorConfig.Builder copy() {
    return MonitorConfig.builder(name).withTags(tags).withPublishingPolicy(policy);
  }

  public MonitorConfig withAdditionalTag(Tag tag) {
    return copy().withTag(tag).build();
  }

  public MonitorConfig withAdditionalTags(TagList newTags) {
    return copy().withTags(newTags).build();
  }
}

```



## Monitor.java

```java
package com.netflix.servo.monitor;

public interface Monitor<T> {

  T getValue();

  T getValue(int pollerIndex);

  MonitorConfig getConfig();
}

```

### AbstractMonitor.java

```java
package com.netflix.servo.monitor;
/**
 * 基本抽象实现
 */
public abstract class AbstractMonitor<T> implements Monitor<T> {
  protected final MonitorConfig config;

  protected AbstractMonitor(MonitorConfig config) {
    this.config = Preconditions.checkNotNull(config, "config");
  }

  @Override
  public MonitorConfig getConfig() {
    return config;
  }

  @Override
  public T getValue() {
    return getValue(0);
  }
}

```



### Informational.java

```java
package com.netflix.servo.monitor;

/**
 * Monitor with a value type of string.
 */
public interface Informational extends Monitor<String> {
}

```

#### BasicInformational.java

```java
package com.netflix.servo.monitor;
/**
 * A simple informational implementation that maintains a string value.
 */
public final class BasicInformational extends AbstractMonitor<String> implements Informational {
  private final AtomicReference<String> info = new AtomicReference<>();

  public BasicInformational(MonitorConfig config) {
    super(config.withAdditionalTag(DataSourceType.INFORMATIONAL));
  }

  public void setValue(String value) {
    info.set(value);
  }

  @Override
  public String getValue(int pollerIndex) {
    return info.get();
  }

  @Override
  public boolean equals(Object o) {
    if (this == o) {
      return true;
    }
    if (o == null || !(o instanceof BasicInformational)) {
      return false;
    }
    BasicInformational that = (BasicInformational) o;

    String thisInfo = info.get();
    String thatInfo = that.info.get();
    return config.equals(that.config)
        && (thisInfo == null ? thatInfo == null : thisInfo.equals(thatInfo));
  }

  @Override
  public int hashCode() {
    int result = config.hashCode();
    int infoHashcode = info.get() != null ? info.get().hashCode() : 0;
    result = 31 * result + infoHashcode;
    return result;
  }

  @Override
  public String toString() {
    return "BasicInformational{config=" + config + ", info=" + info + '}';
  }
}

```

### NumericMonitor.java

```java
package com.netflix.servo.monitor;
/**
 * A monitor type that has a numeric value.
 */
public interface NumericMonitor<T extends Number> extends Monitor<T> {
}

```

#### Counter.java

```java
package com.netflix.servo.monitor;

/**
 * Monitor type for tracking how often some event is occurring.
 */
public interface Counter extends NumericMonitor<Number> {

  void increment();

  void increment(long amount);
}

```

#### Gauge.java

```java
package com.netflix.servo.monitor;

/**
 * Monitor type that provides the current value, e.g., the percentage of disk space used.
 */
public interface Gauge<T extends Number> extends NumericMonitor<T> {
}

```

#### Timer.java

```java
package com.netflix.servo.monitor;

import java.util.concurrent.TimeUnit;

/**
 * Monitor type for tracking how much time something is taking.
 */
public interface Timer extends NumericMonitor<Long> {

  Stopwatch start();

  TimeUnit getTimeUnit();

  @Deprecated
  void record(long duration);

  void record(long duration, TimeUnit timeUnit);
}

```

### CompositeMonitor.java

```java

package com.netflix.servo.monitor;
/**
 * Used as a mixin for monitors that are composed of a number of sub-monitors.
 */
public interface CompositeMonitor<T> extends Monitor<T> {

  List<Monitor<?>> getMonitors();
}

```

## MonitorRegistry.java

```java
package com.netflix.servo;
/**
 * Registry to keep track of objects with
 * {@link com.netflix.servo.annotations.Monitor} annotations.
 */
public interface MonitorRegistry {

  Collection<Monitor<?>> getRegisteredMonitors();

  void register(Monitor<?> monitor);

  void unregister(Monitor<?> monitor);

  boolean isRegistered(Monitor<?> monitor);
}

```

#### BasicMonitorRegistry.java

```java
package com.netflix.servo;
/**
 * Simple monitor registry backed by a {@link java.util.Set}.
 */
public final class BasicMonitorRegistry implements MonitorRegistry {

  private final Set<Monitor<?>> monitors;

  public BasicMonitorRegistry() {
    monitors = Collections.synchronizedSet(new HashSet<>());
  }

  @Override
  public Collection<Monitor<?>> getRegisteredMonitors() {
    return UnmodifiableList.copyOf(monitors);
  }

  @Override
  public void register(Monitor<?> monitor) {
    Preconditions.checkNotNull(monitor, "monitor");
    try {
      monitors.add(monitor);
    } catch (Exception e) {
      throw new IllegalArgumentException("invalid object", e);
    }
  }

  @Override
  public void unregister(Monitor<?> monitor) {
    Preconditions.checkNotNull(monitor, "monitor");
    try {
      monitors.remove(monitor);
    } catch (Exception e) {
      throw new IllegalArgumentException("invalid object", e);
    }
  }

  @Override
  public boolean isRegistered(Monitor<?> monitor) {
    return monitors.contains(monitor);
  }
}

```

#### DefaultMonitorRegistry.java

```java
package com.netflix.servo;

/**
 * Default registry that delegates all actions to a class specified by the
 * {@code com.netflix.servo.DefaultMonitorRegistry.registryClass} property. The
 * specified registry class must have a constructor with no arguments. If the
 * property is not specified or the class cannot be loaded an instance of
 * {@link com.netflix.servo.jmx.JmxMonitorRegistry} will be used.
 * <p/>
 * If the default {@link com.netflix.servo.jmx.JmxMonitorRegistry} is used, the property
 * {@code com.netflix.servo.DefaultMonitorRegistry.jmxMapperClass} can optionally be
 * specified to control how monitors are mapped to JMX {@link javax.management.ObjectName}.
 * This property specifies the {@link com.netflix.servo.jmx.ObjectNameMapper}
 * implementation class to use. The implementation must have a constructor with
 * no arguments.
 */
public final class DefaultMonitorRegistry implements MonitorRegistry {

  private static final Logger LOG = LoggerFactory.getLogger(DefaultMonitorRegistry.class);
  private static final String CLASS_NAME = DefaultMonitorRegistry.class.getCanonicalName();
  private static final String REGISTRY_CLASS_PROP = CLASS_NAME + ".registryClass";
  private static final String REGISTRY_NAME_PROP = CLASS_NAME + ".registryName";
  private static final String REGISTRY_JMX_NAME_PROP = CLASS_NAME + ".jmxMapperClass";
  private static final MonitorRegistry INSTANCE = new DefaultMonitorRegistry();
  private static final String DEFAULT_REGISTRY_NAME = "com.netflix.servo";

  private final MonitorRegistry registry;

  public static MonitorRegistry getInstance() {
    return INSTANCE;
  }

  DefaultMonitorRegistry() {
    this(loadProps());
  }

  DefaultMonitorRegistry(Properties props) {
    final String className = props.getProperty(REGISTRY_CLASS_PROP);
    final String registryName = props.getProperty(REGISTRY_NAME_PROP, DEFAULT_REGISTRY_NAME);
    if (className != null) {
      MonitorRegistry r;
      try {
        Class<?> c = Class.forName(className);
        r = (MonitorRegistry) c.newInstance();
      } catch (Throwable t) {
        LOG.error(
            "failed to create instance of class " + className + ", "
                + "using default class "
                + JmxMonitorRegistry.class.getName(),
            t);
        r = new JmxMonitorRegistry(registryName);
      }
      registry = r;
    } else {
      registry = new JmxMonitorRegistry(registryName,
          getObjectNameMapper(props));
    }
  }

  private static ObjectNameMapper getObjectNameMapper(Properties props) {
    ObjectNameMapper mapper = ObjectNameMapper.DEFAULT;
    final String jmxNameMapperClass = props.getProperty(REGISTRY_JMX_NAME_PROP);
    if (jmxNameMapperClass != null) {
      try {
        Class<?> mapperClazz = Class.forName(jmxNameMapperClass);
        mapper = (ObjectNameMapper) mapperClazz.newInstance();
      } catch (Throwable t) {
        LOG.error(
            "failed to create the JMX ObjectNameMapper instance of class "
                + jmxNameMapperClass
                + ", using the default naming scheme",
            t);
      }
    }

    return mapper;
  }

  private static Properties loadProps() {
    String registryClassProp = System.getProperty(REGISTRY_CLASS_PROP);
    String registryNameProp = System.getProperty(REGISTRY_NAME_PROP);
    String registryJmxNameProp = System.getProperty(REGISTRY_JMX_NAME_PROP);

    Properties props = new Properties();
    if (registryClassProp != null) {
      props.setProperty(REGISTRY_CLASS_PROP, registryClassProp);
    }
    if (registryNameProp != null) {
      props.setProperty(REGISTRY_NAME_PROP, registryNameProp);
    }
    if (registryJmxNameProp != null) {
      props.setProperty(REGISTRY_JMX_NAME_PROP, registryJmxNameProp);
    }
    return props;
  }

  @Override
  public Collection<Monitor<?>> getRegisteredMonitors() {
    return registry.getRegisteredMonitors();
  }

  @Override
  public void register(Monitor<?> monitor) {
    SpectatorContext.register(monitor);
    registry.register(monitor);
  }

  @Override
  public void unregister(Monitor<?> monitor) {
    SpectatorContext.unregister(monitor);
    registry.unregister(monitor);
  }

  MonitorRegistry getInnerRegistry() {
    return registry;
  }

  @Override
  public boolean isRegistered(Monitor<?> monitor) {
    return registry.isRegistered(monitor);
  }
}

```

#### JmxMonitorRegistry.java

```java
package com.netflix.servo.jmx;
/**
 * Monitor registry backed by JMX. The monitor annotations on registered
 * objects will be used to export the data to JMX. For details about the
 * representation in JMX see {@link MonitorMBean}. The {@link ObjectName}
 * used for each monitor depends on the implementation of {@link ObjectNameMapper}
 * that is specified for the registry. See {@link ObjectNameMapper#DEFAULT} for
 * the default naming implementation.
 */
public final class JmxMonitorRegistry implements MonitorRegistry {

  private static final Logger LOG = LoggerFactory.getLogger(JmxMonitorRegistry.class);

  private final MBeanServer mBeanServer;
  private final ConcurrentMap<MonitorConfig, Monitor<?>> monitors;
  private final String name;
  private final ObjectNameMapper mapper;
  private final ConcurrentMap<ObjectName, Object> locks = new ConcurrentHashMap<>();

  private final AtomicBoolean updatePending = new AtomicBoolean(false);
  private final AtomicReference<Collection<Monitor<?>>> monitorList =
      new AtomicReference<>(UnmodifiableList.<Monitor<?>>of());

  /**
   * Creates a new instance that registers metrics with the local mbean
   * server using the default ObjectNameMapper {@link ObjectNameMapper#DEFAULT}.
   *
   * @param name the registry name
   */
  public JmxMonitorRegistry(String name) {
    this(name, ObjectNameMapper.DEFAULT);
  }

  /**
   * Creates a new instance that registers metrics with the local mbean
   * server using the ObjectNameMapper provided.
   *
   * @param name   the registry name
   * @param mapper the monitor to object name mapper
   */
  public JmxMonitorRegistry(String name, ObjectNameMapper mapper) {
    this.name = name;
    this.mapper = mapper;
    mBeanServer = ManagementFactory.getPlatformMBeanServer();
    monitors = new ConcurrentHashMap<>();
  }

  private void register(ObjectName objectName, DynamicMBean mbean) throws Exception {
    synchronized (getLock(objectName)) {
      if (mBeanServer.isRegistered(objectName)) {
        mBeanServer.unregisterMBean(objectName);
      }
      mBeanServer.registerMBean(mbean, objectName);
    }
  }

  /**
   * The set of registered Monitor objects.
   */
  @Override
  public Collection<Monitor<?>> getRegisteredMonitors() {
    if (updatePending.getAndSet(false)) {
      monitorList.set(UnmodifiableList.copyOf(monitors.values()));
    }
    return monitorList.get();
  }

  /**
   * Register a new monitor in the registry.
   */
  @Override
  public void register(Monitor<?> monitor) {
    try {
      List<MonitorMBean> beans = MonitorMBean.createMBeans(name, monitor, mapper);
      for (MonitorMBean bean : beans) {
        register(bean.getObjectName(), bean);
      }
      monitors.put(monitor.getConfig(), monitor);
      updatePending.set(true);
    } catch (Exception e) {
      LOG.warn("Unable to register Monitor:" + monitor.getConfig(), e);
    }
  }

  /**
   * Unregister a Monitor from the registry.
   */
  @Override
  public void unregister(Monitor<?> monitor) {
    try {
      List<MonitorMBean> beans = MonitorMBean.createMBeans(name, monitor, mapper);
      for (MonitorMBean bean : beans) {
        try {
          mBeanServer.unregisterMBean(bean.getObjectName());
          locks.remove(bean.getObjectName());
        } catch (InstanceNotFoundException ignored) {
          // ignore errors attempting to unregister a non-registered monitor
          // a common error is to unregister twice
        }
      }
      monitors.remove(monitor.getConfig());
      updatePending.set(true);
    } catch (Exception e) {
      LOG.warn("Unable to un-register Monitor:" + monitor.getConfig(), e);
    }
  }

  @Override
  public boolean isRegistered(Monitor<?> monitor) {
    try {
      List<MonitorMBean> beans = MonitorMBean.createMBeans(name, monitor, mapper);
      for (MonitorMBean bean : beans) {
        if (mBeanServer.isRegistered(bean.getObjectName())) {
          return true;
        }
      }
    } catch (Exception e) {
      return false;
    }
    return false;
  }

  private Object getLock(ObjectName objectName) {
    return locks.computeIfAbsent(objectName, k -> new Object());
  }
}

```



# end