# spectator

## Tag.java

```java

package com.netflix.spectator.api;

public interface Tag {
  String key();

  String value();
}

```

### BasicTag.java

```java
package com.netflix.spectator.api;
/**
 * Immutable implementation of Tag.
 */
public final class BasicTag implements Tag {

  static BasicTag convert(Tag t) {
    return (t instanceof BasicTag) ? (BasicTag) t : new BasicTag(t.key(), t.value());
  }

  private final String key;
  private final String value;
  private final int hc;

  public BasicTag(String key, String value) {
    this.key = Preconditions.checkNotNull(key, "key");
    this.value = Preconditions.checkNotNull(value, "value");
    this.hc = 31 * key.hashCode() + value.hashCode();
  }

  @Override
  public String key() {
    return key;
  }

  @Override
  public String value() {
    return value;
  }

  @Override public boolean equals(Object obj) {
    if (this == obj) return true;
    if (obj == null || !(obj instanceof BasicTag)) return false;
    BasicTag other = (BasicTag) obj;
    return key.equals(other.key) && value.equals(other.value);
  }
  
  @Override public int hashCode() {
    return hc;
  }

  @Override
  public String toString() {
    return key + '=' + value;
  }
}

```

### Statistic.java

```java
package com.netflix.spectator.api;

/**
 * The valid set of statistics that can be reported by timers and distribution summaries.
 */
public enum Statistic implements Tag {
  /** A value sampled at a point in time. */
  gauge,

  /** Rate per second for calls to record. */
  count,

  /** The maximum amount recorded. */
  max,

  /** The sum of the amounts recorded. */
  totalAmount,

  /** The sum of the squares of the amounts recorded. */
  totalOfSquares,

  /** The sum of the times recorded. */
  totalTime,

  /** Number of currently active tasks for a long task timer. */
  activeTasks,

  /** Duration of a running task. */
  duration,

  /** Value used to compute a distributed percentile estimate. */
  percentile;

  @Override public String key() {
    return "statistic";
  }

  @Override public String value() {
    return name();
  }
}

```

## ArrayTagSet.java

```java
package com.netflix.spectator.api;

/**
 * An immutable set of tags sorted by the tag key.
 */
final class ArrayTagSet implements Iterable<Tag> {

  private static final Comparator<Tag> TAG_COMPARATOR = (t1, t2) -> t1.key().compareTo(t2.key());

  /** Empty tag set. */
  static final ArrayTagSet EMPTY = new ArrayTagSet(new String[0]);

  /** Create a new tag set. */
  static ArrayTagSet create(String... tags) {
    return EMPTY.addAll(tags);
  }

  /** Create a new tag set. */
  static ArrayTagSet create(Tag... tags) {
    return EMPTY.addAll(tags);
  }

  /** Create a new tag set. */
  static ArrayTagSet create(Iterable<Tag> tags) {
    return (tags instanceof ArrayTagSet) ? (ArrayTagSet) tags : EMPTY.addAll(tags);
  }

  /** Create a new tag set. */
  static ArrayTagSet create(Map<String, String> tags) {
    return EMPTY.addAll(tags);
  }

  private final String[] tags;
  private final int length;

  private int cachedHashCode;

  private ArrayTagSet(String[] tags) {
    this(tags, tags.length);
  }

  private ArrayTagSet(String[] tags, int length) {
    if (tags.length % 2 != 0) {
      throw new IllegalArgumentException("length of tags array must be even");
    }
    if (length > tags.length) {
      throw new IllegalArgumentException("length cannot be larger than tags array");
    }
    this.tags = tags;
    this.length = length;
    this.cachedHashCode = 0;
  }

  @Override public Iterator<Tag> iterator() {
    return new Iterator<Tag>() {
      private int i = 0;

      @Override public boolean hasNext() {
        return i < length;
      }

      @Override public Tag next() {
        if (i >= length) {
          throw new NoSuchElementException("next called after end of iterator");
        }
        final String k = tags[i++];
        final String v = tags[i++];
        return new BasicTag(k, v);
      }
    };
  }

  /** Check if this set is empty. */
  boolean isEmpty() {
    return length == 0;
  }

  /** Add a new tag to the set. */
  ArrayTagSet add(String k, String v) {
    return add(new BasicTag(k, v));
  }

  /** Add a new tag to the set. */
  @SuppressWarnings("PMD.AvoidArrayLoops")
  ArrayTagSet add(Tag tag) {
    if (length == 0) {
      return new ArrayTagSet(new String[] {tag.key(), tag.value()});
    } else {
      String[] newTags = new String[length + 2];
      String k = tag.key();
      int i = 0;
      for (; i < length && tags[i].compareTo(k) < 0; i += 2) {
        newTags[i] = tags[i];
        newTags[i + 1] = tags[i + 1];
      }
      if (i < length && tags[i].equals(k)) {
        // Override
        newTags[i++] = tag.key();
        newTags[i++] = tag.value();
        System.arraycopy(tags, i, newTags, i, length - i);
        i = length;
      } else {
        // Insert
        newTags[i] = tag.key();
        newTags[i + 1] = tag.value();
        System.arraycopy(tags, i, newTags, i + 2, length - i);
        i = newTags.length;
      }
      return new ArrayTagSet(newTags, i);
    }
  }

  /** Add a collection of tags to the set. */
  ArrayTagSet addAll(Iterable<Tag> ts) {
    if (ts instanceof Collection) {
      Collection<Tag> data = (Collection<Tag>) ts;
      if (data.isEmpty()) {
        return this;
      } else {
        Tag[] newTags = new Tag[data.size()];
        int i = 0;
        for (Tag t : data) {
          newTags[i] = BasicTag.convert(t);
          ++i;
        }
        return addAll(newTags);
      }
    } else {
      List<Tag> data = new ArrayList<>();
      for (Tag t : ts) {
        data.add(t);
      }
      return addAll(data);
    }
  }

  /** Add a collection of tags to the set. */
  ArrayTagSet addAll(Map<String, String> ts) {
    if (ts.isEmpty()) {
      return this;
    } else {
      Tag[] newTags = new Tag[ts.size()];
      int i = 0;
      for (Map.Entry<String, String> entry : ts.entrySet()) {
        newTags[i++] = new BasicTag(entry.getKey(), entry.getValue());
      }
      return addAll(newTags);
    }
  }

  /** Add a collection of tags to the set. */
  ArrayTagSet addAll(String[] ts) {
    if (ts.length % 2 != 0) {
      throw new IllegalArgumentException("array length must be even, (length=" + ts.length + ")");
    }

    if (ts.length == 0) {
      return this;
    } else {
      int tsLength = ts.length / 2;
      Tag[] newTags = new Tag[tsLength];
      for (int i = 0; i < tsLength; ++i) {
        final int j = i * 2;
        newTags[i] = new BasicTag(ts[j], ts[j + 1]);
      }
      return addAll(newTags, tsLength);
    }
  }

  /** Add a collection of tags to the set. */
  ArrayTagSet addAll(Tag[] ts) {
    return addAll(ts, ts.length);
  }

  /** Add a collection of tags to the set. */
  ArrayTagSet addAll(Tag[] ts, int tsLength) {
    if (tsLength == 0) {
      return this;
    } else if (length == 0) {
      Arrays.sort(ts, 0, tsLength, TAG_COMPARATOR);
      int len = dedup(ts, 0, ts, 0, tsLength);
      return new ArrayTagSet(toStringArray(ts, len));
    } else {
      String[] newTags = new String[(length + tsLength) * 2];
      Arrays.sort(ts, 0, tsLength, TAG_COMPARATOR);
      int newLength = merge(newTags, tags, length, ts, tsLength);
      return new ArrayTagSet(newTags, newLength);
    }
  }

  private String[] toStringArray(Tag[] ts, int length) {
    String[] strs = new String[length * 2];
    for (int i = 0; i < length; ++i) {
      strs[2 * i] = ts[i].key();
      strs[2 * i + 1] = ts[i].value();
    }
    return strs;
  }

  /**
   * Merge and dedup any entries in {@code ts} that have the same key. The last entry
   * with a given key will get selected.
   */
  private int merge(String[] dst, String[] srcA, int lengthA, Tag[] srcB, int lengthB) {
    int i = 0;
    int ai = 0;
    int bi = 0;

    while (ai < lengthA && bi < lengthB) {
      final String ak = srcA[ai];
      final String av = srcA[ai + 1];
      Tag b = srcB[bi];
      int cmp = ak.compareTo(b.key());
      if (cmp < 0) {
        dst[i++] = ak;
        dst[i++] = av;
        ai += 2;
      } else if (cmp > 0) {
        dst[i++] = b.key();
        dst[i++] = b.value();
        ++bi;
      } else {
        // Newer tags should override, use source B if there are duplicate keys.
        // If source B has duplicates, then use the last value for the given key.
        int j = bi + 1;
        for (; j < lengthB && ak.equals(srcB[j].key()); ++j) {
          b = srcB[j];
        }
        dst[i++] = b.key();
        dst[i++] = b.value();
        bi = j;
        ai += 2; // Ignore
      }
    }

    if (ai < lengthA) {
      System.arraycopy(srcA, ai, dst, i, lengthA - ai);
      i += lengthA - ai;
    } else if (bi < lengthB) {
      i = dedup(srcB, bi, dst, i, lengthB - bi);
    }

    return i;
  }

  /**
   * Dedup any entries in {@code ts} that have the same key. The last entry with a given
   * key will get selected. Input data must already be sorted by the tag key. Returns the
   * length of the overall deduped array.
   */
  private int dedup(Tag[] src, int ss, Tag[] dst, int ds, int len) {
    if (len == 0) {
      return ds;
    } else {
      dst[ds] = src[ss];
      String k = src[ss].key();
      int j = ds;
      final int e = ss + len;
      for (int i = ss + 1; i < e; ++i) {
        if (k.equals(src[i].key())) {
          dst[j] = src[i];
        } else {
          k = src[i].key();
          dst[++j] = src[i];
        }
      }
      return j + 1;
    }
  }

  /**
   * Dedup any entries in {@code ts} that have the same key. The last entry with a given
   * key will get selected. Input data must already be sorted by the tag key. Returns the
   * length of the overall deduped array.
   */
  private int dedup(Tag[] src, int ss, String[] dst, int ds, int len) {
    if (len == 0) {
      return ds;
    } else {
      String k = src[ss].key();
      dst[ds] = k;
      dst[ds + 1] = src[ss].value();
      int j = ds;
      final int e = ss + len;
      for (int i = ss + 1; i < e; ++i) {
        if (k.equals(src[i].key())) {
          dst[j] = src[i].key();
          dst[j + 1] = src[i].value();
        } else {
          j += 2; // Not deduping, skip over previous entry
          k = src[i].key();
          dst[j] = k;
          dst[j + 1] = src[i].value();
        }
      }
      return j + 2;
    }
  }

  @Override public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;

    ArrayTagSet other = (ArrayTagSet) o;
    if (length != other.length) return false;

    for (int i = 0; i < length; ++i) {
      if (!tags[i].equals(other.tags[i])) return false;
    }
    return true;
  }

  @Override public int hashCode() {
    if (cachedHashCode == 0) {
      int result = length;
      for (int i = 0; i < length; ++i) {
        result = 31 * result + tags[i].hashCode();
      }
      cachedHashCode = result;
    }
    return cachedHashCode;
  }

  @Override public String toString() {
    StringBuilder builder = new StringBuilder();
    for (int i = 0; i < length; i += 2) {
      builder.append(':').append(tags[i]).append('=').append(tags[i + 1]);
    }
    return builder.toString();
  }
}

```

## Id.java

```java
package com.netflix.spectator.api;
/**
 * Identifier for a meter or measurement.
 */
public interface Id {
  /** Description of the measurement that is being collected. */
  String name();

  /** Other dimensions that can be used to classify the measurement. */
  Iterable<Tag> tags();

  /** Return a new id with an additional tag value. */
  Id withTag(String k, String v);

  /** Return a new id with an additional tag value. */
  Id withTag(Tag t);

  /**
   * Return a new id with an additional tag value using {@link Boolean#toString(boolean)} to
   * convert the boolean value to a string representation. This is merely a convenience function
   * for:
   *
   * <pre>
   *   id.withTag("key", Boolean.toString(value))
   * </pre>
   */
  default Id withTag(String k, boolean v) {
    return withTag(k, Boolean.toString(v));
  }

  /**
   * Return a new id with additional tag values. This overload is to avoid allocating a
   * parameters array for the more generic varargs method {@link #withTags(String...)}.
   */
  default Id withTags(String k1, String v1) {
    return withTag(k1, v1);
  }

  /**
   * Return a new id with additional tag values. This overload is to avoid allocating a
   * parameters array for the more generic varargs method {@link #withTags(String...)}.
   */
  @SuppressWarnings("PMD.UseObjectForClearerAPI")
  default Id withTags(String k1, String v1, String k2, String v2) {
    final Tag[] ts = new Tag[] {
        new BasicTag(k1, v1),
        new BasicTag(k2, v2)
    };
    return withTags(ts);
  }

  /**
   * Return a new id with additional tag values. This overload is to avoid allocating a
   * parameters array for the more generic varargs method {@link #withTags(String...)}.
   */
  @SuppressWarnings("PMD.UseObjectForClearerAPI")
  default Id withTags(String k1, String v1, String k2, String v2, String k3, String v3) {
    final Tag[] ts = new Tag[] {
        new BasicTag(k1, v1),
        new BasicTag(k2, v2),
        new BasicTag(k3, v3)
    };
    return withTags(ts);
  }

  /** Return a new id with additional tag values. */
  default Id withTags(String... tags) {
    Id tmp = this;
    for (int i = 0; i < tags.length; i += 2) {
      tmp = tmp.withTag(tags[i], tags[i + 1]);
    }
    return tmp;
  }

  /** Return a new id with additional tag values. */
  default Id withTags(Tag... tags) {
    Id tmp = this;
    for (Tag t : tags) {
      tmp = tmp.withTag(t);
    }
    return tmp;
  }

  /** Return a new id with additional tag values. */
  default Id withTags(Iterable<Tag> tags) {
    Id tmp = this;
    for (Tag t : tags) {
      tmp = tmp.withTag(t);
    }
    return tmp;
  }

  /** Return a new id with additional tag values. */
  default Id withTags(Map<String, String> tags) {
    Id tmp = this;
    for (Map.Entry<String, String> entry : tags.entrySet()) {
      tmp = tmp.withTag(entry.getKey(), entry.getValue());
    }
    return tmp;
  }
}

```

### NoopId.java

```java
package com.netflix.spectator.api;
/** Id implementation for the no-op registry. */
final class NoopId implements Id {
  /** Singleton instance. */
  static final Id INSTANCE = new NoopId();

  private NoopId() {
  }

  @Override public String name() {
    return "noop";
  }

  @Override public Iterable<Tag> tags() {
    return Collections.emptyList();
  }

  @Override public Id withTag(String k, String v) {
    return this;
  }

  @Override public Id withTag(Tag tag) {
    return this;
  }

  @Override public Id withTags(Iterable<Tag> tags) {
    return this;
  }

  @Override public Id withTags(Map<String, String> tags) {
    return this;
  }

  @Override public String toString() {
    return name();
  }
}

```

### DefaultId.java

```java
package com.netflix.spectator.api;
/** Id implementation for the default registry. */
final class DefaultId implements Id {

  private final String name;
  private final ArrayTagSet tags;

  /** Create a new instance. */
  public DefaultId(String name) {
    this(name, ArrayTagSet.EMPTY);
  }

  /** Create a new instance. */
  DefaultId(String name, ArrayTagSet tags) {
    this.name = Preconditions.checkNotNull(name, "name");
    this.tags = Preconditions.checkNotNull(tags, "tags");
  }

  @Override public String name() {
    return name;
  }

  @Override public Iterable<Tag> tags() {
    return tags;
  }

  @Override public DefaultId withTag(Tag tag) {
    return new DefaultId(name, tags.add(tag));
  }

  @Override public DefaultId withTag(String key, String value) {
    return new DefaultId(name, tags.add(key, value));
  }

  @Override public DefaultId withTags(String... ts) {
    return new DefaultId(name, tags.addAll(ts));
  }

  @Override public DefaultId withTags(Tag... ts) {
    return new DefaultId(name, tags.addAll(ts));
  }

  @Override public DefaultId withTags(Iterable<Tag> ts) {
    return new DefaultId(name, tags.addAll(ts));
  }

  @Override public DefaultId withTags(Map<String, String> ts) {
    return new DefaultId(name, tags.addAll(ts));
  }

  DefaultId normalize() {
    return rollup(Collections.emptySet(), false);
  }

  DefaultId rollup(Set<String> keys, boolean keep) {
    if (tags.isEmpty()) {
      return this;
    } else {
      Map<String, String> ts = new TreeMap<>();
      for (Tag t : tags) {
        if (keys.contains(t.key()) == keep && !ts.containsKey(t.key())) {
          ts.put(t.key(), t.value());
        }
      }
      return new DefaultId(name, ArrayTagSet.create(ts));
    }
  }

  @Override public boolean equals(Object obj) {
    if (this == obj) return true;
    if (obj == null || !(obj instanceof DefaultId)) return false;
    DefaultId other = (DefaultId) obj;
    return name.equals(other.name) && tags.equals(other.tags);
  }

  @Override public int hashCode() {
    int result = name.hashCode();
    result = 31 * result + tags.hashCode();
    return result;
  }

  @Override public String toString() {
    return name + tags;
  }
}

```

## Measurement.java

```java
package com.netflix.spectator.api;
/**
 * A measurement sampled from a meter.
 */
public final class Measurement {

  private final Id id;
  private final long timestamp;
  private final double value;

  /** Create a new instance. */
  public Measurement(Id id, long timestamp, double value) {
    this.id = id;
    this.timestamp = timestamp;
    this.value = value;
  }

  /** Identifier for the measurement. */
  public Id id() {
    return id;
  }

  /**
   * The timestamp in milliseconds since the epoch for when the measurement was taken.
   */
  public long timestamp() {
    return timestamp;
  }

  /** Value for the measurement. */
  public double value() {
    return value;
  }

  @Override
  public boolean equals(Object obj) {
    if (this == obj) return true;
    if (obj == null || !(obj instanceof Measurement)) return false;
    Measurement other = (Measurement) obj;
    return id.equals(other.id)
      && timestamp == other.timestamp
      && Double.compare(value, other.value) == 0;
  }

  @Override
  public int hashCode() {
    final int prime = 31;
    int hc = prime;
    hc = prime * hc + id.hashCode();
    hc = prime * hc + Long.valueOf(timestamp).hashCode();
    hc = prime * hc + Double.valueOf(value).hashCode();
    return hc;
  }

  @Override
  public String toString() {
    return "Measurement(" + id.toString() + "," + timestamp + "," + value + ")";
  }
}

```

## Meter.java

```java
package com.netflix.spectator.api;
/**
 * A device for collecting a set of measurements. Note, this interface is only intended to be
 * implemented by registry implementations.
 */
public interface Meter {

  Id id();

  Iterable<Measurement> measure();

  boolean hasExpired();
}

```

### Counter.java

```java
package com.netflix.spectator.api;
/**
 * Measures the rate of change based on calls to increment.
 */
public interface Counter extends Meter {
  /** Update the counter by one. */
  default void increment() {
    add(1.0);
  }

  default void increment(long amount) {
    add(amount);
  }

  void add(double amount);

  default long count() {
    return (long) actualCount();
  }

  double actualCount();
}

```

### DistributionSummary.java

```java
package com.netflix.spectator.api;

public interface DistributionSummary extends Meter {

  void record(long amount);

  long count();

  long totalAmount();
}

```

### Gauge.java

```java
package com.netflix.spectator.api;

/**
 * A meter with a single value that can only be sampled at a point in time. A typical example is
 * a queue size.
 */
public interface Gauge extends Meter {

  default void set(double value) {
  }

  double value();
}

```

### LongTaskTimer.java

```java
package com.netflix.spectator.api;
/**
 * Timer intended to track a small number of long running tasks. Example would be something like
 * a batch hadoop job. Though "long running" is a bit subjective the assumption is that anything
 * over a minute is long running.
 */
public interface LongTaskTimer extends Meter {

  long start();

  long stop(long task);

  long duration(long task);

  long duration();

  int activeTasks();
}

```

### Timer.java

```java
package com.netflix.spectator.api;
/**
 * Timer intended to track a large number of short running events. Example would be something like
 * an http request. Though "short running" is a bit subjective the assumption is that it should be
 * under a minute.
 *
 * The precise set of information maintained by the timer depends on the implementation. Most
 * should try to provide a consistent implementation of {@link #count()} and {@link #totalTime()},
 * but some implementations may not. In particular, the implementation from {@link NoopRegistry}
 * will always return 0.
 */
public interface Timer extends Meter {

  void record(long amount, TimeUnit unit);

  default void record(Duration amount) {
    record(amount.toNanos(), TimeUnit.NANOSECONDS);
  }

  <T> T record(Callable<T> f) throws Exception;

  void record(Runnable f);

  long count();

  long totalTime();
}

```



## Clock.java

```java
package com.netflix.spectator.api;
/**
 * A timing source that can be used to access the current wall time as well as a high resolution
 * monotonic time to measuring elapsed times. Most of the time the {@link #SYSTEM} implementation
 * that calls the builtin java methods is probably the right one to use. Other implementations
 * would typically only get used for unit tests or other cases where precise control of the clock
 * is needed.
 */
public interface Clock {

  long wallTime();


  long monotonicTime();

  Clock SYSTEM = new Clock() {
    public long wallTime() {
      return System.currentTimeMillis();
    }

    public long monotonicTime() {
      return System.nanoTime();
    }
  };
}

```

### ManualClock.java

```java
package com.netflix.spectator.api;

/**
 * Clock implementation that allows the user to explicitly control the time. Typically used for
 * unit tests.
 */
public class ManualClock implements Clock {

  private final AtomicLong wall;
  private final AtomicLong monotonic;

  /** Create a new instance. */
  public ManualClock() {
    this(0L, 0L);
  }

  public ManualClock(long wallInit, long monotonicInit) {
    wall = new AtomicLong(wallInit);
    monotonic = new AtomicLong(monotonicInit);
  }

  @Override public long wallTime() {
    return wall.get();
  }

  @Override public long monotonicTime() {
    return monotonic.get();
  }

  public void setWallTime(long t) {
    wall.set(t);
  }

  public void setMonotonicTime(long t) {
    monotonic.set(t);
  }
}

```



## Registry.java

```java
package com.netflix.spectator.api;
/**
 * Registry to manage a set of meters.
 */
public interface Registry extends Iterable<Meter> {

  Clock clock();

  default RegistryConfig config() {
    return Config.defaultConfig();
  }

  Id createId(String name);

  Id createId(String name, Iterable<Tag> tags);

  void register(Meter meter);

  ConcurrentMap<Id, Object> state();

  Counter counter(Id id);

  DistributionSummary distributionSummary(Id id);

  Timer timer(Id id);

  Gauge gauge(Id id);

  Gauge maxGauge(Id id);

  Meter get(Id id);

  Iterator<Meter> iterator();

  @SuppressWarnings("unchecked")
  default <T extends Registry> T underlying(Class<T> c) {
    if (c.isAssignableFrom(getClass())) {
      return (T) this;
    } else if (this instanceof CompositeRegistry) {
      return ((CompositeRegistry) this).find(c);
    } else {
      return null;
    }
  }

  default Id createId(String name, String... tags) {
    return createId(name, Utils.toIterable(tags));
  }

  default Id createId(String name, Map<String, String> tags) {
    return createId(name).withTags(tags);
  }

  default Counter counter(String name) {
    return counter(createId(name));
  }

  default Counter counter(String name, Iterable<Tag> tags) {
    return counter(createId(name, tags));
  }

  default Counter counter(String name, String... tags) {
    return counter(createId(name, Utils.toIterable(tags)));
  }

  default DistributionSummary distributionSummary(String name) {
    return distributionSummary(createId(name));
  }

  default DistributionSummary distributionSummary(String name, Iterable<Tag> tags) {
    return distributionSummary(createId(name, tags));
  }

  default DistributionSummary distributionSummary(String name, String... tags) {
    return distributionSummary(createId(name, Utils.toIterable(tags)));
  }

  default Timer timer(String name) {
    return timer(createId(name));
  }

  default Timer timer(String name, Iterable<Tag> tags) {
    return timer(createId(name, tags));
  }

  default Timer timer(String name, String... tags) {
    return timer(createId(name, Utils.toIterable(tags)));
  }

  default Gauge gauge(String name) {
    return gauge(createId(name));
  }

  default Gauge gauge(String name, Iterable<Tag> tags) {
    return gauge(createId(name, tags));
  }

  default Gauge gauge(String name, String... tags) {
    return gauge(createId(name, Utils.toIterable(tags)));
  }

  default Gauge maxGauge(String name) {
    return maxGauge(createId(name));
  }

  default Gauge maxGauge(String name, Iterable<Tag> tags) {
    return maxGauge(createId(name, tags));
  }

  default Gauge maxGauge(String name, String... tags) {
    return maxGauge(createId(name, Utils.toIterable(tags)));
  }

  @Deprecated
  default LongTaskTimer longTaskTimer(Id id) {
    // Note: this method is only included in the registry for historical reasons to
    // maintain compatibility. Future patterns should just use the registry not be
    // created by the registry.
    return com.netflix.spectator.api.patterns.LongTaskTimer.get(this, id);
  }

  @Deprecated
  default LongTaskTimer longTaskTimer(String name) {
    return longTaskTimer(createId(name));
  }

  @Deprecated
  default LongTaskTimer longTaskTimer(String name, Iterable<Tag> tags) {
    return longTaskTimer(createId(name, tags));
  }

  @Deprecated
  default LongTaskTimer longTaskTimer(String name, String... tags) {
    return longTaskTimer(createId(name, Utils.toIterable(tags)));
  }

  @Deprecated
  default <T extends Number> T gauge(Id id, T number) {
    return PolledMeter.using(this).withId(id).monitorValue(number);
  }

  @Deprecated
  default <T extends Number> T gauge(String name, T number) {
    return gauge(createId(name), number);
  }

  @Deprecated
  default <T extends Number> T gauge(String name, Iterable<Tag> tags, T number) {
    return gauge(createId(name, tags), number);
  }

  @Deprecated
  default <T> T gauge(Id id, T obj, ToDoubleFunction<T> f) {
    return PolledMeter.using(this).withId(id).monitorValue(obj, f);
  }

  @Deprecated
  default <T> T gauge(String name, T obj, ToDoubleFunction<T> f) {
    return gauge(createId(name), obj, f);
  }

  @Deprecated
  default <T extends Collection<?>> T collectionSize(Id id, T collection) {
    return PolledMeter.using(this).withId(id).monitorSize(collection);
  }

  @Deprecated
  default <T extends Collection<?>> T collectionSize(String name, T collection) {
    return collectionSize(createId(name), collection);
  }

  @Deprecated
  default <T extends Map<?, ?>> T mapSize(Id id, T collection) {
    return PolledMeter.using(this).withId(id).monitorSize(collection);
  }

  @Deprecated
  default <T extends Map<?, ?>> T mapSize(String name, T collection) {
    return mapSize(createId(name), collection);
  }

  @Deprecated
  default void methodValue(Id id, Object obj, String method) {
    final Method m = Utils.getGaugeMethod(this, id, obj, method);
    if (m != null) {
      PolledMeter.using(this).withId(id).monitorValue(obj, Functions.invokeMethod(m));
    }
  }

  @Deprecated
  default void methodValue(String name, Object obj, String method) {
    methodValue(createId(name), obj, method);
  }

  /** Returns a stream of all registered meters. */
  default Stream<Meter> stream() {
    return StreamSupport.stream(spliterator(), false);
  }

  default Stream<Counter> counters() {
    return stream().filter(m -> m instanceof Counter).map(m -> (Counter) m);
  }

  default Stream<DistributionSummary> distributionSummaries() {
    return stream().filter(m -> m instanceof DistributionSummary).map(m -> (DistributionSummary) m);
  }

  default Stream<Timer> timers() {
    return stream().filter(m -> m instanceof Timer).map(m -> (Timer) m);
  }

  default Stream<Gauge> gauges() {
    return stream().filter(m -> m instanceof Gauge).map(m -> (Gauge) m);
  }

  default void propagate(String msg, Throwable t) {
    LoggerFactory.getLogger(getClass()).warn(msg, t);
    if (config().propagateWarnings()) {
      if (t instanceof RuntimeException) {
        throw (RuntimeException) t;
      } else {
        throw new RuntimeException(t);
      }
    }
  }

  default void propagate(Throwable t) {
    propagate(t.getMessage(), t);
  }
}

```

# end