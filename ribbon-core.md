# Core

## IClientConfigKey.java

```java
package com.netflix.client.config;

public interface IClientConfigKey<T> {

    @SuppressWarnings("rawtypes")
    public static final class Keys extends CommonClientConfigKey {
        private Keys(String configKey) {
            super(configKey);
        }
    }

	public String key();

	public Class<T> type();
}

```

### CommonClientConfigKey.java

```java
package com.netflix.client.config;

public abstract class CommonClientConfigKey<T> implements IClientConfigKey<T> {

    public static final IClientConfigKey<String> AppName = new CommonClientConfigKey<String>("AppName"){};
    
    public static final IClientConfigKey<String> Version = new CommonClientConfigKey<String>("Version"){};
        
    public static final IClientConfigKey<Integer> Port = new CommonClientConfigKey<Integer>("Port"){};
    
    public static final IClientConfigKey<Integer> SecurePort = new CommonClientConfigKey<Integer>("SecurePort"){};
    
    public static final IClientConfigKey<String> VipAddress = new CommonClientConfigKey<String>("VipAddress"){};
    
    public static final IClientConfigKey<Boolean> ForceClientPortConfiguration = new CommonClientConfigKey<Boolean>("ForceClientPortConfiguration"){}; // use client defined port regardless of server advert

    public static final IClientConfigKey<String> DeploymentContextBasedVipAddresses = new CommonClientConfigKey<String>("DeploymentContextBasedVipAddresses"){};
    
    public static final IClientConfigKey<Integer> MaxAutoRetries = new CommonClientConfigKey<Integer>("MaxAutoRetries"){};
    
    public static final IClientConfigKey<Integer> MaxAutoRetriesNextServer = new CommonClientConfigKey<Integer>("MaxAutoRetriesNextServer"){};
    
    public static final IClientConfigKey<Boolean> OkToRetryOnAllOperations = new CommonClientConfigKey<Boolean>("OkToRetryOnAllOperations"){};
    
    public static final IClientConfigKey<Boolean> RequestSpecificRetryOn = new CommonClientConfigKey<Boolean>("RequestSpecificRetryOn"){};
    
    public static final IClientConfigKey<Integer> ReceiveBufferSize = new CommonClientConfigKey<Integer>("ReceiveBufferSize"){};
    
    public static final IClientConfigKey<Boolean> EnablePrimeConnections = new CommonClientConfigKey<Boolean>("EnablePrimeConnections"){};
    
    public static final IClientConfigKey<String> PrimeConnectionsClassName = new CommonClientConfigKey<String>("PrimeConnectionsClassName"){};
    
    public static final IClientConfigKey<Integer> MaxRetriesPerServerPrimeConnection = new CommonClientConfigKey<Integer>("MaxRetriesPerServerPrimeConnection"){};
    
    public static final IClientConfigKey<Integer> MaxTotalTimeToPrimeConnections = new CommonClientConfigKey<Integer>("MaxTotalTimeToPrimeConnections"){};
    
    public static final IClientConfigKey<Float> MinPrimeConnectionsRatio = new CommonClientConfigKey<Float>("MinPrimeConnectionsRatio"){};
    
    public static final IClientConfigKey<String> PrimeConnectionsURI = new CommonClientConfigKey<String>("PrimeConnectionsURI"){};
    
    public static final IClientConfigKey<Integer> PoolMaxThreads = new CommonClientConfigKey<Integer>("PoolMaxThreads"){};
    
    public static final IClientConfigKey<Integer> PoolMinThreads = new CommonClientConfigKey<Integer>("PoolMinThreads"){};
    
    public static final IClientConfigKey<Integer> PoolKeepAliveTime = new CommonClientConfigKey<Integer>("PoolKeepAliveTime"){};
    
    public static final IClientConfigKey<String> PoolKeepAliveTimeUnits = new CommonClientConfigKey<String>("PoolKeepAliveTimeUnits"){};

    public static final IClientConfigKey<Boolean> EnableConnectionPool = new CommonClientConfigKey<Boolean>("EnableConnectionPool") {};

    @Deprecated    
    public static final IClientConfigKey<Integer> MaxHttpConnectionsPerHost = new CommonClientConfigKey<Integer>("MaxHttpConnectionsPerHost"){};

    @Deprecated
    public static final IClientConfigKey<Integer> MaxTotalHttpConnections = new CommonClientConfigKey<Integer>("MaxTotalHttpConnections"){};
    
    public static final IClientConfigKey<Integer> MaxConnectionsPerHost = new CommonClientConfigKey<Integer>("MaxConnectionsPerHost"){};
    
    public static final IClientConfigKey<Integer> MaxTotalConnections = new CommonClientConfigKey<Integer>("MaxTotalConnections"){};
    
    public static final IClientConfigKey<Boolean> IsSecure = new CommonClientConfigKey<Boolean>("IsSecure"){};
    
    public static final IClientConfigKey<Boolean> GZipPayload = new CommonClientConfigKey<Boolean>("GZipPayload"){};
    
    public static final IClientConfigKey<Integer> ConnectTimeout = new CommonClientConfigKey<Integer>("ConnectTimeout"){};
    
    public static final IClientConfigKey<Integer> BackoffInterval = new CommonClientConfigKey<Integer>("BackoffTimeout"){};
    
    public static final IClientConfigKey<Integer> ReadTimeout = new CommonClientConfigKey<Integer>("ReadTimeout"){};
    
    public static final IClientConfigKey<Integer> SendBufferSize = new CommonClientConfigKey<Integer>("SendBufferSize"){};
    
    public static final IClientConfigKey<Boolean> StaleCheckingEnabled = new CommonClientConfigKey<Boolean>("StaleCheckingEnabled"){};
    
    public static final IClientConfigKey<Integer> Linger = new CommonClientConfigKey<Integer>("Linger"){};
    
    public static final IClientConfigKey<Integer> ConnectionManagerTimeout = new CommonClientConfigKey<Integer>("ConnectionManagerTimeout"){};
    
    public static final IClientConfigKey<Boolean> FollowRedirects = new CommonClientConfigKey<Boolean>("FollowRedirects"){};
    
    public static final IClientConfigKey<Boolean> ConnectionPoolCleanerTaskEnabled = new CommonClientConfigKey<Boolean>("ConnectionPoolCleanerTaskEnabled"){};
    
    public static final IClientConfigKey<Integer> ConnIdleEvictTimeMilliSeconds = new CommonClientConfigKey<Integer>("ConnIdleEvictTimeMilliSeconds"){};
    
    public static final IClientConfigKey<Integer> ConnectionCleanerRepeatInterval = new CommonClientConfigKey<Integer>("ConnectionCleanerRepeatInterval"){};
    
    public static final IClientConfigKey<Boolean> EnableGZIPContentEncodingFilter = new CommonClientConfigKey<Boolean>("EnableGZIPContentEncodingFilter"){};
    
    public static final IClientConfigKey<String> ProxyHost = new CommonClientConfigKey<String>("ProxyHost"){};
    
    public static final IClientConfigKey<Integer> ProxyPort = new CommonClientConfigKey<Integer>("ProxyPort"){};
    
    public static final IClientConfigKey<String> KeyStore = new CommonClientConfigKey<String>("KeyStore"){};
    
    public static final IClientConfigKey<String> KeyStorePassword = new CommonClientConfigKey<String>("KeyStorePassword"){};
    
    public static final IClientConfigKey<String> TrustStore = new CommonClientConfigKey<String>("TrustStore"){};
    
    public static final IClientConfigKey<String> TrustStorePassword = new CommonClientConfigKey<String>("TrustStorePassword"){};
    
    // if this is a secure rest client, must we use client auth too?    
    public static final IClientConfigKey<Boolean> IsClientAuthRequired = new CommonClientConfigKey<Boolean>("IsClientAuthRequired"){};
    
    public static final IClientConfigKey<String> CustomSSLSocketFactoryClassName = new CommonClientConfigKey<String>("CustomSSLSocketFactoryClassName"){};
     // must host name match name in certificate?
    public static final IClientConfigKey<Boolean> IsHostnameValidationRequired = new CommonClientConfigKey<Boolean>("IsHostnameValidationRequired"){}; 

    // see also http://hc.apache.org/httpcomponents-client-ga/tutorial/html/advanced.html
    public static final IClientConfigKey<Boolean> IgnoreUserTokenInConnectionPoolForSecureClient = new CommonClientConfigKey<Boolean>("IgnoreUserTokenInConnectionPoolForSecureClient"){}; 
    
    // Client implementation
    public static final IClientConfigKey<String> ClientClassName = new CommonClientConfigKey<String>("ClientClassName"){};

    //LoadBalancer Related
    public static final IClientConfigKey<Boolean> InitializeNFLoadBalancer = new CommonClientConfigKey<Boolean>("InitializeNFLoadBalancer"){};
    
    public static final IClientConfigKey<String> NFLoadBalancerClassName = new CommonClientConfigKey<String>("NFLoadBalancerClassName"){};
    
    public static final IClientConfigKey<String> NFLoadBalancerRuleClassName = new CommonClientConfigKey<String>("NFLoadBalancerRuleClassName"){};
    
    public static final IClientConfigKey<String> NFLoadBalancerPingClassName = new CommonClientConfigKey<String>("NFLoadBalancerPingClassName"){};
    
    public static final IClientConfigKey<Integer> NFLoadBalancerPingInterval = new CommonClientConfigKey<Integer>("NFLoadBalancerPingInterval"){};
    
    public static final IClientConfigKey<Integer> NFLoadBalancerMaxTotalPingTime = new CommonClientConfigKey<Integer>("NFLoadBalancerMaxTotalPingTime"){};
    
    public static final IClientConfigKey<String> NIWSServerListClassName = new CommonClientConfigKey<String>("NIWSServerListClassName"){};

    public static final IClientConfigKey<String> ServerListUpdaterClassName = new CommonClientConfigKey<String>("ServerListUpdaterClassName"){};
    
    public static final IClientConfigKey<String> NIWSServerListFilterClassName = new CommonClientConfigKey<String>("NIWSServerListFilterClassName"){};
    
    public static final IClientConfigKey<Integer> ServerListRefreshInterval = new CommonClientConfigKey<Integer>("ServerListRefreshInterval"){};
    
    public static final IClientConfigKey<Boolean> EnableMarkingServerDownOnReachingFailureLimit = new CommonClientConfigKey<Boolean>("EnableMarkingServerDownOnReachingFailureLimit"){};
    
    public static final IClientConfigKey<Integer> ServerDownFailureLimit = new CommonClientConfigKey<Integer>("ServerDownFailureLimit"){};
    
    public static final IClientConfigKey<Integer> ServerDownStatWindowInMillis = new CommonClientConfigKey<Integer>("ServerDownStatWindowInMillis"){};
    
    public static final IClientConfigKey<Boolean> EnableZoneAffinity = new CommonClientConfigKey<Boolean>("EnableZoneAffinity"){};
    
    public static final IClientConfigKey<Boolean> EnableZoneExclusivity = new CommonClientConfigKey<Boolean>("EnableZoneExclusivity"){};
    
    public static final IClientConfigKey<Boolean> PrioritizeVipAddressBasedServers = new CommonClientConfigKey<Boolean>("PrioritizeVipAddressBasedServers"){};
    
    public static final IClientConfigKey<String> VipAddressResolverClassName = new CommonClientConfigKey<String>("VipAddressResolverClassName"){};
    
    public static final IClientConfigKey<String> TargetRegion = new CommonClientConfigKey<String>("TargetRegion"){};
    
    public static final IClientConfigKey<String> RulePredicateClasses = new CommonClientConfigKey<String>("RulePredicateClasses"){};
    
    public static final IClientConfigKey<String> RequestIdHeaderName = new CommonClientConfigKey<String>("RequestIdHeaderName") {};
    
    public static final IClientConfigKey<Boolean> UseIPAddrForServer = new CommonClientConfigKey<Boolean>("UseIPAddrForServer") {};
    
    public static final IClientConfigKey<String> ListOfServers = new CommonClientConfigKey<String>("listOfServers") {};

    private static final Set<IClientConfigKey> keys = new HashSet<IClientConfigKey>();
    //类加载时反射初始化keys    
    static {
        for (Field f: CommonClientConfigKey.class.getDeclaredFields()) {
            if (Modifier.isStatic(f.getModifiers()) //&& Modifier.isPublic(f.getModifiers())
                    && IClientConfigKey.class.isAssignableFrom(f.getType())) {
                try {
                    keys.add((IClientConfigKey) f.get(null));
                } catch (IllegalAccessException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }

    /**
     * @deprecated see {@link #keys()} 
     */
    @edu.umd.cs.findbugs.annotations.SuppressWarnings
    @Deprecated
    public static IClientConfigKey[] values() {
       return keys().toArray(new IClientConfigKey[0]);
    }

    /**
     * return all the public static keys defined in this class 
     */
    public static Set<IClientConfigKey> keys() {
        return keys;
    }

    public static IClientConfigKey valueOf(final String name) {
        for (IClientConfigKey key: keys()) {
            if (key.key().equals(name)) {
                return key;
            }
        }
        return new IClientConfigKey() {
            @Override
            public String key() {
                return name;
            }

            @Override
            public Class type() {
                return String.class;
            }
        };
    }
    
    private final String configKey;
    private final Class<T> type;
    //使用泛型类型初始化type
    @SuppressWarnings("unchecked")
    protected CommonClientConfigKey(String configKey) {
        this.configKey = configKey;
        Type superclass = getClass().getGenericSuperclass();
        checkArgument(superclass instanceof ParameterizedType,
            "%s isn't parameterized", superclass);
        Type runtimeType = ((ParameterizedType) superclass).getActualTypeArguments()[0];
        type = (Class<T>) TypeToken.of(runtimeType).getRawType();
    }
    
    @Override
    public Class<T> type() {
        return type;
    }

    @Override
	public String key() {
        return configKey;
    }
    
    @Override
    public String toString() {
        return configKey;
    }
}

```

## IClientConfig.java

```java
package com.netflix.client.config;
/**
 * 客户端配置
 *
 */

public interface IClientConfig {
	
	public String getClientName();
		
	public String getNameSpace();

	public void loadProperties(String clientName);

	public void loadDefaultValues();

	public Map<String, Object> getProperties();

	@Deprecated
	public void setProperty(IClientConfigKey key, Object value);

    @Deprecated
	public Object getProperty(IClientConfigKey key);

    @Deprecated
	public Object getProperty(IClientConfigKey key, Object defaultVal);

	public boolean containsProperty(IClientConfigKey key);

	public String resolveDeploymentContextbasedVipAddresses();
	
	public int getPropertyAsInteger(IClientConfigKey key, int defaultValue);

    public String getPropertyAsString(IClientConfigKey key, String defaultValue);
    
    public boolean getPropertyAsBoolean(IClientConfigKey key, boolean defaultValue);

    public <T> T get(IClientConfigKey<T> key);

    public <T> T get(IClientConfigKey<T> key, T defaultValue);

    public <T> IClientConfig set(IClientConfigKey<T> key, T value);
    
    public static class Builder {
        
        private IClientConfig config;
        
        Builder() {
        }

        public static Builder newBuilder() {
            Builder builder = new Builder();
            builder.config = new DefaultClientConfigImpl();
            return builder;
        }

        public static Builder newBuilder(String clientName) {
            Builder builder = new Builder();
            builder.config = new DefaultClientConfigImpl();
            builder.config.loadProperties(clientName);
            return builder;
        }

        public static Builder newBuilder(String clientName, String propertyNameSpace) {
            Builder builder = new Builder();
            builder.config = new DefaultClientConfigImpl(propertyNameSpace);
            builder.config.loadProperties(clientName);
            return builder;
        }

        public static Builder newBuilder(Class<? extends IClientConfig> implClass, String clientName) {
            Builder builder = new Builder();
            try {
                builder.config = implClass.newInstance();
                builder.config.loadProperties(clientName);
            } catch (Exception e) {
                throw new IllegalArgumentException(e);
            }
            return builder;
        }

        public static Builder newBuilder(Class<? extends IClientConfig> implClass) {
            Builder builder = new Builder();
            try {
                builder.config = implClass.newInstance();
            } catch (Exception e) {
                throw new IllegalArgumentException(e);
            }
            return builder;        
        }
        
        public IClientConfig build() {
            return config;
        }

        public Builder withDefaultValues() {
            config.loadDefaultValues();
            return this;
        }
        
        public Builder withDeploymentContextBasedVipAddresses(String vipAddress) {
            config.set(CommonClientConfigKey.DeploymentContextBasedVipAddresses, vipAddress);
            return this;
        }

        public Builder withForceClientPortConfiguration(boolean forceClientPortConfiguration) {
            config.set(CommonClientConfigKey.ForceClientPortConfiguration, forceClientPortConfiguration);
            return this;
        }

        public Builder withMaxAutoRetries(int value) {
            config.set(CommonClientConfigKey.MaxAutoRetries, value);
            return this;
        }

        public Builder withMaxAutoRetriesNextServer(int value) {
            config.set(CommonClientConfigKey.MaxAutoRetriesNextServer, value);
            return this;
        }

        public Builder withRetryOnAllOperations(boolean value) {
            config.set(CommonClientConfigKey.OkToRetryOnAllOperations, value);
            return this;
        }

        public Builder withRequestSpecificRetryOn(boolean value) {
            config.set(CommonClientConfigKey.RequestSpecificRetryOn, value);
            return this;
        }
            
        public Builder withEnablePrimeConnections(boolean value) {
            config.set(CommonClientConfigKey.EnablePrimeConnections, value);
            return this;
        }

        public Builder withMaxConnectionsPerHost(int value) {
            config.set(CommonClientConfigKey.MaxHttpConnectionsPerHost, value);
            config.set(CommonClientConfigKey.MaxConnectionsPerHost, value);
            return this;
        }

        public Builder withMaxTotalConnections(int value) {
            config.set(CommonClientConfigKey.MaxTotalHttpConnections, value);
            config.set(CommonClientConfigKey.MaxTotalConnections, value);
            return this;
        }
        
        public Builder withSecure(boolean secure) {
            config.set(CommonClientConfigKey.IsSecure, secure);
            return this;
        }

        public Builder withConnectTimeout(int value) {
            config.set(CommonClientConfigKey.ConnectTimeout, value);
            return this;
        }

        public Builder withReadTimeout(int value) {
            config.set(CommonClientConfigKey.ReadTimeout, value);
            return this;
        }

        public Builder withConnectionManagerTimeout(int value) {
            config.set(CommonClientConfigKey.ConnectionManagerTimeout, value);
            return this;
        }
        
        public Builder withFollowRedirects(boolean value) {
            config.set(CommonClientConfigKey.FollowRedirects, value);
            return this;
        }
        
        public Builder withConnectionPoolCleanerTaskEnabled(boolean value) {
            config.set(CommonClientConfigKey.ConnectionPoolCleanerTaskEnabled, value);
            return this;
        }
            
        public Builder withConnIdleEvictTimeMilliSeconds(int value) {
            config.set(CommonClientConfigKey.ConnIdleEvictTimeMilliSeconds, value);
            return this;
        }
        
        public Builder withConnectionCleanerRepeatIntervalMills(int value) {
            config.set(CommonClientConfigKey.ConnectionCleanerRepeatInterval, value);
            return this;
        }
        
        public Builder withGZIPContentEncodingFilterEnabled(boolean value) {
            config.set(CommonClientConfigKey.EnableGZIPContentEncodingFilter, value);
            return this;
        }

        public Builder withProxyHost(String proxyHost) {
            config.set(CommonClientConfigKey.ProxyHost, proxyHost);
            return this;
        }

        public Builder withProxyPort(int value) {
            config.set(CommonClientConfigKey.ProxyPort, value);
            return this;
        }

        public Builder withKeyStore(String value) {
            config.set(CommonClientConfigKey.KeyStore, value);
            return this;
        }

        public Builder withKeyStorePassword(String value) {
            config.set(CommonClientConfigKey.KeyStorePassword, value);
            return this;
        }
        
        public Builder withTrustStore(String value) {
            config.set(CommonClientConfigKey.TrustStore, value);
            return this;
        }

        public Builder withTrustStorePassword(String value) {
            config.set(CommonClientConfigKey.TrustStorePassword, value);
            return this;
        }
        
        public Builder withClientAuthRequired(boolean value) {
            config.set(CommonClientConfigKey.IsClientAuthRequired, value);
            return this;
        }
        
        public Builder withCustomSSLSocketFactoryClassName(String value) {
            config.set(CommonClientConfigKey.CustomSSLSocketFactoryClassName, value);
            return this;
        }
        
        public Builder withHostnameValidationRequired(boolean value) {
            config.set(CommonClientConfigKey.IsHostnameValidationRequired, value);
            return this;
        }

        // see also http://hc.apache.org/httpcomponents-client-ga/tutorial/html/advanced.html
        public Builder ignoreUserTokenInConnectionPoolForSecureClient(boolean value) {
            config.set(CommonClientConfigKey.IgnoreUserTokenInConnectionPoolForSecureClient, value);
            return this;
        }

        public Builder withLoadBalancerEnabled(boolean value) {
            config.set(CommonClientConfigKey.InitializeNFLoadBalancer, value);
            return this;
        }
        
        public Builder withServerListRefreshIntervalMills(int value) {
            config.set(CommonClientConfigKey.ServerListRefreshInterval, value);
            return this;
        }
        
        public Builder withZoneAffinityEnabled(boolean value) {
            config.set(CommonClientConfigKey.EnableZoneAffinity, value);
            return this;
        }
        
        public Builder withZoneExclusivityEnabled(boolean value) {
            config.set(CommonClientConfigKey.EnableZoneExclusivity, value);
            return this;
        }

        public Builder prioritizeVipAddressBasedServers(boolean value) {
            config.set(CommonClientConfigKey.PrioritizeVipAddressBasedServers, value);
            return this;
        }
        
        public Builder withTargetRegion(String value) {
            config.set(CommonClientConfigKey.TargetRegion, value);
            return this;
        }
    }

}

```

### DefaultClientConfigImpl.java

```java
package com.netflix.client.config;
/**
 * 默认的客户端配置
 */
public class DefaultClientConfigImpl implements IClientConfig {

    public static final Boolean DEFAULT_PRIORITIZE_VIP_ADDRESS_BASED_SERVERS = Boolean.TRUE;

	public static final String DEFAULT_NFLOADBALANCER_PING_CLASSNAME = "com.netflix.loadbalancer.DummyPing"; // DummyPing.class.getName();

    public static final String DEFAULT_NFLOADBALANCER_RULE_CLASSNAME = "com.netflix.loadbalancer.AvailabilityFilteringRule";

    public static final String DEFAULT_NFLOADBALANCER_CLASSNAME = "com.netflix.loadbalancer.ZoneAwareLoadBalancer";
    
    public static final boolean DEFAULT_USEIPADDRESS_FOR_SERVER = Boolean.FALSE;

    public static final String DEFAULT_CLIENT_CLASSNAME = "com.netflix.niws.client.http.RestClient";

    public static final String DEFAULT_VIPADDRESS_RESOLVER_CLASSNAME = "com.netflix.client.SimpleVipAddressResolver";

    public static final String DEFAULT_PRIME_CONNECTIONS_URI = "/";

    public static final int DEFAULT_MAX_TOTAL_TIME_TO_PRIME_CONNECTIONS = 30000;

    public static final int DEFAULT_MAX_RETRIES_PER_SERVER_PRIME_CONNECTION = 9;

    public static final Boolean DEFAULT_ENABLE_PRIME_CONNECTIONS = Boolean.FALSE;

    public static final int DEFAULT_MAX_REQUESTS_ALLOWED_PER_WINDOW = Integer.MAX_VALUE;

    public static final int DEFAULT_REQUEST_THROTTLING_WINDOW_IN_MILLIS = 60000;

    public static final Boolean DEFAULT_ENABLE_REQUEST_THROTTLING = Boolean.FALSE;

    public static final Boolean DEFAULT_ENABLE_GZIP_CONTENT_ENCODING_FILTER = Boolean.FALSE;

    public static final Boolean DEFAULT_CONNECTION_POOL_CLEANER_TASK_ENABLED = Boolean.TRUE;

    public static final Boolean DEFAULT_FOLLOW_REDIRECTS = Boolean.FALSE;

    public static final float DEFAULT_PERCENTAGE_NIWS_EVENT_LOGGED = 0.0f;

    public static final int DEFAULT_MAX_AUTO_RETRIES_NEXT_SERVER = 1;

    public static final int DEFAULT_MAX_AUTO_RETRIES = 0;

    public static final int DEFAULT_BACKOFF_INTERVAL = 0;
    
    public static final int DEFAULT_READ_TIMEOUT = 5000;

    public static final int DEFAULT_CONNECTION_MANAGER_TIMEOUT = 2000;

    public static final int DEFAULT_CONNECT_TIMEOUT = 2000;

    public static final Boolean DEFAULT_ENABLE_CONNECTION_POOL = Boolean.TRUE;
    
    @Deprecated
    public static final int DEFAULT_MAX_HTTP_CONNECTIONS_PER_HOST = 50;

    @Deprecated
    public static final int DEFAULT_MAX_TOTAL_HTTP_CONNECTIONS = 200;

    public static final int DEFAULT_MAX_CONNECTIONS_PER_HOST = 50;

    public static final int DEFAULT_MAX_TOTAL_CONNECTIONS = 200;

    public static final float DEFAULT_MIN_PRIME_CONNECTIONS_RATIO = 1.0f;

    public static final String DEFAULT_PRIME_CONNECTIONS_CLASS = "com.netflix.niws.client.http.HttpPrimeConnection";

    public static final String DEFAULT_SEVER_LIST_CLASS = "com.netflix.loadbalancer.ConfigurationBasedServerList";

    public static final String DEFAULT_SERVER_LIST_UPDATER_CLASS = "com.netflix.loadbalancer.PollingServerListUpdater";

    public static final int DEFAULT_CONNECTION_IDLE_TIMERTASK_REPEAT_IN_MSECS = 30000; // every half minute (30 secs)

    public static final int DEFAULT_CONNECTIONIDLE_TIME_IN_MSECS = 30000; // all connections idle for 30 secs
    
    protected volatile Map<String, Object> properties = new ConcurrentHashMap<String, Object>();
    
    protected Map<IClientConfigKey<?>, Object> typedProperties = new ConcurrentHashMap<IClientConfigKey<?>, Object>();

    private static final Logger LOG = LoggerFactory.getLogger(DefaultClientConfigImpl.class);

    private String clientName = null;

    private VipAddressResolver resolver = null;

    private boolean enableDynamicProperties = true;
    /**
     * Defaults for the parameters for the thread pool used by batchParallel
     * calls
     */
    public static final int DEFAULT_POOL_MAX_THREADS = DEFAULT_MAX_TOTAL_HTTP_CONNECTIONS;
    public static final int DEFAULT_POOL_MIN_THREADS = 1;
    public static final long DEFAULT_POOL_KEEP_ALIVE_TIME = 15 * 60L;
    public static final TimeUnit DEFAULT_POOL_KEEP_ALIVE_TIME_UNITS = TimeUnit.SECONDS;
    public static final Boolean DEFAULT_ENABLE_ZONE_AFFINITY = Boolean.FALSE;
    public static final Boolean DEFAULT_ENABLE_ZONE_EXCLUSIVITY = Boolean.FALSE;
    public static final int DEFAULT_PORT = 7001;
    public static final Boolean DEFAULT_ENABLE_LOADBALANCER = Boolean.TRUE;

    public static final String DEFAULT_PROPERTY_NAME_SPACE = "ribbon";

    private String propertyNameSpace = DEFAULT_PROPERTY_NAME_SPACE;

    public static final Boolean DEFAULT_OK_TO_RETRY_ON_ALL_OPERATIONS = Boolean.FALSE;

    public static final Boolean DEFAULT_ENABLE_NIWS_EVENT_LOGGING = Boolean.TRUE;

    public static final Boolean DEFAULT_IS_CLIENT_AUTH_REQUIRED = Boolean.FALSE;

    private final Map<String, DynamicStringProperty> dynamicProperties = new ConcurrentHashMap<String, DynamicStringProperty>();

    public Boolean getDefaultPrioritizeVipAddressBasedServers() {
		return DEFAULT_PRIORITIZE_VIP_ADDRESS_BASED_SERVERS;
	}

	public String getDefaultNfloadbalancerPingClassname() {
		return DEFAULT_NFLOADBALANCER_PING_CLASSNAME;
	}

	public String getDefaultNfloadbalancerRuleClassname() {
		return DEFAULT_NFLOADBALANCER_RULE_CLASSNAME;
	}

	public String getDefaultNfloadbalancerClassname() {
		return DEFAULT_NFLOADBALANCER_CLASSNAME;
	}
	
	public boolean getDefaultUseIpAddressForServer() {
		return DEFAULT_USEIPADDRESS_FOR_SERVER;
	}

	public String getDefaultClientClassname() {
		return DEFAULT_CLIENT_CLASSNAME;
	}

	public String getDefaultVipaddressResolverClassname() {
		return DEFAULT_VIPADDRESS_RESOLVER_CLASSNAME;
	}

	public String getDefaultPrimeConnectionsUri() {
		return DEFAULT_PRIME_CONNECTIONS_URI;
	}

	public int getDefaultMaxTotalTimeToPrimeConnections() {
		return DEFAULT_MAX_TOTAL_TIME_TO_PRIME_CONNECTIONS;
	}

	public int getDefaultMaxRetriesPerServerPrimeConnection() {
		return DEFAULT_MAX_RETRIES_PER_SERVER_PRIME_CONNECTION;
	}

	public Boolean getDefaultEnablePrimeConnections() {
		return DEFAULT_ENABLE_PRIME_CONNECTIONS;
	}

	public int getDefaultMaxRequestsAllowedPerWindow() {
		return DEFAULT_MAX_REQUESTS_ALLOWED_PER_WINDOW;
	}

	public int getDefaultRequestThrottlingWindowInMillis() {
		return DEFAULT_REQUEST_THROTTLING_WINDOW_IN_MILLIS;
	}

	public Boolean getDefaultEnableRequestThrottling() {
		return DEFAULT_ENABLE_REQUEST_THROTTLING;
	}

	public Boolean getDefaultEnableGzipContentEncodingFilter() {
		return DEFAULT_ENABLE_GZIP_CONTENT_ENCODING_FILTER;
	}

	public Boolean getDefaultConnectionPoolCleanerTaskEnabled() {
		return DEFAULT_CONNECTION_POOL_CLEANER_TASK_ENABLED;
	}

	public Boolean getDefaultFollowRedirects() {
		return DEFAULT_FOLLOW_REDIRECTS;
	}

	public float getDefaultPercentageNiwsEventLogged() {
		return DEFAULT_PERCENTAGE_NIWS_EVENT_LOGGED;
	}

	public int getDefaultMaxAutoRetriesNextServer() {
		return DEFAULT_MAX_AUTO_RETRIES_NEXT_SERVER;
	}

	public int getDefaultMaxAutoRetries() {
		return DEFAULT_MAX_AUTO_RETRIES;
	}

	public int getDefaultReadTimeout() {
		return DEFAULT_READ_TIMEOUT;
	}

	public int getDefaultConnectionManagerTimeout() {
		return DEFAULT_CONNECTION_MANAGER_TIMEOUT;
	}

	public int getDefaultConnectTimeout() {
		return DEFAULT_CONNECT_TIMEOUT;
	}

	@Deprecated
	public int getDefaultMaxHttpConnectionsPerHost() {
		return DEFAULT_MAX_HTTP_CONNECTIONS_PER_HOST;
	}

	@Deprecated
	public int getDefaultMaxTotalHttpConnections() {
		return DEFAULT_MAX_TOTAL_HTTP_CONNECTIONS;
	}

	public int getDefaultMaxConnectionsPerHost() {
	    return DEFAULT_MAX_CONNECTIONS_PER_HOST;
	}

	public int getDefaultMaxTotalConnections() {
	    return DEFAULT_MAX_TOTAL_CONNECTIONS;
	}
	
	public float getDefaultMinPrimeConnectionsRatio() {
		return DEFAULT_MIN_PRIME_CONNECTIONS_RATIO;
	}

	public String getDefaultPrimeConnectionsClass() {
		return DEFAULT_PRIME_CONNECTIONS_CLASS;
	}

	public String getDefaultSeverListClass() {
		return DEFAULT_SEVER_LIST_CLASS;
	}

	public int getDefaultConnectionIdleTimertaskRepeatInMsecs() {
		return DEFAULT_CONNECTION_IDLE_TIMERTASK_REPEAT_IN_MSECS;
	}

	public int getDefaultConnectionidleTimeInMsecs() {
		return DEFAULT_CONNECTIONIDLE_TIME_IN_MSECS;
	}
	
	public VipAddressResolver getResolver() {
		return resolver;
	}

	public boolean isEnableDynamicProperties() {
		return enableDynamicProperties;
	}

	public int getDefaultPoolMaxThreads() {
		return DEFAULT_POOL_MAX_THREADS;
	}

	public int getDefaultPoolMinThreads() {
		return DEFAULT_POOL_MIN_THREADS;
	}

	public long getDefaultPoolKeepAliveTime() {
		return DEFAULT_POOL_KEEP_ALIVE_TIME;
	}

	public TimeUnit getDefaultPoolKeepAliveTimeUnits() {
		return DEFAULT_POOL_KEEP_ALIVE_TIME_UNITS;
	}

	public Boolean getDefaultEnableZoneAffinity() {
		return DEFAULT_ENABLE_ZONE_AFFINITY;
	}

	public Boolean getDefaultEnableZoneExclusivity() {
		return DEFAULT_ENABLE_ZONE_EXCLUSIVITY;
	}

	public int getDefaultPort() {
		return DEFAULT_PORT;
	}

	public Boolean getDefaultEnableLoadbalancer() {
		return DEFAULT_ENABLE_LOADBALANCER;
	}


	public Boolean getDefaultOkToRetryOnAllOperations() {
		return DEFAULT_OK_TO_RETRY_ON_ALL_OPERATIONS;
	}

	public Boolean getDefaultIsClientAuthRequired(){
		return DEFAULT_IS_CLIENT_AUTH_REQUIRED;
	}


	/**
	 * Create instance with no properties in default name space {@link #DEFAULT_PROPERTY_NAME_SPACE}
	 */
    public DefaultClientConfigImpl() {
        this.dynamicProperties.clear();
        this.enableDynamicProperties = false;
    }

	/**
	 * Create instance with no properties in the specified name space
	 */
    public DefaultClientConfigImpl(String nameSpace) {
    	this();
    	this.propertyNameSpace = nameSpace;
    }

    public void loadDefaultValues() {
        putDefaultIntegerProperty(CommonClientConfigKey.MaxHttpConnectionsPerHost, getDefaultMaxHttpConnectionsPerHost());
        putDefaultIntegerProperty(CommonClientConfigKey.MaxTotalHttpConnections, getDefaultMaxTotalHttpConnections());
        putDefaultBooleanProperty(CommonClientConfigKey.EnableConnectionPool, getDefaultEnableConnectionPool());
        putDefaultIntegerProperty(CommonClientConfigKey.MaxConnectionsPerHost, getDefaultMaxConnectionsPerHost());
        putDefaultIntegerProperty(CommonClientConfigKey.MaxTotalConnections, getDefaultMaxTotalConnections());
        putDefaultIntegerProperty(CommonClientConfigKey.ConnectTimeout, getDefaultConnectTimeout());
        putDefaultIntegerProperty(CommonClientConfigKey.ConnectionManagerTimeout, getDefaultConnectionManagerTimeout());
        putDefaultIntegerProperty(CommonClientConfigKey.ReadTimeout, getDefaultReadTimeout());
        putDefaultIntegerProperty(CommonClientConfigKey.MaxAutoRetries, getDefaultMaxAutoRetries());
        putDefaultIntegerProperty(CommonClientConfigKey.MaxAutoRetriesNextServer, getDefaultMaxAutoRetriesNextServer());
        putDefaultBooleanProperty(CommonClientConfigKey.OkToRetryOnAllOperations, getDefaultOkToRetryOnAllOperations());
        putDefaultBooleanProperty(CommonClientConfigKey.FollowRedirects, getDefaultFollowRedirects());
        putDefaultBooleanProperty(CommonClientConfigKey.ConnectionPoolCleanerTaskEnabled, getDefaultConnectionPoolCleanerTaskEnabled());
        putDefaultIntegerProperty(CommonClientConfigKey.ConnIdleEvictTimeMilliSeconds, getDefaultConnectionidleTimeInMsecs());
        putDefaultIntegerProperty(CommonClientConfigKey.ConnectionCleanerRepeatInterval, getDefaultConnectionIdleTimertaskRepeatInMsecs());
        putDefaultBooleanProperty(CommonClientConfigKey.EnableGZIPContentEncodingFilter, getDefaultEnableGzipContentEncodingFilter());
        String proxyHost = ConfigurationManager.getConfigInstance().getString(getDefaultPropName(CommonClientConfigKey.ProxyHost.key()));
        if (proxyHost != null && proxyHost.length() > 0) {
            setProperty(CommonClientConfigKey.ProxyHost, proxyHost);
        }
        Integer proxyPort = ConfigurationManager
                .getConfigInstance()
                .getInteger(
                        getDefaultPropName(CommonClientConfigKey.ProxyPort),
                        (Integer.MIN_VALUE + 1)); // + 1 just to avoid potential clash with user setting
        if (proxyPort != (Integer.MIN_VALUE + 1)) {
            setProperty(CommonClientConfigKey.ProxyPort, proxyPort);
        }
        putDefaultIntegerProperty(CommonClientConfigKey.Port, getDefaultPort());
        putDefaultBooleanProperty(CommonClientConfigKey.EnablePrimeConnections, getDefaultEnablePrimeConnections());
        putDefaultIntegerProperty(CommonClientConfigKey.MaxRetriesPerServerPrimeConnection, getDefaultMaxRetriesPerServerPrimeConnection());
        putDefaultIntegerProperty(CommonClientConfigKey.MaxTotalTimeToPrimeConnections, getDefaultMaxTotalTimeToPrimeConnections());
        putDefaultStringProperty(CommonClientConfigKey.PrimeConnectionsURI, getDefaultPrimeConnectionsUri());
        putDefaultIntegerProperty(CommonClientConfigKey.PoolMinThreads, getDefaultPoolMinThreads());
        putDefaultIntegerProperty(CommonClientConfigKey.PoolMaxThreads, getDefaultPoolMaxThreads());
        putDefaultLongProperty(CommonClientConfigKey.PoolKeepAliveTime, getDefaultPoolKeepAliveTime());
        putDefaultTimeUnitProperty(CommonClientConfigKey.PoolKeepAliveTimeUnits, getDefaultPoolKeepAliveTimeUnits());
        putDefaultBooleanProperty(CommonClientConfigKey.EnableZoneAffinity, getDefaultEnableZoneAffinity());
        putDefaultBooleanProperty(CommonClientConfigKey.EnableZoneExclusivity, getDefaultEnableZoneExclusivity());
        putDefaultStringProperty(CommonClientConfigKey.ClientClassName, getDefaultClientClassname());
        putDefaultStringProperty(CommonClientConfigKey.NFLoadBalancerClassName, getDefaultNfloadbalancerClassname());
        putDefaultStringProperty(CommonClientConfigKey.NFLoadBalancerRuleClassName, getDefaultNfloadbalancerRuleClassname());
        putDefaultStringProperty(CommonClientConfigKey.NFLoadBalancerPingClassName, getDefaultNfloadbalancerPingClassname());
        putDefaultBooleanProperty(CommonClientConfigKey.PrioritizeVipAddressBasedServers, getDefaultPrioritizeVipAddressBasedServers());
        putDefaultFloatProperty(CommonClientConfigKey.MinPrimeConnectionsRatio, getDefaultMinPrimeConnectionsRatio());
        putDefaultStringProperty(CommonClientConfigKey.PrimeConnectionsClassName, getDefaultPrimeConnectionsClass());
        putDefaultStringProperty(CommonClientConfigKey.NIWSServerListClassName, getDefaultSeverListClass());
        putDefaultStringProperty(CommonClientConfigKey.VipAddressResolverClassName, getDefaultVipaddressResolverClassname());
        putDefaultBooleanProperty(CommonClientConfigKey.IsClientAuthRequired, getDefaultIsClientAuthRequired());
        // putDefaultStringProperty(CommonClientConfigKey.RequestIdHeaderName, getDefaultRequestIdHeaderName());
        putDefaultBooleanProperty(CommonClientConfigKey.UseIPAddrForServer, getDefaultUseIpAddressForServer());
        putDefaultStringProperty(CommonClientConfigKey.ListOfServers, "");
    }

    public Boolean getDefaultEnableConnectionPool() {
        return DEFAULT_ENABLE_CONNECTION_POOL;
    }

    protected void setPropertyInternal(IClientConfigKey propName, Object value) {
        setPropertyInternal(propName.key(), value);
    }

    private String getConfigKey(String propName) {
        return (clientName == null) ? getDefaultPropName(propName) : getInstancePropName(clientName, propName);
    }

    protected void setPropertyInternal(final String propName, Object value) {
        String stringValue = (value == null) ? "" : String.valueOf(value);
        properties.put(propName, stringValue);
        if (!enableDynamicProperties) {
            return;
        }
        String configKey = getConfigKey(propName);
        final DynamicStringProperty prop = DynamicPropertyFactory.getInstance().getStringProperty(configKey, null);
        Runnable callback = new Runnable() {
            @Override
            public void run() {
                String value = prop.get();
                if (value != null) {
                    properties.put(propName, value);
                } else {
                    properties.remove(propName);
                }
            }

            // equals and hashcode needed
            // since this is anonymous object is later used as a set key

            @Override
            public boolean equals(Object other){
            	if (other == null) {
            		return false;
            	}
            	if (getClass() == other.getClass()) {
                    return toString().equals(other.toString());
                }
                return false;
            }

            @Override
            public String toString(){
            	return propName;
            }

            @Override
            public int hashCode(){
            	return propName.hashCode();
            }


        };
        prop.addCallback(callback);
        dynamicProperties.put(propName, prop);
    }


	// Helper methods which first check if a "default" (with rest client name)
	// property exists. If so, that value is used, else the default value
	// passed as argument is used to put into the properties member variable
    protected void putDefaultIntegerProperty(IClientConfigKey propName, Integer defaultValue) {
        Integer value = ConfigurationManager.getConfigInstance().getInteger(
                getDefaultPropName(propName), defaultValue);
        setPropertyInternal(propName, value);
    }

    protected void putDefaultLongProperty(IClientConfigKey propName, Long defaultValue) {
        Long value = ConfigurationManager.getConfigInstance().getLong(
                getDefaultPropName(propName), defaultValue);
        setPropertyInternal(propName, value);
    }

    protected void putDefaultFloatProperty(IClientConfigKey propName, Float defaultValue) {
        Float value = ConfigurationManager.getConfigInstance().getFloat(
                getDefaultPropName(propName), defaultValue);
        setPropertyInternal(propName, value);
    }

    protected void putDefaultTimeUnitProperty(IClientConfigKey propName, TimeUnit defaultValue) {
        TimeUnit value = defaultValue;
        String propValue = ConfigurationManager.getConfigInstance().getString(
                getDefaultPropName(propName));
        if(propValue != null && propValue.length() > 0) {
            value = TimeUnit.valueOf(propValue);
        }
        setPropertyInternal(propName, value);
    }

    String getDefaultPropName(String propName) {
        return getNameSpace() + "." + propName;
    }

    public String getDefaultPropName(IClientConfigKey propName) {
        return getDefaultPropName(propName.key());
    }


    protected void putDefaultStringProperty(IClientConfigKey propName, String defaultValue) {
        String value = ConfigurationManager.getConfigInstance().getString(
                getDefaultPropName(propName), defaultValue);
        setPropertyInternal(propName, value);
    }

    protected void putDefaultBooleanProperty(IClientConfigKey propName, Boolean defaultValue) {
        Boolean value = ConfigurationManager.getConfigInstance().getBoolean(
                getDefaultPropName(propName), defaultValue);
        setPropertyInternal(propName, value);
    }

    public void setClientName(String clientName){
        this.clientName  = clientName;
    }

    /* (non-Javadoc)
	 * @see com.netflix.niws.client.CliengConfig#getClientName()
	 */
    @Override
	public String getClientName() {
        return clientName;
    }

    /**
     * Load properties for a given client. It first loads the default values for all properties,
     * and any properties already defined with Archaius ConfigurationManager.
     */
    @Override
	public void loadProperties(String restClientName){
        enableDynamicProperties = true;
        setClientName(restClientName);
        loadDefaultValues();
        Configuration props = ConfigurationManager.getConfigInstance().subset(restClientName);
        for (Iterator<String> keys = props.getKeys(); keys.hasNext(); ){
            String key = keys.next();
            String prop = key;
            try {
                if (prop.startsWith(getNameSpace())){
                    prop = prop.substring(getNameSpace().length() + 1);
                }
                setPropertyInternal(prop, getStringValue(props, key));
            } catch (Exception ex) {
                throw new RuntimeException(String.format("Property %s is invalid", prop));
            }
        }
    }
    
    /**
     * This is to workaround the issue that {@link AbstractConfiguration} by default
     * automatically convert comma delimited string to array
     */
    protected static String getStringValue(Configuration config, String key) {
        try {
            String values[] = config.getStringArray(key);
            if (values == null) {
                return null;
            }
            if (values.length == 0) {
                return config.getString(key);
            } else if (values.length == 1) {
                return values[0];
            }

            StringBuilder sb = new StringBuilder();
            for (int i = 0; i < values.length; i++) {
                sb.append(values[i]);
                if (i != values.length - 1) {
                    sb.append(",");
                }
            }
            return sb.toString();
        } catch (Exception e) {
            Object v = config.getProperty(key);
            if (v != null) {
                return String.valueOf(v);
            } else {
                return null;
            }
        }
    }

    @edu.umd.cs.findbugs.annotations.SuppressWarnings(value = "DC_DOUBLECHECK")
    private VipAddressResolver getVipAddressResolver() {
        if (resolver == null) {
            synchronized (this) {
                if (resolver == null) {
                    try {
                        resolver = (VipAddressResolver) Class
                                .forName((String) getProperty(CommonClientConfigKey.VipAddressResolverClassName))
                                .newInstance();
                    } catch (InstantiationException | IllegalAccessException | ClassNotFoundException e) {
                        throw new RuntimeException("Cannot instantiate VipAddressResolver", e);
                    }
                }
            }
        }
        return resolver;
    }

    @Override
	public String resolveDeploymentContextbasedVipAddresses(){
        String deploymentContextBasedVipAddressesMacro = (String) getProperty(CommonClientConfigKey.DeploymentContextBasedVipAddresses);
        if (deploymentContextBasedVipAddressesMacro == null) {
            return null;
        }
        return getVipAddressResolver().resolve(deploymentContextBasedVipAddressesMacro, this);
    }

    public String getAppName(){
        String appName = null;
        Object an = getProperty(CommonClientConfigKey.AppName);
        if (an!=null){
            appName = "" + an;
        }
        return appName;
    }

    public String getVersion(){
        String version = null;
        Object an = getProperty(CommonClientConfigKey.Version);
        if (an!=null){
            version = "" + an;
        }
        return version;
    }

    /* (non-Javadoc)
	 * @see com.netflix.niws.client.CliengConfig#getProperties()
	 */
    @Override
	public  Map<String, Object> getProperties() {
        return properties;
    }

    /* (non-Javadoc)
	 * @see com.netflix.niws.client.CliengConfig#setProperty(com.netflix.niws.client.ClientConfigKey, java.lang.Object)
	 */
    @Override
	public void setProperty(IClientConfigKey key, Object value){
        setPropertyInternal(key.key(), value);
    }
    
    public DefaultClientConfigImpl withProperty(IClientConfigKey key, Object value) {
        setProperty(key, value);
        return this;
    }

    public IClientConfig applyOverride(IClientConfig override) {
        if (override == null) {
            return this;
        }
        Map<String, Object> props = override.getProperties();
        for (Map.Entry<String, Object> entry: props.entrySet()) {
        	String key = entry.getKey();
        	Object value = entry.getValue();
        	if (key != null && value != null) {
        	    setPropertyInternal(key, value);
        	}
        }
        return this;
    }

    protected Object getProperty(String key) {
        if (enableDynamicProperties) {
            String dynamicValue = null;
            DynamicStringProperty dynamicProperty = dynamicProperties.get(key);
            if (dynamicProperty != null) {
                dynamicValue = dynamicProperty.get();
            }
            if (dynamicValue == null) {
                dynamicValue = DynamicProperty.getInstance(getConfigKey(key)).getString();
                if (dynamicValue == null) {
                    dynamicValue = DynamicProperty.getInstance(getDefaultPropName(key)).getString();
                }
            }
            if (dynamicValue != null) {
                return dynamicValue;
            }
        }
        return properties.get(key);
    }

    /* (non-Javadoc)
	 * @see com.netflix.niws.client.CliengConfig#getProperty(com.netflix.niws.client.ClientConfigKey)
	 */
    @Override
	public Object getProperty(IClientConfigKey key){
        String propName = key.key();
        Object value = getProperty(propName);
        return value;
    }

    /* (non-Javadoc)
	 * @see com.netflix.niws.client.CliengConfig#getProperty(com.netflix.niws.client.ClientConfigKey, java.lang.Object)
	 */
    @Override
	public Object getProperty(IClientConfigKey key, Object defaultVal){
        Object val = getProperty(key);
        if (val == null){
            return defaultVal;
        }
        return val;
    }

    public static Object getProperty(Map<String, Object> config, IClientConfigKey key, Object defaultVal) {
        Object val = config.get(key.key());
        if (val == null) {
            return defaultVal;
        }
        return val;
    }

    public static Object getProperty(Map<String, Object> config, IClientConfigKey key) {
        return getProperty(config, key, null);
    }

    public boolean isSecure() {
        Object oo = getProperty(CommonClientConfigKey.IsSecure);
        if (oo != null) {
            return Boolean.parseBoolean(oo.toString());
        } else {
        	return false;
        }
    }

    /* (non-Javadoc)
	 * @see com.netflix.niws.client.CliengConfig#containsProperty(com.netflix.niws.client.ClientConfigKey)
	 */
    @Override
	public boolean containsProperty(IClientConfigKey key){
        Object oo = getProperty(key);
        return oo!=null? true: false;
    }

    @Override
    public String toString(){
        final StringBuilder sb = new StringBuilder();
        String separator = "";

        sb.append("ClientConfig:");
        for (IClientConfigKey key: CommonClientConfigKey.values()) {
            final Object value = getProperty(key);

            sb.append(separator);
            separator = ", ";
            sb.append(key).append(":");
            if (key.key().endsWith("Password") && value instanceof String) {
                sb.append(Strings.repeat("*", ((String) value).length()));
            } else {
                sb.append(value);
            }
        }
        return sb.toString();
    }

    public void setProperty(Properties props, String restClientName, String key, String value){
        props.setProperty( getInstancePropName(restClientName, key), value);
    }

    public String getInstancePropName(String restClientName,
            IClientConfigKey configKey) {
        return getInstancePropName(restClientName, configKey.key());
    }

    public String getInstancePropName(String restClientName, String key) {
        return restClientName + "." + getNameSpace() + "."
                + key;
    }


	@Override
	public String getNameSpace() {
		return propertyNameSpace;
	}

	public static DefaultClientConfigImpl getEmptyConfig() {
	    return new DefaultClientConfigImpl();
	}
	
	public static DefaultClientConfigImpl getClientConfigWithDefaultValues(String clientName) {
		return getClientConfigWithDefaultValues(clientName, DEFAULT_PROPERTY_NAME_SPACE);
	}
	
	public static DefaultClientConfigImpl getClientConfigWithDefaultValues() {
        return getClientConfigWithDefaultValues("default", DEFAULT_PROPERTY_NAME_SPACE);
    }


	public static DefaultClientConfigImpl getClientConfigWithDefaultValues(String clientName, String nameSpace) {
	    DefaultClientConfigImpl config = new DefaultClientConfigImpl(nameSpace);
	    config.loadProperties(clientName);
		return config;
	}

    @Override
    public int getPropertyAsInteger(IClientConfigKey key, int defaultValue) {
        Object rawValue = getProperty(key);
        if (rawValue != null) {
            try {
                return Integer.parseInt(String.valueOf(rawValue));
            } catch (NumberFormatException e) {
                return defaultValue;
            }
        }
        return defaultValue;
        
    }

    @Override
    public String getPropertyAsString(IClientConfigKey key, String defaultValue) {
        Object rawValue = getProperty(key);
        if (rawValue != null) {
            return String.valueOf(rawValue);
        }
        return defaultValue;
    }

    @Override
    public boolean getPropertyAsBoolean(IClientConfigKey key,
            boolean defaultValue) {
        Object rawValue = getProperty(key);
        if (rawValue != null) {
            try {
                return Boolean.valueOf(String.valueOf(rawValue));
            } catch (NumberFormatException e) {
                return defaultValue;
            }
        }
        return defaultValue;
    }

    @SuppressWarnings("unchecked")
    @Override
    public <T> T get(IClientConfigKey<T> key) {
        Object obj = getProperty(key.key());
        if (obj == null) {
            return null;
        }
        Class<T> type = key.type();
        if (type.isInstance(obj)) {
            return type.cast(obj);
        } else {
            if (obj instanceof String) {
                String stringValue = (String) obj;
                if (Integer.class.equals(type)) {
                    return (T) Integer.valueOf(stringValue);
                } else if (Boolean.class.equals(type)) {
                    return (T) Boolean.valueOf(stringValue);
                } else if (Float.class.equals(type)) {
                    return (T) Float.valueOf(stringValue);
                } else if (Long.class.equals(type)) {
                    return (T) Long.valueOf(stringValue);
                } else if (Double.class.equals(type)) {
                    return (T) Double.valueOf(stringValue);
                } else if (TimeUnit.class.equals(type)) {
                    return (T) TimeUnit.valueOf(stringValue);
                }
                throw new IllegalArgumentException("Unable to convert string value to desired type " + type);
            }
             
            throw new IllegalArgumentException("Unable to convert value to desired type " + type);
        }
    }

    @Override
    public <T> DefaultClientConfigImpl set(IClientConfigKey<T> key, T value) {
        properties.put(key.key(), value);
        return this;
    }

    @Override
    public <T> T get(IClientConfigKey<T> key, T defaultValue) {
        T value = get(key);
        if (value == null) {
            value = defaultValue;
        }
        return value;
    }
}

```

## ClientConfigFactory.java

```java
package com.netflix.client.config;
/**
 * 客户端配置工厂
 */
public interface ClientConfigFactory {
    IClientConfig newConfig();

    public static class DefaultClientConfigFactory implements ClientConfigFactory {
        @Override
        public IClientConfig newConfig() {
            IClientConfig config = new DefaultClientConfigImpl();
            return config;
        }
    }
    
    public static final ClientConfigFactory DEFAULT = new DefaultClientConfigFactory();
}

```

## Resources.java

```java
package com.netflix.client.util;

public abstract class Resources {
    private static final Logger logger = LoggerFactory.getLogger(Resources.class);
    
    public static URL getResource(String resourceName) {
        URL url = null;
        // attempt to load from the context classpath
        ClassLoader loader = Thread.currentThread().getContextClassLoader();
        if (loader != null) {
            url = loader.getResource(resourceName);
        }
        if (url == null) {
            // attempt to load from the system classpath
            url = ClassLoader.getSystemResource(resourceName);
        }
        if (url == null) {
            try {
                resourceName = URLDecoder.decode(resourceName, "UTF-8");
                url = (new File(resourceName)).toURI().toURL();
            } catch (Exception e) {
                logger.error("Problem loading resource", e);
            }
        }
        return url;
    }
}

```

## ClientRequest.java

```java
package com.netflix.client;

import java.net.URI;

import com.netflix.client.config.IClientConfig;

/**
 * 客户端请求封装，IClientConfig这里面最好不要用
 */
public class ClientRequest implements Cloneable {

    protected URI uri;
    protected Object loadBalancerKey = null;
    protected Boolean isRetriable = null;
    protected IClientConfig overrideConfig;
        
    public ClientRequest() {
    }
    
    public ClientRequest(URI uri) {
        this.uri = uri;
    }

    @Deprecated
    public ClientRequest(URI uri, Object loadBalancerKey, boolean isRetriable, IClientConfig overrideConfig) {
        this.uri = uri;
        this.loadBalancerKey = loadBalancerKey;
        this.isRetriable = isRetriable;
        this.overrideConfig = overrideConfig;
    }

    public ClientRequest(URI uri, Object loadBalancerKey, boolean isRetriable) {
        this.uri = uri;
        this.loadBalancerKey = loadBalancerKey;
        this.isRetriable = isRetriable;
    }
    
    public ClientRequest(ClientRequest request) {
        this.uri = request.uri;
        this.loadBalancerKey = request.loadBalancerKey;
        this.overrideConfig = request.overrideConfig;
        this.isRetriable = request.isRetriable;
    }

    public final URI getUri() {
        return uri;
    }
    

    protected final ClientRequest setUri(URI uri) {
        this.uri = uri;
        return this;
    }

    public final Object getLoadBalancerKey() {
        return loadBalancerKey;
    }

    protected final ClientRequest setLoadBalancerKey(Object loadBalancerKey) {
        this.loadBalancerKey = loadBalancerKey;
        return this;
    }

    public boolean isRetriable() {
        return (Boolean.TRUE.equals(isRetriable));
    }

    protected final ClientRequest setRetriable(boolean isRetriable) {
        this.isRetriable = isRetriable;
        return this;
    }

    @Deprecated
    public final IClientConfig getOverrideConfig() {
        return overrideConfig;
    }

    @Deprecated
    protected final ClientRequest setOverrideConfig(IClientConfig overrideConfig) {
        this.overrideConfig = overrideConfig;
        return this;
    }

    public ClientRequest replaceUri(URI newURI) {
        ClientRequest req;
        try {
            req = (ClientRequest) this.clone();
        } catch (CloneNotSupportedException e) {
            req = new ClientRequest(this);
        }
        req.uri = newURI;
        return req;
    }
}

```

## IResponse.java

```java

package com.netflix.client;


/**
 * 客户端响应
 */
public interface IResponse extends Closeable{

   public Object getPayload() throws ClientException;

   public boolean hasPayload();

   public boolean isSuccess();

   public URI getRequestedURI();

   public Map<String, ?> getHeaders();   
}

```

## IClient.java

```java
package com.netflix.client;
/**
 * 执行单个请求的客户端
 */
public interface IClient<S extends ClientRequest, T extends IResponse> {

    public T execute(S request, IClientConfig requestConfig) throws Exception; 
}

```

## IClientConfigAware.java

```java
package com.netflix.client;

public interface IClientConfigAware {

    public abstract void initWithNiwsConfig(IClientConfig clientConfig);
    
}

```

## RetryHandler.java

```java
package com.netflix.client;
/**
 * 重试处理器
 */
public interface RetryHandler {

    public static final RetryHandler DEFAULT = new DefaultLoadBalancerRetryHandler();
    
    public boolean isRetriableException(Throwable e, boolean sameServer);

    public boolean isCircuitTrippingException(Throwable e);
        
    public int getMaxRetriesOnSameServer();

    public int getMaxRetriesOnNextServer();
}

```

### DefaultLoadBalancerRetryHandler.java

```java
package com.netflix.client;
/**
 * 默认的负载均衡重试处理器
 */
public class DefaultLoadBalancerRetryHandler implements RetryHandler {

    @SuppressWarnings("unchecked")
    private List<Class<? extends Throwable>> retriable = 
            Lists.<Class<? extends Throwable>>newArrayList(ConnectException.class, SocketTimeoutException.class);
    
    @SuppressWarnings("unchecked")
    private List<Class<? extends Throwable>> circuitRelated = 
            Lists.<Class<? extends Throwable>>newArrayList(SocketException.class, SocketTimeoutException.class);

    protected final int retrySameServer;
    protected final int retryNextServer;
    protected final boolean retryEnabled;

    public DefaultLoadBalancerRetryHandler() {
        this.retrySameServer = 0;
        this.retryNextServer = 0;
        this.retryEnabled = false;
    }
    
    public DefaultLoadBalancerRetryHandler(int retrySameServer, int retryNextServer, boolean retryEnabled) {
        this.retrySameServer = retrySameServer;
        this.retryNextServer = retryNextServer;
        this.retryEnabled = retryEnabled;
    }
    
    public DefaultLoadBalancerRetryHandler(IClientConfig clientConfig) {
        this.retrySameServer = clientConfig.get(CommonClientConfigKey.MaxAutoRetries, DefaultClientConfigImpl.DEFAULT_MAX_AUTO_RETRIES);
        this.retryNextServer = clientConfig.get(CommonClientConfigKey.MaxAutoRetriesNextServer, DefaultClientConfigImpl.DEFAULT_MAX_AUTO_RETRIES_NEXT_SERVER);
        this.retryEnabled = clientConfig.get(CommonClientConfigKey.OkToRetryOnAllOperations, false);
    }
    
    @Override
    public boolean isRetriableException(Throwable e, boolean sameServer) {
        if (retryEnabled) {
            if (sameServer) {
                return Utils.isPresentAsCause(e, getRetriableExceptions());
            } else {
                return true;
            }
        }
        return false;
    }

    @Override
    public boolean isCircuitTrippingException(Throwable e) {
        return Utils.isPresentAsCause(e, getCircuitRelatedExceptions());        
    }

    @Override
    public int getMaxRetriesOnSameServer() {
        return retrySameServer;
    }

    @Override
    public int getMaxRetriesOnNextServer() {
        return retryNextServer;
    }
    
    protected List<Class<? extends Throwable>> getRetriableExceptions() {
        return retriable;
    }
    
    protected List<Class<? extends Throwable>>  getCircuitRelatedExceptions() {
        return circuitRelated;
    }
}

```

### RequestSpecificRetryHandler.java

```java
package com.netflix.client;
s
public class RequestSpecificRetryHandler implements RetryHandler {

    private final RetryHandler fallback;
    private int retrySameServer = -1;
    private int retryNextServer = -1;
    private final boolean okToRetryOnConnectErrors;
    private final boolean okToRetryOnAllErrors;
    
    protected List<Class<? extends Throwable>> connectionRelated = 
            Lists.<Class<? extends Throwable>>newArrayList(SocketException.class);

    public RequestSpecificRetryHandler(boolean okToRetryOnConnectErrors, boolean okToRetryOnAllErrors) {
        this(okToRetryOnConnectErrors, okToRetryOnAllErrors, RetryHandler.DEFAULT, null);    
    }
    
    public RequestSpecificRetryHandler(boolean okToRetryOnConnectErrors, boolean okToRetryOnAllErrors, RetryHandler baseRetryHandler, @Nullable IClientConfig requestConfig) {
        Preconditions.checkNotNull(baseRetryHandler);
        this.okToRetryOnConnectErrors = okToRetryOnConnectErrors;
        this.okToRetryOnAllErrors = okToRetryOnAllErrors;
        this.fallback = baseRetryHandler;
        if (requestConfig != null) {
            if (requestConfig.containsProperty(CommonClientConfigKey.MaxAutoRetries)) {
                retrySameServer = requestConfig.get(CommonClientConfigKey.MaxAutoRetries); 
            }
            if (requestConfig.containsProperty(CommonClientConfigKey.MaxAutoRetriesNextServer)) {
                retryNextServer = requestConfig.get(CommonClientConfigKey.MaxAutoRetriesNextServer); 
            } 
        }
    }
    
    public boolean isConnectionException(Throwable e) {
        return Utils.isPresentAsCause(e, connectionRelated);
    }

    @Override
    public boolean isRetriableException(Throwable e, boolean sameServer) {
        if (okToRetryOnAllErrors) {
            return true;
        } 
        else if (e instanceof ClientException) {
            ClientException ce = (ClientException) e;
            if (ce.getErrorType() == ClientException.ErrorType.SERVER_THROTTLED) {
                return !sameServer;
            } else {
                return false;
            }
        } 
        else  {
            return okToRetryOnConnectErrors && isConnectionException(e);
        }
    }

    @Override
    public boolean isCircuitTrippingException(Throwable e) {
        return fallback.isCircuitTrippingException(e);
    }

    @Override
    public int getMaxRetriesOnSameServer() {
        if (retrySameServer >= 0) {
            return retrySameServer;
        }
        return fallback.getMaxRetriesOnSameServer();
    }

    @Override
    public int getMaxRetriesOnNextServer() {
        if (retryNextServer >= 0) {
            return retryNextServer;
        }
        return fallback.getMaxRetriesOnNextServer();
    }    
}

```



# end