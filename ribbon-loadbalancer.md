# LoadBalancer

## Server.java

```java
package com.netflix.loadbalancer;

import com.netflix.util.Pair;

/**
 * Class that represents a typical Server (or an addressable Node) i.e. a
 * Host:port identifier
 */
public class Server {

    /**
     * 服务器的其他元信息，其中包含目标应用程序的信息，以及特定于部署环境(例如AWS)的服务器标识。
     */
    public static interface MetaInfo {

        public String getAppName();

        public String getServerGroup();

        public String getServiceIdForDiscovery();

        public String getInstanceId();
    }

    public static final String UNKNOWN_ZONE = "UNKNOWN";
    private String host;
    private int port = 80;
    private volatile String id;
    private boolean isAliveFlag;
    private String zone = UNKNOWN_ZONE;
    private volatile boolean readyToServe = true;

    private MetaInfo simpleMetaInfo = new MetaInfo() {
        @Override
        public String getAppName() {
            return null;
        }

        @Override
        public String getServerGroup() {
            return null;
        }

        @Override
        public String getServiceIdForDiscovery() {
            return null;
        }

        @Override
        public String getInstanceId() {
            return id;
        }
    };

    public Server(String host, int port) {
        this.host = host;
        this.port = port;
        this.id = host + ":" + port;
        isAliveFlag = false;
    }

    /* host:port combination */
    public Server(String id) {
        setId(id);
        isAliveFlag = false;
    }

    // No reason to synchronize this, I believe.
    // The assignment should be atomic, and two setAlive calls
    // with conflicting results will still give nonsense(last one wins)
    // synchronization or no.

    public void setAlive(boolean isAliveFlag) {
        this.isAliveFlag = isAliveFlag;
    }

    public boolean isAlive() {
        return isAliveFlag;
    }

    @Deprecated
    public void setHostPort(String hostPort) {
        setId(hostPort);
    }

    static public String normalizeId(String id) {
        Pair<String, Integer> hostPort = getHostPort(id);
        if (hostPort == null) {
            return null;
        } else {
            return hostPort.first() + ":" + hostPort.second();
        }
    }

    static Pair<String, Integer> getHostPort(String id) {
        if (id != null) {
            String host = null;
            int port = 80;

            if (id.toLowerCase().startsWith("http://")) {
                id = id.substring(7);
            } else if (id.toLowerCase().startsWith("https://")) {
                id = id.substring(8);
            }

            if (id.contains("/")) {
                int slash_idx = id.indexOf("/");
                id = id.substring(0, slash_idx);
            }

            int colon_idx = id.indexOf(':');

            if (colon_idx == -1) {
                host = id; // default
                port = 80;
            } else {
                host = id.substring(0, colon_idx);
                try {
                    port = Integer.parseInt(id.substring(colon_idx + 1));
                } catch (NumberFormatException e) {
                    throw e;
                }
            }
            return new Pair<String, Integer>(host, port);
        } else {
            return null;
        }

    }

    public void setId(String id) {
        Pair<String, Integer> hostPort = getHostPort(id);
        if (hostPort != null) {
            this.id = hostPort.first() + ":" + hostPort.second();
            this.host = hostPort.first();
            this.port = hostPort.second();
        } else {
            this.id = null;
        }
    }

    public void setPort(int port) {
        this.port = port;

        if (host != null) {
            id = host + ":" + port;
        }
    }

    public void setHost(String host) {
        if (host != null) {
            this.host = host;
            id = host + ":" + port;
        }
    }

    public String getId() {
        return id;
    }

    public String getHost() {
        return host;
    }

    public int getPort() {
        return port;
    }

    public String getHostPort() {
        return host + ":" + port;
    }

    public MetaInfo getMetaInfo() {
        return simpleMetaInfo;
    }

    public String toString() {
        return this.getId();
    }

    public boolean equals(Object obj) {
        if (this == obj)
            return true;
        if (!(obj instanceof Server))
            return false;
        Server svc = (Server) obj;
        return svc.getId().equals(this.getId());

    }

    public int hashCode() {
        int hash = 7;
        hash = 31 * hash + (null == this.getId() ? 0 : this.getId().hashCode());
        return hash;
    }

    public final String getZone() {
        return zone;
    }

    public final void setZone(String zone) {
        this.zone = zone;
    }

    public final boolean isReadyToServe() {
        return readyToServe;
    }

    public final void setReadyToServe(boolean readyToServe) {
        this.readyToServe = readyToServe;
    }
}

```

## ServerListFilter.java

```java
package com.netflix.loadbalancer;

import java.util.List;

/**
 * 该接口允许过滤配置的或动态获得的具有所需特征的候选服务器列表。
 */
public interface ServerListFilter<T extends Server> {

    public List<T> getFilteredListOfServers(List<T> servers);

}

```

### AbstractServerListFilter.java

```java
package com.netflix.loadbalancer;


/**
 * Class that is responsible to Filter out list of servers from the ones 
 * currently available in the Load Balancer
 */
public abstract class AbstractServerListFilter<T extends Server> implements ServerListFilter<T> {

    private volatile LoadBalancerStats stats;
    
    public void setLoadBalancerStats(LoadBalancerStats stats) {
        this.stats = stats;
    }
    
    public LoadBalancerStats getLoadBalancerStats() {
        return stats;
    }

}

```

#### ZoneAffinityServerListFilter.java

```java
package com.netflix.loadbalancer;
/**
 * 区域亲和的服务器列表过滤器
 */
public class ZoneAffinityServerListFilter<T extends Server> extends
        AbstractServerListFilter<T> implements IClientConfigAware {

    private volatile boolean zoneAffinity = DefaultClientConfigImpl.DEFAULT_ENABLE_ZONE_AFFINITY;
    private volatile boolean zoneExclusive = DefaultClientConfigImpl.DEFAULT_ENABLE_ZONE_EXCLUSIVITY;
    private DynamicDoubleProperty activeReqeustsPerServerThreshold;
    private DynamicDoubleProperty blackOutServerPercentageThreshold;
    private DynamicIntProperty availableServersThreshold;
    private Counter overrideCounter;
    private ZoneAffinityPredicate zoneAffinityPredicate = new ZoneAffinityPredicate();
    
    private static Logger logger = LoggerFactory.getLogger(ZoneAffinityServerListFilter.class);
    
    String zone;
        
    public ZoneAffinityServerListFilter() {      
    }
    
    public ZoneAffinityServerListFilter(IClientConfig niwsClientConfig) {
        initWithNiwsConfig(niwsClientConfig);
    }
    
    @Override
    public void initWithNiwsConfig(IClientConfig niwsClientConfig) {
        String sZoneAffinity = "" + niwsClientConfig.getProperty(CommonClientConfigKey.EnableZoneAffinity, false);
        if (sZoneAffinity != null){
            zoneAffinity = Boolean.parseBoolean(sZoneAffinity);
            logger.debug("ZoneAffinity is set to {}", zoneAffinity);
        }
        String sZoneExclusive = "" + niwsClientConfig.getProperty(CommonClientConfigKey.EnableZoneExclusivity, false);
        if (sZoneExclusive != null){
            zoneExclusive = Boolean.parseBoolean(sZoneExclusive);
        }
        if (ConfigurationManager.getDeploymentContext() != null) {
            zone = ConfigurationManager.getDeploymentContext().getValue(ContextKey.zone);
        }
        activeReqeustsPerServerThreshold = DynamicPropertyFactory.getInstance().getDoubleProperty(niwsClientConfig.getClientName() + "." + niwsClientConfig.getNameSpace() + ".zoneAffinity.maxLoadPerServer", 0.6d);
        logger.debug("activeReqeustsPerServerThreshold: {}", activeReqeustsPerServerThreshold.get());
        blackOutServerPercentageThreshold = DynamicPropertyFactory.getInstance().getDoubleProperty(niwsClientConfig.getClientName() + "." + niwsClientConfig.getNameSpace() + ".zoneAffinity.maxBlackOutServesrPercentage", 0.8d);
        logger.debug("blackOutServerPercentageThreshold: {}", blackOutServerPercentageThreshold.get());
        availableServersThreshold = DynamicPropertyFactory.getInstance().getIntProperty(niwsClientConfig.getClientName() + "." + niwsClientConfig.getNameSpace() + ".zoneAffinity.minAvailableServers", 2);
        logger.debug("availableServersThreshold: {}", availableServersThreshold.get());
        overrideCounter = Monitors.newCounter("ZoneAffinity_OverrideCounter");

        Monitors.registerObject("NIWSServerListFilter_" + niwsClientConfig.getClientName());
    }
    
    private boolean shouldEnableZoneAffinity(List<T> filtered) {    
        if (!zoneAffinity && !zoneExclusive) {
            return false;
        }
        if (zoneExclusive) {
            return true;
        }
        LoadBalancerStats stats = getLoadBalancerStats();
        if (stats == null) {
            return zoneAffinity;
        } else {
            logger.debug("Determining if zone affinity should be enabled with given server list: {}", filtered);
            ZoneSnapshot snapshot = stats.getZoneSnapshot(filtered);
            double loadPerServer = snapshot.getLoadPerServer();
            int instanceCount = snapshot.getInstanceCount();            
            int circuitBreakerTrippedCount = snapshot.getCircuitTrippedCount();
            if (((double) circuitBreakerTrippedCount) / instanceCount >= blackOutServerPercentageThreshold.get() 
                    || loadPerServer >= activeReqeustsPerServerThreshold.get()
                    || (instanceCount - circuitBreakerTrippedCount) < availableServersThreshold.get()) {
                logger.debug("zoneAffinity is overriden. blackOutServerPercentage: {}, activeReqeustsPerServer: {}, availableServers: {}", 
                        new Object[] {(double) circuitBreakerTrippedCount / instanceCount,  loadPerServer, instanceCount - circuitBreakerTrippedCount});
                return false;
            } else {
                return true;
            }
            
        }
    }
        
    @Override
    public List<T> getFilteredListOfServers(List<T> servers) {
        if (zone != null && (zoneAffinity || zoneExclusive) && servers !=null && servers.size() > 0){
            List<T> filteredServers = Lists.newArrayList(Iterables.filter(
                    servers, this.zoneAffinityPredicate.getServerOnlyPredicate()));
            if (shouldEnableZoneAffinity(filteredServers)) {
                return filteredServers;
            } else if (zoneAffinity) {
                overrideCounter.increment();
            }
        }
        return servers;
    }

    @Override
    public String toString(){
        StringBuilder sb = new StringBuilder("ZoneAffinityServerListFilter:");
        sb.append(", zone: ").append(zone).append(", zoneAffinity:").append(zoneAffinity);
        sb.append(", zoneExclusivity:").append(zoneExclusive);
        return sb.toString();       
    }
}

```

##### ServerListSubsetFilter.java

```java
package com.netflix.loadbalancer;
/**
 * 服务器列表筛选器，它将负载均衡器使用的服务器数量限制为所有服务器的子集。如果服务器群很大(例如数百个)，并且不需要使用其中的每一个，也不需要在http客户  * 机的连接池中保持连接，那么这种方法就非常有用。它还可以通过比较总网络故障和并发连接来排除相对不健康的服务器。
 */
public class ServerListSubsetFilter<T extends Server> extends ZoneAffinityServerListFilter<T> implements IClientConfigAware, Comparator<T>{

    private Random random = new Random();
    private volatile Set<T> currentSubset = Sets.newHashSet(); 
    private DynamicIntProperty sizeProp = new DynamicIntProperty(DefaultClientConfigImpl.DEFAULT_PROPERTY_NAME_SPACE + ".ServerListSubsetFilter.size", 20);
    private DynamicFloatProperty eliminationPercent = 
            new DynamicFloatProperty(DefaultClientConfigImpl.DEFAULT_PROPERTY_NAME_SPACE + ".ServerListSubsetFilter.forceEliminatePercent", 0.1f);
    private DynamicIntProperty eliminationFailureCountThreshold = 
            new DynamicIntProperty(DefaultClientConfigImpl.DEFAULT_PROPERTY_NAME_SPACE + ".ServerListSubsetFilter.eliminationFailureThresold", 0);
    private DynamicIntProperty eliminationConnectionCountThreshold = 
            new DynamicIntProperty(DefaultClientConfigImpl.DEFAULT_PROPERTY_NAME_SPACE + ".ServerListSubsetFilter.eliminationConnectionThresold", 0);
    
    @Override
    public void initWithNiwsConfig(IClientConfig clientConfig) {
        super.initWithNiwsConfig(clientConfig);
        sizeProp = new DynamicIntProperty(clientConfig.getClientName() + "." + clientConfig.getNameSpace() + ".ServerListSubsetFilter.size", 20);
        eliminationPercent = 
                new DynamicFloatProperty(clientConfig.getClientName() + "." + clientConfig.getNameSpace() + ".ServerListSubsetFilter.forceEliminatePercent", 0.1f);
        eliminationFailureCountThreshold = new DynamicIntProperty( clientConfig.getClientName()  + "." + clientConfig.getNameSpace()
                + ".ServerListSubsetFilter.eliminationFailureThresold", 0);
        eliminationConnectionCountThreshold = new DynamicIntProperty(clientConfig.getClientName() + "." + clientConfig.getNameSpace()
                + ".ServerListSubsetFilter.eliminationConnectionThresold", 0);
    }

    @Override
    public List<T> getFilteredListOfServers(List<T> servers) {
        List<T> zoneAffinityFiltered = super.getFilteredListOfServers(servers);
        Set<T> candidates = Sets.newHashSet(zoneAffinityFiltered);
        Set<T> newSubSet = Sets.newHashSet(currentSubset);
        LoadBalancerStats lbStats = getLoadBalancerStats();
        for (T server: currentSubset) {
            // this server is either down or out of service
            if (!candidates.contains(server)) {
                newSubSet.remove(server);
            } else {
                ServerStats stats = lbStats.getSingleServerStat(server);
                // remove the servers that do not meet health criteria
                if (stats.getActiveRequestsCount() > eliminationConnectionCountThreshold.get()
                        || stats.getFailureCount() > eliminationFailureCountThreshold.get()) {
                    newSubSet.remove(server);
                    // also remove from the general pool to avoid selecting them again
                    candidates.remove(server);
                }
            }
        }
        int targetedListSize = sizeProp.get();
        int numEliminated = currentSubset.size() - newSubSet.size();
        int minElimination = (int) (targetedListSize * eliminationPercent.get());
        int numToForceEliminate = 0;
        if (targetedListSize < newSubSet.size()) {
            // size is shrinking
            numToForceEliminate = newSubSet.size() - targetedListSize;
        } else if (minElimination > numEliminated) {
            numToForceEliminate = minElimination - numEliminated; 
        }
        
        if (numToForceEliminate > newSubSet.size()) {
            numToForceEliminate = newSubSet.size();
        }

        if (numToForceEliminate > 0) {
            List<T> sortedSubSet = Lists.newArrayList(newSubSet);           
            Collections.sort(sortedSubSet, this);
            List<T> forceEliminated = sortedSubSet.subList(0, numToForceEliminate);
            newSubSet.removeAll(forceEliminated);
            candidates.removeAll(forceEliminated);
        }
        
        // after forced elimination or elimination of unhealthy instances,
        // the size of the set may be less than the targeted size,
        // then we just randomly add servers from the big pool
        if (newSubSet.size() < targetedListSize) {
            int numToChoose = targetedListSize - newSubSet.size();
            candidates.removeAll(newSubSet);
            if (numToChoose > candidates.size()) {
                // Not enough healthy instances to choose, fallback to use the
                // total server pool
                candidates = Sets.newHashSet(zoneAffinityFiltered);
                candidates.removeAll(newSubSet);
            }
            List<T> chosen = randomChoose(Lists.newArrayList(candidates), numToChoose);
            for (T server: chosen) {
                newSubSet.add(server);
            }
        }
        currentSubset = newSubSet;       
        return Lists.newArrayList(newSubSet);            
    }

    private List<T> randomChoose(List<T> servers, int toChoose) {
        int size = servers.size();
        if (toChoose >= size || toChoose < 0) {
            return servers;
        } 
        for (int i = 0; i < toChoose; i++) {
            int index = random.nextInt(size);
            T tmp = servers.get(index);
            servers.set(index, servers.get(i));
            servers.set(i, tmp);
        }
        return servers.subList(0, toChoose);        
    }

    @Override
    public int compare(T server1, T server2) {
        LoadBalancerStats lbStats = getLoadBalancerStats();
        ServerStats stats1 = lbStats.getSingleServerStat(server1);
        ServerStats stats2 = lbStats.getSingleServerStat(server2);
        int failuresDiff = (int) (stats2.getFailureCount() - stats1.getFailureCount());
        if (failuresDiff != 0) {
            return failuresDiff;
        } else {
            return (stats2.getActiveRequestsCount() - stats1.getActiveRequestsCount());
        }
    }
}

```



## ServerList.java

```java
package com.netflix.loadbalancer;

import java.util.List;

/**
 * Interface that defines the methods sed to obtain the List of Servers
 */
public interface ServerList<T extends Server> {

    public List<T> getInitialListOfServers();

    public List<T> getUpdatedListOfServers();   

}

```

### AbstractServerList.java

```java
package com.netflix.loadbalancer;
/**
 * 该类包含一个API，用于创建一个过滤器，负载均衡器将使用该过滤器来过滤从getUpdatedListOfServers()或getInitialListOfServers()返回的服务器。
 */
public abstract class AbstractServerList<T extends Server> implements ServerList<T>, IClientConfigAware {   

    public AbstractServerListFilter<T> getFilterImpl(IClientConfig niwsClientConfig) throws ClientException{
        try {
            String niwsServerListFilterClassName = niwsClientConfig
                    .getProperty(
                            CommonClientConfigKey.NIWSServerListFilterClassName,
                            ZoneAffinityServerListFilter.class.getName())
                    .toString();

            AbstractServerListFilter<T> abstractNIWSServerListFilter = 
                    (AbstractServerListFilter<T>) ClientFactory.instantiateInstanceWithClientConfig(niwsServerListFilterClassName, niwsClientConfig);
            return abstractNIWSServerListFilter;
        } catch (Throwable e) {
            throw new ClientException(
                    ClientException.ErrorType.CONFIGURATION,
                    "Unable to get an instance of CommonClientConfigKey.NIWSServerListFilterClassName. Configured class:"
                            + niwsClientConfig
                                    .getProperty(CommonClientConfigKey.NIWSServerListFilterClassName), e);
        }
    }
}

```

#### ConfigurationBasedServerList.java

```java
package com.netflix.loadbalancer;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import com.google.common.base.Strings;
import com.google.common.collect.Lists;
import com.netflix.client.config.CommonClientConfigKey;
import com.netflix.client.config.DefaultClientConfigImpl;
import com.netflix.client.config.IClientConfig;
import com.netflix.config.DynamicPropertyFactory;
import com.netflix.config.DynamicStringProperty;

/**
 * 基于配置的服务列表，配置格式<clientName>.<nameSpace>.listOfServers=<comma delimited hostname:port strings>
 */
public class ConfigurationBasedServerList extends AbstractServerList<Server>  {

	private IClientConfig clientConfig;
		
	@Override
	public List<Server> getInitialListOfServers() {
	    return getUpdatedListOfServers();
	}

	@Override
	public List<Server> getUpdatedListOfServers() {
        String listOfServers = clientConfig.get(CommonClientConfigKey.ListOfServers);
        return derive(listOfServers);
	}

	@Override
	public void initWithNiwsConfig(IClientConfig clientConfig) {
	    this.clientConfig = clientConfig;
	}
	
	private List<Server> derive(String value) {
	    List<Server> list = Lists.newArrayList();
		if (!Strings.isNullOrEmpty(value)) {
			for (String s: value.split(",")) {
				list.add(new Server(s.trim()));
			}
		}
        return list;
	}
}

```

## IRule.java

```java
package com.netflix.loadbalancer;

/**
 * 该接口为负载均衡器定义“规则”。规则可以看作是一种负载平衡策略。众所周知的负载平衡策略包括循环、基于响应时间等。
 */
public interface IRule{
  
    public Server choose(Object key);
    
    public void setLoadBalancer(ILoadBalancer lb);
    
    public ILoadBalancer getLoadBalancer();    
}

```

### AbstractLoadBalancerRule.java

```java
package com.netflix.loadbalancer;

import com.netflix.client.IClientConfigAware;

/**
 * 为设置和获取负载均衡器提供默认实现的类
 */
public abstract class AbstractLoadBalancerRule implements IRule, IClientConfigAware {

    private ILoadBalancer lb;
        
    @Override
    public void setLoadBalancer(ILoadBalancer lb){
        this.lb = lb;
    }
    
    @Override
    public ILoadBalancer getLoadBalancer(){
        return lb;
    }      
}

```

#### RandomRule.java

```java
package com.netflix.loadbalancer;
/**
 * 随机规则
 */
public class RandomRule extends AbstractLoadBalancerRule {
    Random rand;

    public RandomRule() {
        rand = new Random();
    }

    @edu.umd.cs.findbugs.annotations.SuppressWarnings(value = "RCN_REDUNDANT_NULLCHECK_OF_NULL_VALUE")
    public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            return null;
        }
        Server server = null;

        while (server == null) {
            if (Thread.interrupted()) {
                return null;
            }
            List<Server> upList = lb.getReachableServers();
            List<Server> allList = lb.getAllServers();

            int serverCount = allList.size();
            if (serverCount == 0) {
                return null;
            }

            int index = rand.nextInt(serverCount);
            server = upList.get(index);

            if (server == null) {
                Thread.yield();
                continue;
            }

            if (server.isAlive()) {
                return (server);
            }

            // Shouldn't actually happen.. but must be transient or a bug.
            server = null;
            Thread.yield();
        }

        return server;

    }

	@Override
	public Server choose(Object key) {
		return choose(getLoadBalancer(), key);
	}

	@Override
	public void initWithNiwsConfig(IClientConfig clientConfig) {
		// TODO Auto-generated method stub
		
	}
}

```

#### RoundRobinRule.java

```java
package com.netflix.loadbalancer;
/**
 * 轮询规则
 *
 */
public class RoundRobinRule extends AbstractLoadBalancerRule {

    private AtomicInteger nextServerCyclicCounter;
    private static final boolean AVAILABLE_ONLY_SERVERS = true;
    private static final boolean ALL_SERVERS = false;

    private static Logger log = LoggerFactory.getLogger(RoundRobinRule.class);

    public RoundRobinRule() {
        nextServerCyclicCounter = new AtomicInteger(0);
    }

    public RoundRobinRule(ILoadBalancer lb) {
        this();
        setLoadBalancer(lb);
    }

    public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            log.warn("no load balancer");
            return null;
        }

        Server server = null;
        int count = 0;
        while (server == null && count++ < 10) {
            List<Server> reachableServers = lb.getReachableServers();
            List<Server> allServers = lb.getAllServers();
            int upCount = reachableServers.size();
            int serverCount = allServers.size();

            if ((upCount == 0) || (serverCount == 0)) {
                log.warn("No up servers available from load balancer: " + lb);
                return null;
            }

            int nextServerIndex = incrementAndGetModulo(serverCount);
            server = allServers.get(nextServerIndex);

            if (server == null) {
                /* Transient. */
                Thread.yield();
                continue;
            }

            if (server.isAlive() && (server.isReadyToServe())) {
                return (server);
            }

            // Next.
            server = null;
        }

        if (count >= 10) {
            log.warn("No available alive servers after 10 tries from load balancer: "
                    + lb);
        }
        return server;
    }

    private int incrementAndGetModulo(int modulo) {
        for (;;) {
            int current = nextServerCyclicCounter.get();
            int next = (current + 1) % modulo;
            if (nextServerCyclicCounter.compareAndSet(current, next))
                return next;
        }
    }

    @Override
    public Server choose(Object key) {
        return choose(getLoadBalancer(), key);
    }

    @Override
    public void initWithNiwsConfig(IClientConfig clientConfig) {
    }
}

```

## IPing.java

```java
package com.netflix.loadbalancer;

/**
 * ping服务器策略接口
 */
public interface IPing {

    public boolean isAlive(Server server);
}

```

### NoOpPing.java

```java
package com.netflix.loadbalancer;

/**
 * No Op Ping
 */
public class NoOpPing implements IPing {

    @Override
    public boolean isAlive(Server server) {
        return true;
    }

}

```

### PingConstant.java

```java
package com.netflix.loadbalancer;

public class PingConstant implements IPing {
		boolean constant = true;

		public void setConstant(String constantStr) {
				constant = (constantStr != null) && (constantStr.toLowerCase().equals("true"));
		}

		public void setConstant(boolean constant) {
				this.constant = constant;
		}

		public boolean getConstant() {
				return constant;
		}

		public boolean isAlive(Server server) {
				return constant;
		}
}

```

### AbstractLoadBalancerPing.java

```java
package com.netflix.loadbalancer;
/**
 * Class that provides the basic implementation of detmerining the "liveness" or
 * suitability of a Server (a node)
 */
public abstract class AbstractLoadBalancerPing implements IPing, IClientConfigAware{

    AbstractLoadBalancer lb;
    
    @Override
    public boolean isAlive(Server server) {
        return true;
    }
    
    public void setLoadBalancer(AbstractLoadBalancer lb){
        this.lb = lb;
    }
    
    public AbstractLoadBalancer getLoadBalancer(){
        return lb;
    }

}

```

#### DummyPing.java

```java
package com.netflix.loadbalancer;
/**
 * Default simple implementation that marks the liveness of a Server
 */
public class DummyPing extends AbstractLoadBalancerPing {

    public DummyPing() {
    }

    public boolean isAlive(Server server) {
        return true;
    }

    @Override
    public void initWithNiwsConfig(IClientConfig clientConfig) {
    }
}

```

## IPingStrategy.java

```java
package com.netflix.loadbalancer;

/**
 * 定义用于ping在com.netflix.loadbalancer.BaseLoadBalancer中注册的所有服务器的策略。如果希望并行地ping服务器，通常需要创建此接口的自定义实现。请注意，这个接口的实现应该是不可变的。
 */
public interface IPingStrategy {

    boolean[] pingServers(IPing ping, Server[] servers);
}

```

### BaseLoadBalancer#SerialPingStrategy.java

```java
/**
  * IPingStrategy的默认实现是串行执行ping，如果您的IPing实现很慢，或者您有大量的服务器，那么这可能是不可取的。
  */
private static class SerialPingStrategy implements IPingStrategy {

        @Override
        public boolean[] pingServers(IPing ping, Server[] servers) {
            int numCandidates = servers.length;
            boolean[] results = new boolean[numCandidates];

            if (logger.isDebugEnabled()) {
                logger.debug("LoadBalancer:  PingTask executing ["
                             + numCandidates + "] servers configured");
            }

            for (int i = 0; i < numCandidates; i++) {
                results[i] = false; /* Default answer is DEAD. */
                try {
                    if (ping != null) {
                        results[i] = ping.isAlive(servers[i]);
                    }
                } catch (Throwable t) {
                    logger.error("Exception while pinging Server:"
                                 + servers[i], t);
                }
            }
            return results;
        }
    }
```



## ILoadBalancer.java

```java
package com.netflix.loadbalancer;

import java.util.List;

/**
 * 该接口定义软件负载均衡器的操作。典型的loadbalancer至少需要一组用于loadbalance的服务器、标记某个服务器停止运行的方法以及从现有服务器列表中选择服务器
 * 的调用。
 */
public interface ILoadBalancer {

	public void addServers(List<Server> newServers);

	public Server chooseServer(Object key);

	public void markServerDown(Server server);

	@Deprecated
	public List<Server> getServerList(boolean availableOnly);

    public List<Server> getReachableServers();

	public List<Server> getAllServers();
}

```

### AbstractLoadBalancer.java

```java
package com.netflix.loadbalancer;

public abstract class AbstractLoadBalancer implements ILoadBalancer {
    
    public enum ServerGroup{
        ALL,
        STATUS_UP,
        STATUS_NOT_UP        
    }

    public Server chooseServer() {
    	return chooseServer(null);
    }

    public abstract List<Server> getServerList(ServerGroup serverGroup);

    public abstract LoadBalancerStats getLoadBalancerStats();    
}

```

#### BaseLoadBalancer.java

```java
package com.netflix.loadbalancer;
/**
 * 负载均衡器的基本实现，其中可以将任意服务器列表设置为服务器池。可以设置ping来确定服务器的活动状态。在内部，该类维护一个“all”服务器列表和一个“up”服务器列表，并根据调用者的要求使用它们。
 */
public class BaseLoadBalancer extends AbstractLoadBalancer implements
        PrimeConnections.PrimeConnectionListener, IClientConfigAware {

    private static Logger logger = LoggerFactory
            .getLogger(BaseLoadBalancer.class);
    private final static IRule DEFAULT_RULE = new RoundRobinRule();
    private final static SerialPingStrategy DEFAULT_PING_STRATEGY = new SerialPingStrategy();
    private static final String DEFAULT_NAME = "default";
    private static final String PREFIX = "LoadBalancer_";

    protected IRule rule = DEFAULT_RULE;

    protected IPingStrategy pingStrategy = DEFAULT_PING_STRATEGY;

    protected IPing ping = null;

    @Monitor(name = PREFIX + "AllServerList", type = DataSourceType.INFORMATIONAL)
    protected volatile List<Server> allServerList = Collections
            .synchronizedList(new ArrayList<Server>());
    @Monitor(name = PREFIX + "UpServerList", type = DataSourceType.INFORMATIONAL)
    protected volatile List<Server> upServerList = Collections
            .synchronizedList(new ArrayList<Server>());

    protected ReadWriteLock allServerLock = new ReentrantReadWriteLock();
    protected ReadWriteLock upServerLock = new ReentrantReadWriteLock();

    protected String name = DEFAULT_NAME;

    protected Timer lbTimer = null;
    protected int pingIntervalSeconds = 10;
    protected int maxTotalPingTimeSeconds = 5;
    protected Comparator<Server> serverComparator = new ServerComparator();

    protected AtomicBoolean pingInProgress = new AtomicBoolean(false);

    protected LoadBalancerStats lbStats;

    private volatile Counter counter = Monitors.newCounter("LoadBalancer_ChooseServer");

    private PrimeConnections primeConnections;

    private volatile boolean enablePrimingConnections = false;
    
    private IClientConfig config;
    
    private List<ServerListChangeListener> changeListeners = new CopyOnWriteArrayList<ServerListChangeListener>();

    private List<ServerStatusChangeListener> serverStatusListeners = new CopyOnWriteArrayList<ServerStatusChangeListener>();

    /**
     * Default constructor which sets name as "default", sets null ping, and
     * {@link RoundRobinRule} as the rule.
     * <p>
     * This constructor is mainly used by {@link ClientFactory}. Calling this
     * constructor must be followed by calling {@link #init()} or
     * {@link #initWithNiwsConfig(IClientConfig)} to complete initialization.
     * This constructor is provided for reflection. When constructing
     * programatically, it is recommended to use other constructors.
     */
    public BaseLoadBalancer() {
        this.name = DEFAULT_NAME;
        this.ping = null;
        setRule(DEFAULT_RULE);
        setupPingTask();
        lbStats = new LoadBalancerStats(DEFAULT_NAME);
    }

    public BaseLoadBalancer(String lbName, IRule rule, LoadBalancerStats lbStats) {
        this(lbName, rule, lbStats, null);
    }

    public BaseLoadBalancer(IPing ping, IRule rule) {
        this(DEFAULT_NAME, rule, new LoadBalancerStats(DEFAULT_NAME), ping);
    }

    public BaseLoadBalancer(IPing ping, IRule rule, IPingStrategy pingStrategy) {
        this(DEFAULT_NAME, rule, new LoadBalancerStats(DEFAULT_NAME), ping, pingStrategy);
    }

    public BaseLoadBalancer(String name, IRule rule, LoadBalancerStats stats,
            IPing ping) {
        this(name, rule, stats, ping, DEFAULT_PING_STRATEGY);
    }
    
    public BaseLoadBalancer(String name, IRule rule, LoadBalancerStats stats,
            IPing ping, IPingStrategy pingStrategy) {
        if (logger.isDebugEnabled()) {
            logger.debug("LoadBalancer:  initialized");
        }
        this.name = name;
        this.ping = ping;
        this.pingStrategy = pingStrategy;
        setRule(rule);
        setupPingTask();
        lbStats = stats;
        init();
    }

    public BaseLoadBalancer(IClientConfig config) {
        initWithNiwsConfig(config);
    }

    public BaseLoadBalancer(IClientConfig config, IRule rule, IPing ping) {
        initWithConfig(config, rule, ping);
    }
    
    void initWithConfig(IClientConfig clientConfig, IRule rule, IPing ping) {
        this.config = clientConfig;
        String clientName = clientConfig.getClientName();
        this.name = clientName;
        int pingIntervalTime = Integer.parseInt(""
                + clientConfig.getProperty(
                        CommonClientConfigKey.NFLoadBalancerPingInterval,
                        Integer.parseInt("30")));
        int maxTotalPingTime = Integer.parseInt(""
                + clientConfig.getProperty(
                        CommonClientConfigKey.NFLoadBalancerMaxTotalPingTime,
                        Integer.parseInt("2")));

        setPingInterval(pingIntervalTime);
        setMaxTotalPingTime(maxTotalPingTime);

        // cross associate with each other
        // i.e. Rule,Ping meet your container LB
        // LB, these are your Ping and Rule guys ...
        setRule(rule);
        setPing(ping);
        setLoadBalancerStats(new LoadBalancerStats(clientName));
        rule.setLoadBalancer(this);
        if (ping instanceof AbstractLoadBalancerPing) {
            ((AbstractLoadBalancerPing) ping).setLoadBalancer(this);
        }
        logger.info("Client:" + name + " instantiated a LoadBalancer:"
                + toString());
        boolean enablePrimeConnections = clientConfig.get(
                CommonClientConfigKey.EnablePrimeConnections, DefaultClientConfigImpl.DEFAULT_ENABLE_PRIME_CONNECTIONS);

        if (enablePrimeConnections) {
            this.setEnablePrimingConnections(true);
            PrimeConnections primeConnections = new PrimeConnections(
                    this.getName(), clientConfig);
            this.setPrimeConnections(primeConnections);
        }
        init();

    }
    
    @Override
    public void initWithNiwsConfig(IClientConfig clientConfig) {
        String ruleClassName = (String) clientConfig
                .getProperty(CommonClientConfigKey.NFLoadBalancerRuleClassName);
        String pingClassName = (String) clientConfig
                .getProperty(CommonClientConfigKey.NFLoadBalancerPingClassName);

        IRule rule;
        IPing ping;
        try {
            rule = (IRule) ClientFactory.instantiateInstanceWithClientConfig(
                    ruleClassName, clientConfig);
            ping = (IPing) ClientFactory.instantiateInstanceWithClientConfig(
                    pingClassName, clientConfig);
        } catch (Exception e) {
            throw new RuntimeException("Error initializing load balancer", e);
        }
        initWithConfig(clientConfig, rule, ping);
    }

    public void addServerListChangeListener(ServerListChangeListener listener) {
        changeListeners.add(listener);
    }
    
    public void removeServerListChangeListener(ServerListChangeListener listener) {
        changeListeners.remove(listener);
    }

    public void addServerStatusChangeListener(ServerStatusChangeListener listener) {
        serverStatusListeners.add(listener);
    }

    public void removeServerStatusChangeListener(ServerStatusChangeListener listener) {
        serverStatusListeners.remove(listener);
    }

    public IClientConfig getClientConfig() {
    	return config;
    }
    
    private boolean canSkipPing() {
        if (ping == null
                || ping.getClass().getName().equals(DummyPing.class.getName())) {
            // default ping, no need to set up timer
            return true;
        } else {
            return false;
        }
    }

    void setupPingTask() {
        if (canSkipPing()) {
            return;
        }
        if (lbTimer != null) {
            lbTimer.cancel();
        }
        lbTimer = new ShutdownEnabledTimer("NFLoadBalancer-PingTimer-" + name,
                true);
        lbTimer.schedule(new PingTask(), 0, pingIntervalSeconds * 1000);
        forceQuickPing();
    }

    /**
     * Set the name for the load balancer. This should not be called since name
     * should be immutable after initialization. Calling this method does not
     * guarantee that all other data structures that depend on this name will be
     * changed accordingly.
     */
    void setName(String name) {
        // and register
        this.name = name;
        if (lbStats == null) {
            lbStats = new LoadBalancerStats(name);
        } else {
            lbStats.setName(name);
        }
    }

    public String getName() {
        return name;
    }

    @Override
    public LoadBalancerStats getLoadBalancerStats() {
        return lbStats;
    }

    public void setLoadBalancerStats(LoadBalancerStats lbStats) {
        this.lbStats = lbStats;
    }

    public Lock lockAllServerList(boolean write) {
        Lock aproposLock = write ? allServerLock.writeLock() : allServerLock
                .readLock();
        aproposLock.lock();
        return aproposLock;
    }

    public Lock lockUpServerList(boolean write) {
        Lock aproposLock = write ? upServerLock.writeLock() : upServerLock
                .readLock();
        aproposLock.lock();
        return aproposLock;
    }

    public void setPingInterval(int pingIntervalSeconds) {
        if (pingIntervalSeconds < 1) {
            return;
        }

        this.pingIntervalSeconds = pingIntervalSeconds;
        if (logger.isDebugEnabled()) {
            logger.debug("LoadBalancer:  pingIntervalSeconds set to "
                    + this.pingIntervalSeconds);
        }
        setupPingTask(); // since ping data changed
    }

    public int getPingInterval() {
        return pingIntervalSeconds;
    }

    /*
     * Maximum time allowed for the ping cycle
     */
    public void setMaxTotalPingTime(int maxTotalPingTimeSeconds) {
        if (maxTotalPingTimeSeconds < 1) {
            return;
        }
        this.maxTotalPingTimeSeconds = maxTotalPingTimeSeconds;
        if (logger.isDebugEnabled()) {
            logger.debug("LoadBalancer: maxTotalPingTime set to "
                    + this.maxTotalPingTimeSeconds);
        }

    }

    public int getMaxTotalPingTime() {
        return maxTotalPingTimeSeconds;
    }

    public IPing getPing() {
        return ping;
    }

    public IRule getRule() {
        return rule;
    }

    public boolean isPingInProgress() {
        return pingInProgress.get();
    }

    /* Specify the object which is used to send pings. */

    public void setPing(IPing ping) {
        if (ping != null) {
            if (!ping.equals(this.ping)) {
                this.ping = ping;
                setupPingTask(); // since ping data changed
            }
        } else {
            this.ping = null;
            // cancel the timer task
            lbTimer.cancel();
        }
    }

    /* Ignore null rules */

    public void setRule(IRule rule) {
        if (rule != null) {
            this.rule = rule;
        } else {
            /* default rule */
            this.rule = new RoundRobinRule();
        }
        if (this.rule.getLoadBalancer() != this) {
            this.rule.setLoadBalancer(this);
        }
    }

    /**
     * get the count of servers.
     * 
     * @param onlyAvailable
     *            if true, return only up servers.
     */
    public int getServerCount(boolean onlyAvailable) {
        if (onlyAvailable) {
            return upServerList.size();
        } else {
            return allServerList.size();
        }
    }

    /**
     * Add a server to the 'allServer' list; does not verify uniqueness, so you
     * could give a server a greater share by adding it more than once.
     */
    public void addServer(Server newServer) {
        if (newServer != null) {
            try {
                ArrayList<Server> newList = new ArrayList<Server>();

                newList.addAll(allServerList);
                newList.add(newServer);
                setServersList(newList);
            } catch (Exception e) {
                logger.error("Exception while adding a newServer", e);
            }
        }
    }

    /**
     * Add a list of servers to the 'allServer' list; does not verify
     * uniqueness, so you could give a server a greater share by adding it more
     * than once
     */
    @Override
    public void addServers(List<Server> newServers) {
        if (newServers != null && newServers.size() > 0) {
            try {
                ArrayList<Server> newList = new ArrayList<Server>();
                newList.addAll(allServerList);
                newList.addAll(newServers);
                setServersList(newList);
            } catch (Exception e) {
                logger.error("Exception while adding Servers", e);
            }
        }
    }

    /*
     * Add a list of servers to the 'allServer' list; does not verify
     * uniqueness, so you could give a server a greater share by adding it more
     * than once USED by Test Cases only for legacy reason. DO NOT USE!!
     */
    void addServers(Object[] newServers) {
        if ((newServers != null) && (newServers.length > 0)) {

            try {
                ArrayList<Server> newList = new ArrayList<Server>();
                newList.addAll(allServerList);

                for (Object server : newServers) {
                    if (server != null) {
                        if (server instanceof String) {
                            server = new Server((String) server);
                        }
                        if (server instanceof Server) {
                            newList.add((Server) server);
                        }
                    }
                }
                setServersList(newList);
            } catch (Exception e) {
                logger.error("Exception while adding Servers", e);
            }
        }
    }

    /**
     * Set the list of servers used as the server pool. This overrides existing
     * server list.
     */
    public void setServersList(List lsrv) {
        Lock writeLock = allServerLock.writeLock();
        if (logger.isDebugEnabled()) {
            logger.debug("LoadBalancer:  clearing server list (SET op)");
        }
        ArrayList<Server> newServers = new ArrayList<Server>();
        writeLock.lock();
        try {
            ArrayList<Server> allServers = new ArrayList<Server>();
            for (Object server : lsrv) {
                if (server == null) {
                    continue;
                }

                if (server instanceof String) {
                    server = new Server((String) server);
                }

                if (server instanceof Server) {
                    if (logger.isDebugEnabled()) {
                        logger.debug("LoadBalancer:  addServer ["
                                + ((Server) server).getId() + "]");
                    }
                    allServers.add((Server) server);
                } else {
                    throw new IllegalArgumentException(
                            "Type String or Server expected, instead found:"
                                    + server.getClass());
                }

            }
            boolean listChanged = false;
            if (!allServerList.equals(allServers)) {
                listChanged = true;
                if (changeListeners != null && changeListeners.size() > 0) {
                   List<Server> oldList = ImmutableList.copyOf(allServerList);
                   List<Server> newList = ImmutableList.copyOf(allServers);                   
                   for (ServerListChangeListener l: changeListeners) {
                       try {
                           l.serverListChanged(oldList, newList);
                       } catch (Throwable e) {
                           logger.error("Error invoking server list change listener", e);
                       }
                   }
                }
            }
            if (isEnablePrimingConnections()) {
                for (Server server : allServers) {
                    if (!allServerList.contains(server)) {
                        server.setReadyToServe(false);
                        newServers.add((Server) server);
                    }
                }
                if (primeConnections != null) {
                    primeConnections.primeConnectionsAsync(newServers, this);
                }
            }
            // This will reset readyToServe flag to true on all servers
            // regardless whether
            // previous priming connections are success or not
            allServerList = allServers;
            if (canSkipPing()) {
                for (Server s : allServerList) {
                    s.setAlive(true);
                }
                upServerList = allServerList;
            } else if (listChanged) {
                forceQuickPing();
            }
        } finally {
            writeLock.unlock();
        }
    }

    /* List in string form. SETS, does not add. */
    void setServers(String srvString) {
        if (srvString != null) {

            try {
                String[] serverArr = srvString.split(",");
                ArrayList<Server> newList = new ArrayList<Server>();

                for (String serverString : serverArr) {
                    if (serverString != null) {
                        serverString = serverString.trim();
                        if (serverString.length() > 0) {
                            Server svr = new Server(serverString);
                            newList.add(svr);
                        }
                    }
                }
                setServersList(newList);
            } catch (Exception e) {
                logger.error("Exception while adding Servers", e);
            }
        }
    }

    /**
     * return the server
     * 
     * @param index
     * @param availableOnly
     */
    public Server getServerByIndex(int index, boolean availableOnly) {
        try {
            return (availableOnly ? upServerList.get(index) : allServerList
                    .get(index));
        } catch (Exception e) {
            return null;
        }
    }

    @Override
    public List<Server> getServerList(boolean availableOnly) {
        return (availableOnly ? getReachableServers() : getAllServers());
    }

    @Override
    public List<Server> getReachableServers() {
        return Collections.unmodifiableList(upServerList);
    }

    @Override
    public List<Server> getAllServers() {
        return Collections.unmodifiableList(allServerList);
    }

    @Override
    public List<Server> getServerList(ServerGroup serverGroup) {
        switch (serverGroup) {
        case ALL:
            return allServerList;
        case STATUS_UP:
            return upServerList;
        case STATUS_NOT_UP:
            ArrayList<Server> notAvailableServers = new ArrayList<Server>(
                    allServerList);
            ArrayList<Server> upServers = new ArrayList<Server>(upServerList);
            notAvailableServers.removeAll(upServers);
            return notAvailableServers;
        }
        return new ArrayList<Server>();
    }

    public void cancelPingTask() {
        if (lbTimer != null) {
            lbTimer.cancel();
        }
    }

    /**
     * TimerTask that keeps runs every X seconds to check the status of each
     * server/node in the Server List
     * 
     * @author stonse
     * 
     */
    class PingTask extends TimerTask {
        public void run() {
            Pinger ping = new Pinger(pingStrategy);
            try {
                ping.runPinger();
            } catch (Throwable t) {
                logger.error("Throwable caught while running extends for "
                        + name, t);
            }
        }
    }

    /**
     * Class that contains the mechanism to "ping" all the instances
     * 
     * @author stonse
     *
     */
    class Pinger {

        private final IPingStrategy pingerStrategy;

        public Pinger(IPingStrategy pingerStrategy) {
            this.pingerStrategy = pingerStrategy;
        }

        public void runPinger() {
            if (pingInProgress.get()) {
                return; // Ping in progress - nothing to do
            } else {
                pingInProgress.set(true);
            }
            // we are "in" - we get to Ping

            Server[] allServers = null;
            boolean[] results = null;

            Lock allLock = null;
            Lock upLock = null;

            try {
                /*
                 * The readLock should be free unless an addServer operation is
                 * going on...
                 */
                allLock = allServerLock.readLock();
                allLock.lock();
                allServers = allServerList.toArray(new Server[allServerList.size()]);
                allLock.unlock();

                int numCandidates = allServers.length;
                results = pingerStrategy.pingServers(ping, allServers);

                final List<Server> newUpList = new ArrayList<Server>();
                final List<Server> changedServers = new ArrayList<Server>();

                for (int i = 0; i < numCandidates; i++) {
                    boolean isAlive = results[i];
                    Server svr = allServers[i];
                    boolean oldIsAlive = svr.isAlive();

                    svr.setAlive(isAlive);

                    if (oldIsAlive != isAlive) {
                        changedServers.add(svr);
                        if (logger.isDebugEnabled()) {
                            logger.debug("LoadBalancer:  Server [" + svr.getId()
                                    + "] status changed to "
                                    + (isAlive ? "ALIVE" : "DEAD"));
                        }
                    }

                    if (isAlive) {
                        newUpList.add(svr);
                    }
                }
                upLock = upServerLock.writeLock();
                upLock.lock();
                upServerList = newUpList;
                upLock.unlock();

                notifyServerStatusChangeListener(changedServers);

            } catch (Throwable t) {
                logger.error("Throwable caught while running the Pinger-"
                        + name, t);
            } finally {
                pingInProgress.set(false);
            }
        }
    }

    private void notifyServerStatusChangeListener(final Collection<Server> changedServers) {
        if (changedServers != null && !changedServers.isEmpty() && !serverStatusListeners.isEmpty()) {
            for (ServerStatusChangeListener listener : serverStatusListeners) {
                try {
                    listener.serverStatusChanged(changedServers);
                } catch (Throwable e) {
                    logger.error("Error invoking server status change listener", e);
                }
            }
        }
    }

    private final Counter createCounter() {
        return Monitors.newCounter("LoadBalancer_ChooseServer");
    }

    /*
     * Get the alive server dedicated to key
     * 
     * @return the dedicated server
     */
    public Server chooseServer(Object key) {
        if (counter == null) {
            counter = createCounter();
        }
        counter.increment();
        if (rule == null) {
            return null;
        } else {
            try {
                return rule.choose(key);
            } catch (Throwable t) {
                return null;
            }
        }
    }

    /* Returns either null, or "server:port/servlet" */
    public String choose(Object key) {
        if (rule == null) {
            return null;
        } else {
            try {
                Server svr = rule.choose(key);
                return ((svr == null) ? null : svr.getId());
            } catch (Throwable t) {
                return null;
            }
        }
    }

    public void markServerDown(Server server) {
        if (server == null) {
            return;
        }

        if (!server.isAlive()) {
            return;
        }

        logger.error("LoadBalancer:  markServerDown called on ["
                + server.getId() + "]");
        server.setAlive(false);
        // forceQuickPing();

        notifyServerStatusChangeListener(singleton(server));
    }

    public void markServerDown(String id) {
        boolean triggered = false;

        id = Server.normalizeId(id);

        if (id == null) {
            return;
        }

        Lock writeLock = upServerLock.writeLock();

        try {

            final List<Server> changedServers = new ArrayList<Server>();

            for (Server svr : upServerList) {
                if (svr.isAlive() && (svr.getId().equals(id))) {
                    triggered = true;
                    svr.setAlive(false);
                    changedServers.add(svr);
                }
            }

            if (triggered) {
                logger.error("LoadBalancer:  markServerDown called on [" + id
                        + "]");
                notifyServerStatusChangeListener(changedServers);
            }

        } finally {
            try {
                writeLock.unlock();
            } catch (Exception e) { // NOPMD
            }
        }
    }

    /*
     * Force an immediate ping, if we're not currently pinging and don't have a
     * quick-ping already scheduled.
     */
    public void forceQuickPing() {
        if (canSkipPing()) {
            return;
        }
        if (logger.isDebugEnabled()) {
            logger.debug("LoadBalancer:  forceQuickPing invoked");
        }
        Pinger ping = new Pinger(pingStrategy);
        try {
            ping.runPinger();
        } catch (Throwable t) {
            logger.error("Throwable caught while running forceQuickPing() for "
                    + name, t);
        }
    }

    public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append("{NFLoadBalancer:name=").append(this.getName())
                .append(",current list of Servers=").append(this.allServerList)
                .append(",Load balancer stats=")
                .append(this.lbStats.toString()).append("}");
        return sb.toString();
    }

    /**
     * Register with monitors and start priming connections if it is set.
     */
    protected void init() {
        Monitors.registerObject("LoadBalancer_" + name, this);
        // register the rule as it contains metric for available servers count
        Monitors.registerObject("Rule_" + name, this.getRule());
        if (enablePrimingConnections && primeConnections != null) {
            primeConnections.primeConnections(getReachableServers());
        }
    }

    public final PrimeConnections getPrimeConnections() {
        return primeConnections;
    }

    public final void setPrimeConnections(PrimeConnections primeConnections) {
        this.primeConnections = primeConnections;
    }

    @Override
    public void primeCompleted(Server s, Throwable lastException) {
        s.setReadyToServe(true);
    }

    public boolean isEnablePrimingConnections() {
        return enablePrimingConnections;
    }

    public final void setEnablePrimingConnections(
            boolean enablePrimingConnections) {
        this.enablePrimingConnections = enablePrimingConnections;
    }
    
    public void shutdown() {
        cancelPingTask();
        if (primeConnections != null) {
            primeConnections.shutdown();
        }
        Monitors.unregisterObject("LoadBalancer_" + name, this);
        Monitors.unregisterObject("Rule_" + name, this.getRule());
    }
}

```

##### DynamicServerListLoadBalancer.java

```java
package com.netflix.loadbalancer;
/**
 * LoadBalancer具有使用动态源获取服务器候选列表的功能。也就是说，服务器列表可以在运行时更改。它还包含一些工具，其中可以通过筛选条件传递服务器列表，以筛选出不满足所需条件的服务器。
 */
public class DynamicServerListLoadBalancer<T extends Server> extends BaseLoadBalancer {
    private static final Logger LOGGER = LoggerFactory.getLogger(DynamicServerListLoadBalancer.class);

    boolean isSecure = false;
    boolean useTunnel = false;

    // to keep track of modification of server lists
    protected AtomicBoolean serverListUpdateInProgress = new AtomicBoolean(false);

    volatile ServerList<T> serverListImpl;

    volatile ServerListFilter<T> filter;

    protected final ServerListUpdater.UpdateAction updateAction = new ServerListUpdater.UpdateAction() {
        @Override
        public void doUpdate() {
            updateListOfServers();
        }
    };

    protected volatile ServerListUpdater serverListUpdater;

    public DynamicServerListLoadBalancer() {
        super();
    }

    @Deprecated
    public DynamicServerListLoadBalancer(IClientConfig clientConfig, IRule rule, IPing ping, 
            ServerList<T> serverList, ServerListFilter<T> filter) {
        this(
                clientConfig,
                rule,
                ping,
                serverList,
                filter,
                new PollingServerListUpdater()
        );
    }

    public DynamicServerListLoadBalancer(IClientConfig clientConfig, IRule rule, IPing ping,
                                         ServerList<T> serverList, ServerListFilter<T> filter,
                                         ServerListUpdater serverListUpdater) {
        super(clientConfig, rule, ping);
        this.serverListImpl = serverList;
        this.filter = filter;
        this.serverListUpdater = serverListUpdater;
        if (filter instanceof AbstractServerListFilter) {
            ((AbstractServerListFilter) filter).setLoadBalancerStats(getLoadBalancerStats());
        }
        restOfInit(clientConfig);
    }

    public DynamicServerListLoadBalancer(IClientConfig clientConfig) {
        initWithNiwsConfig(clientConfig);
    }
    
    @Override
    public void initWithNiwsConfig(IClientConfig clientConfig) {
        try {
            super.initWithNiwsConfig(clientConfig);
            String niwsServerListClassName = clientConfig.getPropertyAsString(
                    CommonClientConfigKey.NIWSServerListClassName,
                    DefaultClientConfigImpl.DEFAULT_SEVER_LIST_CLASS);

            ServerList<T> niwsServerListImpl = (ServerList<T>) ClientFactory
                    .instantiateInstanceWithClientConfig(niwsServerListClassName, clientConfig);
            this.serverListImpl = niwsServerListImpl;

            if (niwsServerListImpl instanceof AbstractServerList) {
                AbstractServerListFilter<T> niwsFilter = ((AbstractServerList) niwsServerListImpl)
                        .getFilterImpl(clientConfig);
                niwsFilter.setLoadBalancerStats(getLoadBalancerStats());
                this.filter = niwsFilter;
            }

            String serverListUpdaterClassName = clientConfig.getPropertyAsString(
                    CommonClientConfigKey.ServerListUpdaterClassName,
                    DefaultClientConfigImpl.DEFAULT_SERVER_LIST_UPDATER_CLASS
            );

            this.serverListUpdater = (ServerListUpdater) ClientFactory
                    .instantiateInstanceWithClientConfig(serverListUpdaterClassName, clientConfig);

            restOfInit(clientConfig);
        } catch (Exception e) {
            throw new RuntimeException(
                    "Exception while initializing NIWSDiscoveryLoadBalancer:"
                            + clientConfig.getClientName()
                            + ", niwsClientConfig:" + clientConfig, e);
        }
    }

    void restOfInit(IClientConfig clientConfig) {
        boolean primeConnection = this.isEnablePrimingConnections();
        // turn this off to avoid duplicated asynchronous priming done in BaseLoadBalancer.setServerList()
        this.setEnablePrimingConnections(false);
        enableAndInitLearnNewServersFeature();

        updateListOfServers();
        if (primeConnection && this.getPrimeConnections() != null) {
            this.getPrimeConnections()
                    .primeConnections(getReachableServers());
        }
        this.setEnablePrimingConnections(primeConnection);
        LOGGER.info("DynamicServerListLoadBalancer for client {} initialized: {}", clientConfig.getClientName(), this.toString());
    }
    
    
    @Override
    public void setServersList(List lsrv) {
        super.setServersList(lsrv);
        List<T> serverList = (List<T>) lsrv;
        Map<String, List<Server>> serversInZones = new HashMap<String, List<Server>>();
        for (Server server : serverList) {
            // make sure ServerStats is created to avoid creating them on hot
            // path
            getLoadBalancerStats().getSingleServerStat(server);
            String zone = server.getZone();
            if (zone != null) {
                zone = zone.toLowerCase();
                List<Server> servers = serversInZones.get(zone);
                if (servers == null) {
                    servers = new ArrayList<Server>();
                    serversInZones.put(zone, servers);
                }
                servers.add(server);
            }
        }
        setServerListForZones(serversInZones);
    }

    protected void setServerListForZones(
            Map<String, List<Server>> zoneServersMap) {
        LOGGER.debug("Setting server list for zones: {}", zoneServersMap);
        getLoadBalancerStats().updateZoneServerMapping(zoneServersMap);
    }

    public ServerList<T> getServerListImpl() {
        return serverListImpl;
    }

    public void setServerListImpl(ServerList<T> niwsServerList) {
        this.serverListImpl = niwsServerList;
    }

    public ServerListFilter<T> getFilter() {
        return filter;
    }

    public void setFilter(ServerListFilter<T> filter) {
        this.filter = filter;
    }

    @Override
    /**
     * Makes no sense to ping an inmemory disc client
     * 
     */
    public void forceQuickPing() {
        // no-op
    }

    /**
     * Feature that lets us add new instances (from AMIs) to the list of
     * existing servers that the LB will use Call this method if you want this
     * feature enabled
     */
    public void enableAndInitLearnNewServersFeature() {
        LOGGER.info("Using serverListUpdater {}", serverListUpdater.getClass().getSimpleName());
        serverListUpdater.start(updateAction);
    }

    private String getIdentifier() {
        return this.getClientConfig().getClientName();
    }

    public void stopServerListRefreshing() {
        if (serverListUpdater != null) {
            serverListUpdater.stop();
        }
    }

    @VisibleForTesting
    public void updateListOfServers() {
        List<T> servers = new ArrayList<T>();
        if (serverListImpl != null) {
            servers = serverListImpl.getUpdatedListOfServers();
            LOGGER.debug("List of Servers for {} obtained from Discovery client: {}",
                    getIdentifier(), servers);

            if (filter != null) {
                servers = filter.getFilteredListOfServers(servers);
                LOGGER.debug("Filtered List of Servers for {} obtained from Discovery client: {}",
                        getIdentifier(), servers);
            }
        }
        updateAllServerList(servers);
    }

    /**
     * Update the AllServer list in the LoadBalancer if necessary and enabled
     * 
     * @param ls
     */
    protected void updateAllServerList(List<T> ls) {
        // other threads might be doing this - in which case, we pass
        if (serverListUpdateInProgress.compareAndSet(false, true)) {
            for (T s : ls) {
                s.setAlive(true); // set so that clients can start using these
                                  // servers right away instead
                // of having to wait out the ping cycle.
            }
            setServersList(ls);
            super.forceQuickPing();
            serverListUpdateInProgress.set(false);
        }
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder("DynamicServerListLoadBalancer:");
        sb.append(super.toString());
        sb.append("ServerList:" + String.valueOf(serverListImpl));
        return sb.toString();
    }
    
    @Override 
    public void shutdown() {
        super.shutdown();
        stopServerListRefreshing();
    }


    @Monitor(name="LastUpdated", type=DataSourceType.INFORMATIONAL)
    public String getLastUpdate() {
        return serverListUpdater.getLastUpdate();
    }

    @Monitor(name="DurationSinceLastUpdateMs", type= DataSourceType.GAUGE)
    public long getDurationSinceLastUpdateMs() {
        return serverListUpdater.getDurationSinceLastUpdateMs();
    }

    @Monitor(name="NumUpdateCyclesMissed", type=DataSourceType.GAUGE)
    public int getNumberMissedCycles() {
        return serverListUpdater.getNumberMissedCycles();
    }

    @Monitor(name="NumThreads", type=DataSourceType.GAUGE)
    public int getCoreThreads() {
        return serverListUpdater.getCoreThreads();
    }
}

```



## LoadBalancerContext.java

```java
package com.netflix.loadbalancer;
/**
 * A class contains APIs intended to be used be load balancing client which is subclass of this class.
 */
public class LoadBalancerContext implements IClientConfigAware {
    private static final Logger logger = LoggerFactory.getLogger(LoadBalancerContext.class);

    protected String clientName = "default";          

    protected String vipAddresses;

    protected int maxAutoRetriesNextServer = DefaultClientConfigImpl.DEFAULT_MAX_AUTO_RETRIES_NEXT_SERVER;
    protected int maxAutoRetries = DefaultClientConfigImpl.DEFAULT_MAX_AUTO_RETRIES;

    protected RetryHandler defaultRetryHandler = new DefaultLoadBalancerRetryHandler();


    protected boolean okToRetryOnAllOperations = DefaultClientConfigImpl.DEFAULT_OK_TO_RETRY_ON_ALL_OPERATIONS.booleanValue();

    private ILoadBalancer lb;

    private volatile Timer tracer;

    public LoadBalancerContext(ILoadBalancer lb) {
        this.lb = lb;
    }

    /**
     * Delegate to {@link #initWithNiwsConfig(IClientConfig)}
     * @param clientConfig
     */
    public LoadBalancerContext(ILoadBalancer lb, IClientConfig clientConfig) {
        this.lb = lb;
        initWithNiwsConfig(clientConfig);        
    }

    public LoadBalancerContext(ILoadBalancer lb, IClientConfig clientConfig, RetryHandler handler) {
        this(lb, clientConfig);
        this.defaultRetryHandler = handler;
    }

    /**
     * Set necessary parameters from client configuration and register with Servo monitors.
     */
    @Override
    public void initWithNiwsConfig(IClientConfig clientConfig) {
        if (clientConfig == null) {
            return;    
        }
        clientName = clientConfig.getClientName();
        if (clientName == null) {
            clientName = "default";
        }
        vipAddresses = clientConfig.resolveDeploymentContextbasedVipAddresses();
        maxAutoRetries = clientConfig.getPropertyAsInteger(CommonClientConfigKey.MaxAutoRetries, DefaultClientConfigImpl.DEFAULT_MAX_AUTO_RETRIES);
        maxAutoRetriesNextServer = clientConfig.getPropertyAsInteger(CommonClientConfigKey.MaxAutoRetriesNextServer,maxAutoRetriesNextServer);

        okToRetryOnAllOperations = clientConfig.getPropertyAsBoolean(CommonClientConfigKey.OkToRetryOnAllOperations, okToRetryOnAllOperations);
        defaultRetryHandler = new DefaultLoadBalancerRetryHandler(clientConfig);
        
        tracer = getExecuteTracer();

        Monitors.registerObject("Client_" + clientName, this);
    }

    public Timer getExecuteTracer() {
        if (tracer == null) {
            synchronized(this) {
                if (tracer == null) {
                    tracer = Monitors.newTimer(clientName + "_LoadBalancerExecutionTimer", TimeUnit.MILLISECONDS);                    
                }
            }
        } 
        return tracer;        
    }

    public String getClientName() {
        return clientName;
    }

    public ILoadBalancer getLoadBalancer() {
        return lb;    
    }

    public void setLoadBalancer(ILoadBalancer lb) {
        this.lb = lb;
    }

    /**
     * Use {@link #getRetryHandler()} 
     */
    @Deprecated
    public int getMaxAutoRetriesNextServer() {
        return maxAutoRetriesNextServer;
    }

    /**
     * Use {@link #setRetryHandler(RetryHandler)} 
     */
    @Deprecated
    public void setMaxAutoRetriesNextServer(int maxAutoRetriesNextServer) {
        this.maxAutoRetriesNextServer = maxAutoRetriesNextServer;
    }

    /**
     * Use {@link #getRetryHandler()} 
     */
    @Deprecated
    public int getMaxAutoRetries() {
        return maxAutoRetries;
    }

    /**
     * Use {@link #setRetryHandler(RetryHandler)} 
     */
    @Deprecated
    public void setMaxAutoRetries(int maxAutoRetries) {
        this.maxAutoRetries = maxAutoRetries;
    }

    protected Throwable getDeepestCause(Throwable e) {
        if(e != null) {
            int infiniteLoopPreventionCounter = 10;
            while (e.getCause() != null && infiniteLoopPreventionCounter > 0) {
                infiniteLoopPreventionCounter--;
                e = e.getCause();
            }
        }
        return e;
    }

    private boolean isPresentAsCause(Throwable throwableToSearchIn,
            Class<? extends Throwable> throwableToSearchFor) {
        return isPresentAsCauseHelper(throwableToSearchIn, throwableToSearchFor) != null;
    }

    static Throwable isPresentAsCauseHelper(Throwable throwableToSearchIn,
            Class<? extends Throwable> throwableToSearchFor) {
        int infiniteLoopPreventionCounter = 10;
        while (throwableToSearchIn != null && infiniteLoopPreventionCounter > 0) {
            infiniteLoopPreventionCounter--;
            if (throwableToSearchIn.getClass().isAssignableFrom(
                    throwableToSearchFor)) {
                return throwableToSearchIn;
            } else {
                throwableToSearchIn = throwableToSearchIn.getCause();
            }
        }
        return null;
    }

    protected ClientException generateNIWSException(String uri, Throwable e){
        ClientException niwsClientException;
        if (isPresentAsCause(e, java.net.SocketTimeoutException.class)) {
            niwsClientException = generateTimeoutNIWSException(uri, e);
        }else if (e.getCause() instanceof java.net.UnknownHostException){
            niwsClientException = new ClientException(
                    ClientException.ErrorType.UNKNOWN_HOST_EXCEPTION,
                    "Unable to execute RestClient request for URI:" + uri,
                    e);
        }else if (e.getCause() instanceof java.net.ConnectException){
            niwsClientException = new ClientException(
                    ClientException.ErrorType.CONNECT_EXCEPTION,
                    "Unable to execute RestClient request for URI:" + uri,
                    e);
        }else if (e.getCause() instanceof java.net.NoRouteToHostException){
            niwsClientException = new ClientException(
                    ClientException.ErrorType.NO_ROUTE_TO_HOST_EXCEPTION,
                    "Unable to execute RestClient request for URI:" + uri,
                    e);
        }else if (e instanceof ClientException){
            niwsClientException = (ClientException)e;
        }else {
            niwsClientException = new ClientException(
                    ClientException.ErrorType.GENERAL,
                    "Unable to execute RestClient request for URI:" + uri,
                    e);
        }
        return niwsClientException;
    }

    private boolean isPresentAsCause(Throwable throwableToSearchIn,
            Class<? extends Throwable> throwableToSearchFor, String messageSubStringToSearchFor) {
        Throwable throwableFound = isPresentAsCauseHelper(throwableToSearchIn, throwableToSearchFor);
        if(throwableFound != null) {
            return throwableFound.getMessage().contains(messageSubStringToSearchFor);
        }
        return false;
    }
    private ClientException generateTimeoutNIWSException(String uri, Throwable e){
        ClientException niwsClientException;
        if (isPresentAsCause(e, java.net.SocketTimeoutException.class,
                "Read timed out")) {
            niwsClientException = new ClientException(
                    ClientException.ErrorType.READ_TIMEOUT_EXCEPTION,
                    "Unable to execute RestClient request for URI:" + uri + ":"
                            + getDeepestCause(e).getMessage(), e);
        } else {
            niwsClientException = new ClientException(
                    ClientException.ErrorType.SOCKET_TIMEOUT_EXCEPTION,
                    "Unable to execute RestClient request for URI:" + uri + ":"
                            + getDeepestCause(e).getMessage(), e);
        }
        return niwsClientException;
    }

    private void recordStats(ServerStats stats, long responseTime) {
        stats.decrementActiveRequestsCount();
        stats.incrementNumRequests();
        stats.noteResponseTime(responseTime);
    }

    protected void noteRequestCompletion(ServerStats stats, Object response, Throwable e, long responseTime) {
        noteRequestCompletion(stats, response, e, responseTime, null);
    }
    
    
    /**
     * This is called after a response is received or an exception is thrown from the client
     * to update related stats.  
     */
    public void noteRequestCompletion(ServerStats stats, Object response, Throwable e, long responseTime, RetryHandler errorHandler) {
        try {
            recordStats(stats, responseTime);
            RetryHandler callErrorHandler = errorHandler == null ? getRetryHandler() : errorHandler;
            if (callErrorHandler != null && response != null) {
                stats.clearSuccessiveConnectionFailureCount();
            } else if (callErrorHandler != null && e != null) {
                if (callErrorHandler.isCircuitTrippingException(e)) {
                    stats.incrementSuccessiveConnectionFailureCount();                    
                    stats.addToFailureCount();
                } else {
                    stats.clearSuccessiveConnectionFailureCount();
                }
            }
        } catch (Throwable ex) {
            logger.error("Unexpected exception", ex);
        }            
    }

    /**
     * This is called after an error is thrown from the client
     * to update related stats.  
     */
    protected void noteError(ServerStats stats, ClientRequest request, Throwable e, long responseTime) {
        try {
            recordStats(stats, responseTime);
            RetryHandler errorHandler = getRetryHandler();
            if (errorHandler != null && e != null) {
                if (errorHandler.isCircuitTrippingException(e)) {
                    stats.incrementSuccessiveConnectionFailureCount();                    
                    stats.addToFailureCount();
                } else {
                    stats.clearSuccessiveConnectionFailureCount();
                }
            }
        } catch (Throwable ex) {
            logger.error("Unexpected exception", ex);
        }            
    }

    /**
     * This is called after a response is received from the client
     * to update related stats.  
     */
    protected void noteResponse(ServerStats stats, ClientRequest request, Object response, long responseTime) {
        try {
            recordStats(stats, responseTime);
            RetryHandler errorHandler = getRetryHandler();
            if (errorHandler != null && response != null) {
                stats.clearSuccessiveConnectionFailureCount();
            } 
        } catch (Throwable ex) {
            logger.error("Unexpected exception", ex);
        }            
    }

    /**
     * This is usually called just before client execute a request.
     */
    public void noteOpenConnection(ServerStats serverStats) {
        if (serverStats == null) {
            return;
        }
        try {
            serverStats.incrementActiveRequestsCount();
        } catch (Throwable e) {
            logger.info("Unable to note Server Stats:", e);
        }
    }


    /**
     * Derive scheme and port from a partial URI. For example, for HTTP based client, the URI with 
     * only path "/" should return "http" and 80, whereas the URI constructed with scheme "https" and
     * path "/" should return "https" and 443.
     * This method is called by {@link #getServerFromLoadBalancer(java.net.URI, Object)} and
     * {@link #reconstructURIWithServer(Server, java.net.URI)} methods to get the complete executable URI.
     */
    protected Pair<String, Integer> deriveSchemeAndPortFromPartialUri(URI uri) {
        boolean isSecure = false;
        String scheme = uri.getScheme();
        if (scheme != null) {
            isSecure =  scheme.equalsIgnoreCase("https");
        }
        int port = uri.getPort();
        if (port < 0 && !isSecure){
            port = 80;
        } else if (port < 0 && isSecure){
            port = 443;
        }
        if (scheme == null){
            if (isSecure) {
                scheme = "https";
            } else {
                scheme = "http";
            }
        }
        return new Pair<String, Integer>(scheme, port);
    }

    /**
     * Get the default port of the target server given the scheme of vip address if it is available. 
     * Subclass should override it to provider protocol specific default port number if any.
     * 
     * @param scheme from the vip address. null if not present.
     * @return 80 if scheme is http, 443 if scheme is https, -1 else.
     */
    protected int getDefaultPortFromScheme(String scheme) {
        if (scheme == null) {
            return -1;
        }
        if (scheme.equals("http")) {
            return 80;
        } else if (scheme.equals("https")) {
            return 443;
        } else {
            return -1;
        }
    }


    /**
     * Derive the host and port from virtual address if virtual address is indeed contains the actual host 
     * and port of the server. This is the final resort to compute the final URI in {@link #getServerFromLoadBalancer(java.net.URI, Object)}
     * if there is no load balancer available and the request URI is incomplete. Sub classes can override this method
     * to be more accurate or throws ClientException if it does not want to support virtual address to be the
     * same as physical server address.
     * <p>
     *  The virtual address is used by certain load balancers to filter the servers of the same function 
     *  to form the server pool. 
     *  
     */
    protected  Pair<String, Integer> deriveHostAndPortFromVipAddress(String vipAddress) 
            throws URISyntaxException, ClientException {
        Pair<String, Integer> hostAndPort = new Pair<String, Integer>(null, -1);
        URI uri = new URI(vipAddress);
        String scheme = uri.getScheme();
        if (scheme == null) {
            uri = new URI("http://" + vipAddress);
        }
        String host = uri.getHost();
        if (host == null) {
            throw new ClientException("Unable to derive host/port from vip address " + vipAddress);
        }
        int port = uri.getPort();
        if (port < 0) {
            port = getDefaultPortFromScheme(scheme);
        }
        if (port < 0) {
            throw new ClientException("Unable to derive host/port from vip address " + vipAddress);
        }
        hostAndPort.setFirst(host);
        hostAndPort.setSecond(port);
        return hostAndPort;
    }

    private boolean isVipRecognized(String vipEmbeddedInUri) {
        if (vipEmbeddedInUri == null) {
            return false;
        }
        if (vipAddresses == null) {
            return false;
        }
        String[] addresses = vipAddresses.split(",");
        for (String address: addresses) {
            if (vipEmbeddedInUri.equalsIgnoreCase(address.trim())) {
                return true;
            }
        }
        return false;
    }

    /**
     * Compute the final URI from a partial URI in the request. The following steps are performed:
     * <ul>
     * <li> if host is missing and there is a load balancer, get the host/port from server chosen from load balancer
     * <li> if host is missing and there is no load balancer, try to derive host/port from virtual address set with the client
     * <li> if host is present and the authority part of the URI is a virtual address set for the client, 
     * and there is a load balancer, get the host/port from server chosen from load balancer
     * <li> if host is present but none of the above applies, interpret the host as the actual physical address
     * <li> if host is missing but none of the above applies, throws ClientException
     * </ul>
     *
     * @param original Original URI passed from caller
     */
    public Server getServerFromLoadBalancer(@Nullable URI original, @Nullable Object loadBalancerKey) throws ClientException {
        String host = null;
        int port = -1;
        if (original != null) {
            host = original.getHost();
        }
        if (original != null) {
            Pair<String, Integer> schemeAndPort = deriveSchemeAndPortFromPartialUri(original);        
            port = schemeAndPort.second();
        }

        // Various Supported Cases
        // The loadbalancer to use and the instances it has is based on how it was registered
        // In each of these cases, the client might come in using Full Url or Partial URL
        ILoadBalancer lb = getLoadBalancer();
        if (host == null) {
            // Partial URI or no URI Case
            // well we have to just get the right instances from lb - or we fall back
            if (lb != null){
                Server svc = lb.chooseServer(loadBalancerKey);
                if (svc == null){
                    throw new ClientException(ClientException.ErrorType.GENERAL,
                            "Load balancer does not have available server for client: "
                                    + clientName);
                }
                host = svc.getHost();
                if (host == null){
                    throw new ClientException(ClientException.ErrorType.GENERAL,
                            "Invalid Server for :" + svc);
                }
                logger.debug("{} using LB returned Server: {} for request {}", new Object[]{clientName, svc, original});
                return svc;
            } else {
                // No Full URL - and we dont have a LoadBalancer registered to
                // obtain a server
                // if we have a vipAddress that came with the registration, we
                // can use that else we
                // bail out
                if (vipAddresses != null && vipAddresses.contains(",")) {
                    throw new ClientException(
                            ClientException.ErrorType.GENERAL,
                            "Method is invoked for client " + clientName + " with partial URI of ("
                            + original
                            + ") with no load balancer configured."
                            + " Also, there are multiple vipAddresses and hence no vip address can be chosen"
                            + " to complete this partial uri");
                } else if (vipAddresses != null) {
                    try {
                        Pair<String,Integer> hostAndPort = deriveHostAndPortFromVipAddress(vipAddresses);
                        host = hostAndPort.first();
                        port = hostAndPort.second();
                    } catch (URISyntaxException e) {
                        throw new ClientException(
                                ClientException.ErrorType.GENERAL,
                                "Method is invoked for client " + clientName + " with partial URI of ("
                                + original
                                + ") with no load balancer configured. "
                                + " Also, the configured/registered vipAddress is unparseable (to determine host and port)");
                    }
                } else {
                    throw new ClientException(
                            ClientException.ErrorType.GENERAL,
                            this.clientName
                            + " has no LoadBalancer registered and passed in a partial URL request (with no host:port)."
                            + " Also has no vipAddress registered");
                }
            }
        } else {
            // Full URL Case
            // This could either be a vipAddress or a hostAndPort or a real DNS
            // if vipAddress or hostAndPort, we just have to consult the loadbalancer
            // but if it does not return a server, we should just proceed anyways
            // and assume its a DNS
            // For restClients registered using a vipAddress AND executing a request
            // by passing in the full URL (including host and port), we should only
            // consult lb IFF the URL passed is registered as vipAddress in Discovery
            boolean shouldInterpretAsVip = false;

            if (lb != null) {
                shouldInterpretAsVip = isVipRecognized(original.getAuthority());
            }
            if (shouldInterpretAsVip) {
                Server svc = lb.chooseServer(loadBalancerKey);
                if (svc != null){
                    host = svc.getHost();
                    if (host == null){
                        throw new ClientException(ClientException.ErrorType.GENERAL,
                                "Invalid Server for :" + svc);
                    }
                    logger.debug("using LB returned Server: {} for request: {}", svc, original);
                    return svc;
                } else {
                    // just fall back as real DNS
                    logger.debug("{}:{} assumed to be a valid VIP address or exists in the DNS", host, port);
                }
            } else {
                // consult LB to obtain vipAddress backed instance given full URL
                //Full URL execute request - where url!=vipAddress
                logger.debug("Using full URL passed in by caller (not using load balancer): {}", original);
            }
        }
        // end of creating final URL
        if (host == null){
            throw new ClientException(ClientException.ErrorType.GENERAL,"Request contains no HOST to talk to");
        }
        // just verify that at this point we have a full URL

        return new Server(host, port);
    }

    public URI reconstructURIWithServer(Server server, URI original) {
        String host = server.getHost();
        int port = server .getPort();
        if (host.equals(original.getHost()) 
                && port == original.getPort()) {
            return original;
        }
        String scheme = original.getScheme();
        if (scheme == null) {
            scheme = deriveSchemeAndPortFromPartialUri(original).first();
        }

        try {
            StringBuilder sb = new StringBuilder();
            sb.append(scheme).append("://");
            if (!Strings.isNullOrEmpty(original.getRawUserInfo())) {
                sb.append(original.getRawUserInfo()).append("@");
            }
            sb.append(host);
            if (port >= 0) {
                sb.append(":").append(port);
            }
            sb.append(original.getRawPath());
            if (!Strings.isNullOrEmpty(original.getRawQuery())) {
                sb.append("?").append(original.getRawQuery());
            }
            if (!Strings.isNullOrEmpty(original.getRawFragment())) {
                sb.append("#").append(original.getRawFragment());
            }
            URI newURI = new URI(sb.toString());
            return newURI;            
        } catch (URISyntaxException e) {
            throw new RuntimeException(e);
        }
    }

    /*
    protected boolean isRetriable(T request) {
        if (request.isRetriable()) {
            return true;            
        } else {
            boolean retryOkayOnOperation = okToRetryOnAllOperations;
            IClientConfig overriddenClientConfig = request.getOverrideConfig();
            if (overriddenClientConfig != null) {
                retryOkayOnOperation = overriddenClientConfig.getPropertyAsBoolean(CommonClientConfigKey.RequestSpecificRetryOn, okToRetryOnAllOperations);
            }
            return retryOkayOnOperation;
        }
    }
     */

    protected int getRetriesNextServer(IClientConfig overriddenClientConfig) {
        int numRetries = maxAutoRetriesNextServer;
        if (overriddenClientConfig != null) {
            numRetries = overriddenClientConfig.getPropertyAsInteger(CommonClientConfigKey.MaxAutoRetriesNextServer, maxAutoRetriesNextServer);
        }
        return numRetries;
    }

    public final ServerStats getServerStats(Server server) {
        ServerStats serverStats = null;
        ILoadBalancer lb = this.getLoadBalancer();
        if (lb instanceof AbstractLoadBalancer){
            LoadBalancerStats lbStats = ((AbstractLoadBalancer) lb).getLoadBalancerStats();
            serverStats = lbStats.getSingleServerStat(server);
        }
        return serverStats;

    }

    protected int getNumberRetriesOnSameServer(IClientConfig overriddenClientConfig) {
        int numRetries =  maxAutoRetries;
        if (overriddenClientConfig!=null){
            try {
                numRetries = overriddenClientConfig.getPropertyAsInteger(CommonClientConfigKey.MaxAutoRetries, maxAutoRetries);
            } catch (Exception e) {
                logger.warn("Invalid maxRetries requested for RestClient:" + this.clientName);
            }
        }
        return numRetries;
    }

    public boolean handleSameServerRetry(Server server, int currentRetryCount, int maxRetries, Throwable e) {
        if (currentRetryCount > maxRetries) {
            return false;
        }
        logger.debug("Exception while executing request which is deemed retry-able, retrying ..., SAME Server Retry Attempt#: {}",  
                currentRetryCount, server);
        return true;
    }

    public final RetryHandler getRetryHandler() {
        return defaultRetryHandler;
    }

    public final void setRetryHandler(RetryHandler retryHandler) {
        this.defaultRetryHandler = retryHandler;
    }

    public final boolean isOkToRetryOnAllOperations() {
        return okToRetryOnAllOperations;
    }

    public final void setOkToRetryOnAllOperations(boolean okToRetryOnAllOperations) {
        this.okToRetryOnAllOperations = okToRetryOnAllOperations;
    }
}

```

### AbstractLoadBalancerAwareClient.java

```java
package com.netflix.client;


/**
 * Abstract class that provides the integration of client with load balancers.
 */
public abstract class AbstractLoadBalancerAwareClient<S extends ClientRequest, T extends IResponse> extends LoadBalancerContext implements IClient<S, T>, IClientConfigAware {
    
    public AbstractLoadBalancerAwareClient(ILoadBalancer lb) {
        super(lb);
    }

    public AbstractLoadBalancerAwareClient(ILoadBalancer lb, IClientConfig clientConfig) {
        super(lb, clientConfig);        
    }

    @Deprecated
    protected boolean isCircuitBreakerException(Throwable e) {
        if (getRetryHandler() != null) {
            return getRetryHandler().isCircuitTrippingException(e);
        }
        return false;
    }

    @Deprecated
    protected boolean isRetriableException(Throwable e) {
        if (getRetryHandler() != null) {
            return getRetryHandler().isRetriableException(e, true);
        } 
        return false;
    }
    
    public T executeWithLoadBalancer(S request) throws ClientException {
        return executeWithLoadBalancer(request, null);
    }

    public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {
        RequestSpecificRetryHandler handler = getRequestSpecificRetryHandler(request, requestConfig);
        LoadBalancerCommand<T> command = LoadBalancerCommand.<T>builder()
                .withLoadBalancerContext(this)
                .withRetryHandler(handler)
                .withLoadBalancerURI(request.getUri())
                .build();

        try {
            return command.submit(
                new ServerOperation<T>() {
                    @Override
                    public Observable<T> call(Server server) {
                        URI finalUri = reconstructURIWithServer(server, request.getUri());
                        S requestForServer = (S) request.replaceUri(finalUri);
                        try {
                            return Observable.just(AbstractLoadBalancerAwareClient.this.execute(requestForServer, requestConfig));
                        } 
                        catch (Exception e) {
                            return Observable.error(e);
                        }
                    }
                })
                .toBlocking()
                .single();
        } catch (Exception e) {
            Throwable t = e.getCause();
            if (t instanceof ClientException) {
                throw (ClientException) t;
            } else {
                throw new ClientException(e);
            }
        }
        
    }
    
    public abstract RequestSpecificRetryHandler getRequestSpecificRetryHandler(S request, IClientConfig requestConfig);

    @Deprecated
    protected boolean isRetriable(S request) {
        if (request.isRetriable()) {
            return true;            
        } else {
            boolean retryOkayOnOperation = okToRetryOnAllOperations;
            IClientConfig overriddenClientConfig = request.getOverrideConfig();
            if (overriddenClientConfig != null) {
                retryOkayOnOperation = overriddenClientConfig.getPropertyAsBoolean(CommonClientConfigKey.RequestSpecificRetryOn, okToRetryOnAllOperations);
            }
            return retryOkayOnOperation;
        }
    }
    
}



```



## ClientFactory.java

```java
package com.netflix.client;
/**
 * A factory that creates client, load balancer and client configuration instances from properties. It also keeps mappings of client names to 
 * the  created instances.
 *
 */
public class ClientFactory {
    
    private static Map<String, IClient<?,?>> simpleClientMap = new ConcurrentHashMap<String, IClient<?,?>>();
    private static Map<String, ILoadBalancer> namedLBMap = new ConcurrentHashMap<String, ILoadBalancer>();
    private static ConcurrentHashMap<String, IClientConfig> namedConfig = new ConcurrentHashMap<String, IClientConfig>();
    
    private static Logger logger = LoggerFactory.getLogger(ClientFactory.class);

    public static synchronized IClient<?, ?> registerClientFromProperties(String restClientName, IClientConfig clientConfig) throws ClientException { 
    	IClient<?, ?> client = null;
    	ILoadBalancer loadBalancer = null;
    	if (simpleClientMap.get(restClientName) != null) {
    		throw new ClientException(
    				ClientException.ErrorType.GENERAL,
    				"A Rest Client with this name is already registered. Please use a different name");
    	}
    	try {
    		String clientClassName = (String) clientConfig.getProperty(CommonClientConfigKey.ClientClassName);
    		client = (IClient<?, ?>) instantiateInstanceWithClientConfig(clientClassName, clientConfig);
    		boolean initializeNFLoadBalancer = Boolean.parseBoolean(clientConfig.getProperty(
    				CommonClientConfigKey.InitializeNFLoadBalancer, DefaultClientConfigImpl.DEFAULT_ENABLE_LOADBALANCER).toString());
    		if (initializeNFLoadBalancer) {
    			loadBalancer  = registerNamedLoadBalancerFromclientConfig(restClientName, clientConfig);
    		}
    		if (client instanceof AbstractLoadBalancerAwareClient) {
    			((AbstractLoadBalancerAwareClient) client).setLoadBalancer(loadBalancer);
    		}
    	} catch (Throwable e) {
    		String message = "Unable to InitializeAndAssociateNFLoadBalancer set for RestClient:"
    				+ restClientName;
    		logger.warn(message, e);
    		throw new ClientException(ClientException.ErrorType.CONFIGURATION, 
    				message, e);
    	}
    	simpleClientMap.put(restClientName, client);

    	Monitors.registerObject("Client_" + restClientName, client);

    	logger.info("Client Registered:" + client.toString());
    	return client;
    }

    public static synchronized IClient getNamedClient(String name) {
        return getNamedClient(name, DefaultClientConfigImpl.class);
    }

    public static synchronized IClient getNamedClient(String name, Class<? extends IClientConfig> configClass) {
    	if (simpleClientMap.get(name) != null) {
    	    return simpleClientMap.get(name);
    	}
        try {
            return createNamedClient(name, configClass);
        } catch (ClientException e) {
            throw new RuntimeException("Unable to create client", e);
        }
    }

    public static synchronized IClient createNamedClient(String name, Class<? extends IClientConfig> configClass) throws ClientException {
    	IClientConfig config = getNamedConfig(name, configClass);
        return registerClientFromProperties(name, config);
    }

    public static synchronized ILoadBalancer getNamedLoadBalancer(String name) {
    	return getNamedLoadBalancer(name, DefaultClientConfigImpl.class);
    }

    public static synchronized ILoadBalancer getNamedLoadBalancer(String name, Class<? extends IClientConfig> configClass) {
        ILoadBalancer lb = namedLBMap.get(name);
        if (lb != null) {
            return lb;
        } else {
            try {
                lb = registerNamedLoadBalancerFromProperties(name, configClass);
            } catch (ClientException e) {
                throw new RuntimeException("Unable to create load balancer", e);
            }
            return lb;
        }
    }

    public static ILoadBalancer registerNamedLoadBalancerFromclientConfig(String name, IClientConfig clientConfig) throws ClientException {
        if (namedLBMap.get(name) != null) {
            throw new ClientException("LoadBalancer for name " + name + " already exists");
        }
    	ILoadBalancer lb = null;
        try {
            String loadBalancerClassName = (String) clientConfig.getProperty(CommonClientConfigKey.NFLoadBalancerClassName);
            lb = (ILoadBalancer) ClientFactory.instantiateInstanceWithClientConfig(loadBalancerClassName, clientConfig);                                    
            namedLBMap.put(name, lb);            
            logger.info("Client:" + name
                    + " instantiated a LoadBalancer:" + lb.toString());
            return lb;
        } catch (Exception e) {           
           throw new ClientException("Unable to instantiate/associate LoadBalancer with Client:" + name, e);
        }    	
    }

    public static synchronized ILoadBalancer registerNamedLoadBalancerFromProperties(String name, Class<? extends IClientConfig> configClass) throws ClientException {
        if (namedLBMap.get(name) != null) {
            throw new ClientException("LoadBalancer for name " + name + " already exists");
        }
        IClientConfig clientConfig = getNamedConfig(name, configClass);
        return registerNamedLoadBalancerFromclientConfig(name, clientConfig);
    }    

    @SuppressWarnings("unchecked")
	public static Object instantiateInstanceWithClientConfig(String className, IClientConfig clientConfig) 
    		throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    	Class clazz = Class.forName(className);
    	if (IClientConfigAware.class.isAssignableFrom(clazz)) {
    		IClientConfigAware obj = (IClientConfigAware) clazz.newInstance();
    		obj.initWithNiwsConfig(clientConfig);
    		return obj;
    	} else {
    		try {
    		    if (clazz.getConstructor(IClientConfig.class) != null) {
    		    	return clazz.getConstructor(IClientConfig.class).newInstance(clientConfig);
    		    }
    		} catch (Throwable e) { // NOPMD   			
    		}    		
    	}
    	logger.warn("Class " + className + " neither implements IClientConfigAware nor provides a constructor with IClientConfig as the parameter. Only default constructor will be used.");
    	return clazz.newInstance();
    }

    public static IClientConfig getNamedConfig(String name) {
        return 	getNamedConfig(name, DefaultClientConfigImpl.class);
    }

    public static IClientConfig getNamedConfig(String name, Class<? extends IClientConfig> clientConfigClass) {
    	IClientConfig config = namedConfig.get(name);
        if (config != null) {
            return config;
        } else {
        	try {
                config = (IClientConfig) clientConfigClass.newInstance();
                config.loadProperties(name);//??????????????????????????????????????????????????????????百思不得姐
        	} catch (Throwable e) {
        		logger.error("Unable to create client config instance", e);
        		return null;
        	}
            config.loadProperties(name);//??????????????????????????????????????????????????????????百思不得姐
            IClientConfig old = namedConfig.putIfAbsent(name, config);
            if (old != null) {
                config = old;
            }
            return config;
        }
    }
}

```



# end