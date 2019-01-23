# Feign


## Request.java

```java
package feign;
/**
 * 对http服务器的不可变请求。
 */
public final class Request {

  public static Request create(String method, String url, Map<String, Collection<String>> headers,
                               byte[] body, Charset charset) {
    return new Request(method, url, headers, body, charset);
  }

  private final String method;
  private final String url;
  private final Map<String, Collection<String>> headers;
  private final byte[] body;
  private final Charset charset;

  Request(String method, String url, Map<String, Collection<String>> headers, byte[] body,
          Charset charset) {
    this.method = checkNotNull(method, "method of %s", url);
    this.url = checkNotNull(url, "url");
    this.headers = checkNotNull(headers, "headers of %s %s", method, url);
    this.body = body; // nullable
    this.charset = charset; // nullable
  }

  public String method() {
    return method;
  }

  public String url() {
    return url;
  }

  public Map<String, Collection<String>> headers() {
    return headers;
  }

  public Charset charset() {
    return charset;
  }

  public byte[] body() {
    return body;
  }

  @Override
  public String toString() {
    StringBuilder builder = new StringBuilder();
    builder.append(method).append(' ').append(url).append(" HTTP/1.1\n");
    for (String field : headers.keySet()) {
      for (String value : valuesOrEmpty(headers, field)) {
        builder.append(field).append(": ").append(value).append('\n');
      }
    }
    if (body != null) {
      builder.append('\n').append(charset != null ? new String(body, charset) : "Binary data");
    }
    return builder.toString();
  }

  public static class Options {

    private final int connectTimeoutMillis;
    private final int readTimeoutMillis;

    public Options(int connectTimeoutMillis, int readTimeoutMillis) {
      this.connectTimeoutMillis = connectTimeoutMillis;
      this.readTimeoutMillis = readTimeoutMillis;
    }

    public Options() {
      this(10 * 1000, 60 * 1000);
    }

    public int connectTimeoutMillis() {
      return connectTimeoutMillis;
    }

    public int readTimeoutMillis() {
      return readTimeoutMillis;
    }
  }
}

```

## Response.java

```java
package feign;
/**
 * 对只返回字符串内容的http调用的不可变响应。
 */
public final class Response implements Closeable {

  private final int status;
  private final String reason;
  private final Map<String, Collection<String>> headers;
  private final Body body;

  private Response(int status, String reason, Map<String, Collection<String>> headers, Body body) {
    checkState(status >= 200, "Invalid status code: %s", status);
    this.status = status;
    this.reason = reason; //nullable
    this.headers = Collections.unmodifiableMap(caseInsensitiveCopyOf(headers));
    this.body = body; //nullable
  }

  public static Response create(int status, String reason, Map<String, Collection<String>> headers,
                                InputStream inputStream, Integer length) {
    return new Response(status, reason, headers, InputStreamBody.orNull(inputStream, length));
  }

  public static Response create(int status, String reason, Map<String, Collection<String>> headers,
                                byte[] data) {
    return new Response(status, reason, headers, ByteArrayBody.orNull(data));
  }

  public static Response create(int status, String reason, Map<String, Collection<String>> headers,
                                String text, Charset charset) {
    return new Response(status, reason, headers, ByteArrayBody.orNull(text, charset));
  }

  public static Response create(int status, String reason, Map<String, Collection<String>> headers,
                                Body body) {
    return new Response(status, reason, headers, body);
  }

  public int status() {
    return status;
  }

  public String reason() {
    return reason;
  }

  public Map<String, Collection<String>> headers() {
    return headers;
  }

  public Body body() {
    return body;
  }

  @Override
  public String toString() {
    StringBuilder builder = new StringBuilder("HTTP/1.1 ").append(status);
    if (reason != null) builder.append(' ').append(reason);
    builder.append('\n');
    for (String field : headers.keySet()) {
      for (String value : valuesOrEmpty(headers, field)) {
        builder.append(field).append(": ").append(value).append('\n');
      }
    }
    if (body != null) builder.append('\n').append(body);
    return builder.toString();
  }

  @Override
  public void close() {
    Util.ensureClosed(body);
  }

  public interface Body extends Closeable {

    Integer length();

    boolean isRepeatable();

    InputStream asInputStream() throws IOException;

    Reader asReader() throws IOException;
  }

  private static final class InputStreamBody implements Response.Body {

    private final InputStream inputStream;
    private final Integer length;
    private InputStreamBody(InputStream inputStream, Integer length) {
      this.inputStream = inputStream;
      this.length = length;
    }

    private static Body orNull(InputStream inputStream, Integer length) {
      if (inputStream == null) {
        return null;
      }
      return new InputStreamBody(inputStream, length);
    }

    @Override
    public Integer length() {
      return length;
    }

    @Override
    public boolean isRepeatable() {
      return false;
    }

    @Override
    public InputStream asInputStream() throws IOException {
      return inputStream;
    }

    @Override
    public Reader asReader() throws IOException {
      return new InputStreamReader(inputStream, UTF_8);
    }

    @Override
    public void close() throws IOException {
      inputStream.close();
    }
  }

  private static final class ByteArrayBody implements Response.Body {

    private final byte[] data;

    public ByteArrayBody(byte[] data) {
      this.data = data;
    }

    private static Body orNull(byte[] data) {
      if (data == null) {
        return null;
      }
      return new ByteArrayBody(data);
    }

    private static Body orNull(String text, Charset charset) {
      if (text == null) {
        return null;
      }
      checkNotNull(charset, "charset");
      return new ByteArrayBody(text.getBytes(charset));
    }

    @Override
    public Integer length() {
      return data.length;
    }

    @Override
    public boolean isRepeatable() {
      return true;
    }

    @Override
    public InputStream asInputStream() throws IOException {
      return new ByteArrayInputStream(data);
    }

    @Override
    public Reader asReader() throws IOException {
      return new InputStreamReader(asInputStream(), UTF_8);
    }

    @Override
    public void close() throws IOException {
    }

    @Override
    public String toString() {
      return decodeOrDefault(data, UTF_8, "Binary data");
    }
  }

  private static Map<String, Collection<String>> caseInsensitiveCopyOf(Map<String, Collection<String>> headers) {
    Map<String, Collection<String>> result = new TreeMap<String, Collection<String>>(String.CASE_INSENSITIVE_ORDER);

    for (Map.Entry<String, Collection<String>> entry : headers.entrySet()) {
      String headerName = entry.getKey();
      if (!result.containsKey(headerName)) {
        result.put(headerName.toLowerCase(Locale.ROOT), new LinkedList<String>());
      }
      result.get(headerName).addAll(entry.getValue());
    }
    return result;
  }
}

```
## RequestTemplate.java

```java
package feign;
/**
 * 构建对http目标的请求。不是线程安全的。
 */
public final class RequestTemplate implements Serializable {

  private static final long serialVersionUID = 1L;
  private final Map<String, Collection<String>> queries =
      new LinkedHashMap<String, Collection<String>>();
  private final Map<String, Collection<String>> headers =
      new LinkedHashMap<String, Collection<String>>();
  private String method;
  private StringBuilder url = new StringBuilder();
  private transient Charset charset;
  private byte[] body;
  private String bodyTemplate;
  private boolean decodeSlash = true;

  public RequestTemplate() {
  }

  public RequestTemplate(RequestTemplate toCopy) {
    checkNotNull(toCopy, "toCopy");
    this.method = toCopy.method;
    this.url.append(toCopy.url);
    this.queries.putAll(toCopy.queries);
    this.headers.putAll(toCopy.headers);
    this.charset = toCopy.charset;
    this.body = toCopy.body;
    this.bodyTemplate = toCopy.bodyTemplate;
    this.decodeSlash = toCopy.decodeSlash;
  }

  private static String urlDecode(String arg) {
    try {
      return URLDecoder.decode(arg, UTF_8.name());
    } catch (UnsupportedEncodingException e) {
      throw new RuntimeException(e);
    }
  }

  private static String urlEncode(Object arg) {
    try {
      return URLEncoder.encode(String.valueOf(arg), UTF_8.name());
    } catch (UnsupportedEncodingException e) {
      throw new RuntimeException(e);
    }
  }

  private static boolean isHttpUrl(CharSequence value) {
    return value.length() >= 4 && value.subSequence(0, 3).equals("http".substring(0,  3));
  }

  private static CharSequence removeTrailingSlash(CharSequence charSequence) {
    if (charSequence != null && charSequence.length() > 0 && charSequence.charAt(charSequence.length() - 1) == '/') {
      return charSequence.subSequence(0, charSequence.length() - 1);
    } else {
      return charSequence;
    }
  }

  public static String expand(String template, Map<String, ?> variables) {
    // skip expansion if there's no valid variables set. ex. {a} is the
    // first valid
    if (checkNotNull(template, "template").length() < 3) {
      return template.toString();
    }
    checkNotNull(variables, "variables for %s", template);

    boolean inVar = false;
    StringBuilder var = new StringBuilder();
    StringBuilder builder = new StringBuilder();
    for (char c : template.toCharArray()) {
      switch (c) {
        case '{':
          if (inVar) {
            // '{{' is an escape: write the brace and don't interpret as a variable
            builder.append("{");
            inVar = false;
            break;
          }
          inVar = true;
          break;
        case '}':
          if (!inVar) { // then write the brace literally
            builder.append('}');
            break;
          }
          inVar = false;
          String key = var.toString();
          Object value = variables.get(var.toString());
          if (value != null) {
            builder.append(value);
          } else {
            builder.append('{').append(key).append('}');
          }
          var = new StringBuilder();
          break;
        default:
          if (inVar) {
            var.append(c);
          } else {
            builder.append(c);
          }
      }
    }
    return builder.toString();
  }

  private static Map<String, Collection<String>> parseAndDecodeQueries(String queryLine) {
    Map<String, Collection<String>> map = new LinkedHashMap<String, Collection<String>>();
    if (emptyToNull(queryLine) == null) {
      return map;
    }
    if (queryLine.indexOf('&') == -1) {
      putKV(queryLine, map);
    } else {
      char[] chars = queryLine.toCharArray();
      int start = 0;
      int i = 0;
      for (; i < chars.length; i++) {
        if (chars[i] == '&') {
          putKV(queryLine.substring(start, i), map);
          start = i + 1;
        }
      }
      putKV(queryLine.substring(start, i), map);
    }
    return map;
  }

  private static void putKV(String stringToParse, Map<String, Collection<String>> map) {
    String key;
    String value;
    // note that '=' can be a valid part of the value
    int firstEq = stringToParse.indexOf('=');
    if (firstEq == -1) {
      key = urlDecode(stringToParse);
      value = null;
    } else {
      key = urlDecode(stringToParse.substring(0, firstEq));
      value = urlDecode(stringToParse.substring(firstEq + 1));
    }
    Collection<String> values = map.containsKey(key) ? map.get(key) : new ArrayList<String>();
    values.add(value);
    map.put(key, values);
  }

  public RequestTemplate resolve(Map<String, ?> unencoded) {
    replaceQueryValues(unencoded);
    Map<String, String> encoded = new LinkedHashMap<String, String>();
    for (Entry<String, ?> entry : unencoded.entrySet()) {
      encoded.put(entry.getKey(), urlEncode(String.valueOf(entry.getValue())));
    }
    String resolvedUrl = expand(url.toString(), encoded).replace("+", "%20");
    if (decodeSlash) {
    	resolvedUrl = resolvedUrl.replace("%2F", "/");
    }
    url = new StringBuilder(resolvedUrl);

    Map<String, Collection<String>> resolvedHeaders = new LinkedHashMap<String, Collection<String>>();
    for (String field : headers.keySet()) {
      Collection<String> resolvedValues = new ArrayList<String>();
      for (String value : valuesOrEmpty(headers, field)) {
        String resolved = expand(value, unencoded);
        resolvedValues.add(resolved);
      }
      resolvedHeaders.put(field, resolvedValues);
    }
    headers.clear();
    headers.putAll(resolvedHeaders);
    if (bodyTemplate != null) {
      body(urlDecode(expand(bodyTemplate, encoded)));
    }
    return this;
  }

  public Request request() {
    Map<String, Collection<String>> safeCopy = new LinkedHashMap<String, Collection<String>>();
    safeCopy.putAll(headers);
    return Request.create(
        method,
        new StringBuilder(url).append(queryLine()).toString(),
        Collections.unmodifiableMap(safeCopy),
        body, charset
    );
  }

  public RequestTemplate method(String method) {
    this.method = checkNotNull(method, "method");
    checkArgument(method.matches("^[A-Z]+$"), "Invalid HTTP Method: %s", method);
    return this;
  }
  
  public String method() {
    return method;
  }

  public RequestTemplate decodeSlash(boolean decodeSlash) {
    this.decodeSlash = decodeSlash;
    return this;
  }
  
  public boolean decodeSlash() {
    return decodeSlash;
  }

  public RequestTemplate append(CharSequence value) {
    url.append(value);
    url = pullAnyQueriesOutOfUrl(url);
    return this;
  }

  public RequestTemplate insert(int pos, CharSequence value) {
    if(isHttpUrl(value)) {
      value = removeTrailingSlash(value);
      if(url.length() > 0 && url.charAt(0) != '/') {
        url.insert(0, '/');
      }
    }
    url.insert(pos, pullAnyQueriesOutOfUrl(new StringBuilder(value)));
    return this;
  }

  public String url() {
    return url.toString();
  }

  public RequestTemplate query(boolean encoded, String name, String... values) {
    return doQuery(encoded, name, values);
  }

  public RequestTemplate query(boolean encoded, String name, Iterable<String> values) {
    return doQuery(encoded, name, values);
  }

  public RequestTemplate query(String name, String... values) {
    return doQuery(false, name, values);
  }

  public RequestTemplate query(String name, Iterable<String> values) {
    return doQuery(false, name, values);
  }

  private RequestTemplate doQuery(boolean encoded, String name, String... values) {
    checkNotNull(name, "name");
    String paramName = encoded ? name : encodeIfNotVariable(name);
    queries.remove(paramName);
    if (values != null && values.length > 0 && values[0] != null) {
      ArrayList<String> paramValues = new ArrayList<String>();
      for (String value : values) {
        paramValues.add(encoded ? value : encodeIfNotVariable(value));
      }
      this.queries.put(paramName, paramValues);
    }
    return this;
  }

  private RequestTemplate doQuery(boolean encoded, String name, Iterable<String> values) {
    if (values != null) {
      return doQuery(encoded, name, toArray(values, String.class));
    }
    return doQuery(encoded, name, (String[]) null);
  }

  private static String encodeIfNotVariable(String in) {
    if (in == null || in.indexOf('{') == 0) {
      return in;
    }
    return urlEncode(in);
  }

  public RequestTemplate queries(Map<String, Collection<String>> queries) {
    if (queries == null || queries.isEmpty()) {
      this.queries.clear();
    } else {
      for (Entry<String, Collection<String>> entry : queries.entrySet()) {
        query(entry.getKey(), toArray(entry.getValue(), String.class));
      }
    }
    return this;
  }

  public Map<String, Collection<String>> queries() {
    Map<String, Collection<String>> decoded = new LinkedHashMap<String, Collection<String>>();
    for (String field : queries.keySet()) {
      Collection<String> decodedValues = new ArrayList<String>();
      for (String value : valuesOrEmpty(queries, field)) {
        if (value != null) {
          decodedValues.add(urlDecode(value));
        } else {
          decodedValues.add(null);
        }
      }
      decoded.put(urlDecode(field), decodedValues);
    }
    return Collections.unmodifiableMap(decoded);
  }

  public RequestTemplate header(String name, String... values) {
    checkNotNull(name, "header name");
    if (values == null || (values.length == 1 && values[0] == null)) {
      headers.remove(name);
    } else {
      List<String> headers = new ArrayList<String>();
      headers.addAll(Arrays.asList(values));
      this.headers.put(name, headers);
    }
    return this;
  }

  public RequestTemplate header(String name, Iterable<String> values) {
    if (values != null) {
      return header(name, toArray(values, String.class));
    }
    return header(name, (String[]) null);
  }

  public RequestTemplate headers(Map<String, Collection<String>> headers) {
    if (headers == null || headers.isEmpty()) {
      this.headers.clear();
    } else {
      this.headers.putAll(headers);
    }
    return this;
  }

  public Map<String, Collection<String>> headers() {
    return Collections.unmodifiableMap(headers);
  }

  public RequestTemplate body(byte[] bodyData, Charset charset) {
    this.bodyTemplate = null;
    this.charset = charset;
    this.body = bodyData;
    int bodyLength = bodyData != null ? bodyData.length : 0;
    header(CONTENT_LENGTH, String.valueOf(bodyLength));
    return this;
  }

  public RequestTemplate body(String bodyText) {
    byte[] bodyData = bodyText != null ? bodyText.getBytes(UTF_8) : null;
    return body(bodyData, UTF_8);
  }

  public Charset charset() {
    return charset;
  }

  public byte[] body() {
    return body;
  }

  public RequestTemplate bodyTemplate(String bodyTemplate) {
    this.bodyTemplate = bodyTemplate;
    this.charset = null;
    this.body = null;
    return this;
  }

  public String bodyTemplate() {
    return bodyTemplate;
  }

  private StringBuilder pullAnyQueriesOutOfUrl(StringBuilder url) {
    // parse out queries
    int queryIndex = url.indexOf("?");
    if (queryIndex != -1) {
      String queryLine = url.substring(queryIndex + 1);
      Map<String, Collection<String>> firstQueries = parseAndDecodeQueries(queryLine);
      if (!queries.isEmpty()) {
        firstQueries.putAll(queries);
        queries.clear();
      }
      //Since we decode all queries, we want to use the
      //query()-method to re-add them to ensure that all
      //logic (such as url-encoding) are executed, giving
      //a valid queryLine()
      for (String key : firstQueries.keySet()) {
        Collection<String> values = firstQueries.get(key);
        if (allValuesAreNull(values)) {
          //Queries where all values are null will
          //be ignored by the query(key, value)-method
          //So we manually avoid this case here, to ensure that
          //we still fulfill the contract (ex. parameters without values)
          queries.put(urlEncode(key), values);
        } else {
          query(key, values);
        }

      }
      return new StringBuilder(url.substring(0, queryIndex));
    }
    return url;
  }

  private boolean allValuesAreNull(Collection<String> values) {
    if (values == null || values.isEmpty()) {
      return true;
    }
    for (String val : values) {
      if (val != null) {
        return false;
      }
    }
    return true;
  }

  @Override
  public String toString() {
    return request().toString();
  }

  /**
   * Replaces query values which are templated with corresponding values from the {@code unencoded}
   * map. Any unresolved queries are removed.
   */
  public void replaceQueryValues(Map<String, ?> unencoded) {
    Iterator<Entry<String, Collection<String>>> iterator = queries.entrySet().iterator();
    while (iterator.hasNext()) {
      Entry<String, Collection<String>> entry = iterator.next();
      if (entry.getValue() == null) {
        continue;
      }
      Collection<String> values = new ArrayList<String>();
      for (String value : entry.getValue()) {
        if (value.indexOf('{') == 0 && value.indexOf('}') == value.length() - 1) {
          Object variableValue = unencoded.get(value.substring(1, value.length() - 1));
          // only add non-null expressions
          if (variableValue == null) {
            continue;
          }
          if (variableValue instanceof Iterable) {
            for (Object val : Iterable.class.cast(variableValue)) {
              values.add(urlEncode(String.valueOf(val)));
            }
          } else {
            values.add(urlEncode(String.valueOf(variableValue)));
          }
        } else {
          values.add(value);
        }
      }
      if (values.isEmpty()) {
        iterator.remove();
      } else {
        entry.setValue(values);
      }
    }
  }

  public String queryLine() {
    if (queries.isEmpty()) {
      return "";
    }
    StringBuilder queryBuilder = new StringBuilder();
    for (String field : queries.keySet()) {
      for (String value : valuesOrEmpty(queries, field)) {
        queryBuilder.append('&');
        queryBuilder.append(field);
        if (value != null) {
          queryBuilder.append('=');
          if (!value.isEmpty()) {
            queryBuilder.append(value);
          }
        }
      }
    }
    queryBuilder.deleteCharAt(0);
    return queryBuilder.insert(0, '?').toString();
  }

  interface Factory {

    /**
     * create a request template using args passed to a method invocation.
     */
    RequestTemplate create(Object[] argv);
  }
}
```

## RequestInterceptor.java

```java
package feign;

/**
 * 请求拦截器
 */
public interface RequestInterceptor {

  void apply(RequestTemplate template);
}

```

### BasicAuthRequestInterceptor.java

```java
package feign.auth;
/**
 * An interceptor that adds the request header needed to use HTTP basic authentication.
 */
public class BasicAuthRequestInterceptor implements RequestInterceptor {

  private final String headerValue;

  public BasicAuthRequestInterceptor(String username, String password) {
    this(username, password, ISO_8859_1);
  }
  public BasicAuthRequestInterceptor(String username, String password, Charset charset) {
    checkNotNull(username, "username");
    checkNotNull(password, "password");
    this.headerValue = "Basic " + base64Encode((username + ":" + password).getBytes(charset));
  }

  private static String base64Encode(byte[] bytes) {
    return Base64.encode(bytes);
  }

  @Override
  public void apply(RequestTemplate template) {
    template.header("Authorization", headerValue);
  }
}


```

## Logger.java

```java
package feign;
/**
 * Simple logging abstraction for debug messages.  Adapted from {@code retrofit.RestAdapter.Log}.
 */
public abstract class Logger {

  protected static String methodTag(String configKey) {
    return new StringBuilder().append('[').append(configKey.substring(0, configKey.indexOf('(')))
        .append("] ").toString();
  }

  protected abstract void log(String configKey, String format, Object... args);

  protected void logRequest(String configKey, Level logLevel, Request request) {
    log(configKey, "---> %s %s HTTP/1.1", request.method(), request.url());
    if (logLevel.ordinal() >= Level.HEADERS.ordinal()) {

      for (String field : request.headers().keySet()) {
        for (String value : valuesOrEmpty(request.headers(), field)) {
          log(configKey, "%s: %s", field, value);
        }
      }

      int bodyLength = 0;
      if (request.body() != null) {
        bodyLength = request.body().length;
        if (logLevel.ordinal() >= Level.FULL.ordinal()) {
          String
              bodyText =
              request.charset() != null ? new String(request.body(), request.charset()) : null;
          log(configKey, ""); // CRLF
          log(configKey, "%s", bodyText != null ? bodyText : "Binary data");
        }
      }
      log(configKey, "---> END HTTP (%s-byte body)", bodyLength);
    }
  }

  void logRetry(String configKey, Level logLevel) {
    log(configKey, "---> RETRYING");
  }

  protected Response logAndRebufferResponse(String configKey, Level logLevel, Response response,
                                            long elapsedTime) throws IOException {
    String reason = response.reason() != null && logLevel.compareTo(Level.NONE) > 0 ?
        " " + response.reason() : "";
    log(configKey, "<--- HTTP/1.1 %s%s (%sms)", response.status(), reason, elapsedTime);
    if (logLevel.ordinal() >= Level.HEADERS.ordinal()) {

      for (String field : response.headers().keySet()) {
        for (String value : valuesOrEmpty(response.headers(), field)) {
          log(configKey, "%s: %s", field, value);
        }
      }

      int bodyLength = 0;
      if (response.body() != null) {
        if (logLevel.ordinal() >= Level.FULL.ordinal()) {
          log(configKey, ""); // CRLF
        }
        byte[] bodyData = Util.toByteArray(response.body().asInputStream());
        bodyLength = bodyData.length;
        if (logLevel.ordinal() >= Level.FULL.ordinal() && bodyLength > 0) {
          log(configKey, "%s", decodeOrDefault(bodyData, UTF_8, "Binary data"));
        }
        log(configKey, "<--- END HTTP (%s-byte body)", bodyLength);
        return Response.create(response.status(), response.reason(), response.headers(), bodyData);
      } else {
        log(configKey, "<--- END HTTP (%s-byte body)", bodyLength);
      }
    }
    return response;
  }

  IOException logIOException(String configKey, Level logLevel, IOException ioe, long elapsedTime) {
    log(configKey, "<--- ERROR %s: %s (%sms)", ioe.getClass().getSimpleName(), ioe.getMessage(),
        elapsedTime);
    if (logLevel.ordinal() >= Level.FULL.ordinal()) {
      StringWriter sw = new StringWriter();
      ioe.printStackTrace(new PrintWriter(sw));
      log(configKey, sw.toString());
      log(configKey, "<--- END ERROR");
    }
    return ioe;
  }

  /**
   * Controls the level of logging.
   */
  public enum Level {
    /**
     * No logging.
     */
    NONE,
    /**
     * Log only the request method and URL and the response status code and execution time.
     */
    BASIC,
    /**
     * Log the basic information along with request and response headers.
     */
    HEADERS,
    /**
     * Log the headers, body, and metadata for both requests and responses.
     */
    FULL
  }

  /**
   * logs to the category {@link Logger} at {@link java.util.logging.Level#FINE}.
   */
  public static class ErrorLogger extends Logger {
    @Override
    protected void log(String configKey, String format, Object... args) {
      System.err.printf(methodTag(configKey) + format + "%n", args);
    }
  }

  /**
   * logs to the category {@link Logger} at {@link java.util.logging.Level#FINE}, if loggable.
   */
  public static class JavaLogger extends Logger {

    final java.util.logging.Logger
        logger =
        java.util.logging.Logger.getLogger(Logger.class.getName());

    @Override
    protected void logRequest(String configKey, Level logLevel, Request request) {
      if (logger.isLoggable(java.util.logging.Level.FINE)) {
        super.logRequest(configKey, logLevel, request);
      }
    }

    @Override
    protected Response logAndRebufferResponse(String configKey, Level logLevel, Response response,
                                              long elapsedTime) throws IOException {
      if (logger.isLoggable(java.util.logging.Level.FINE)) {
        return super.logAndRebufferResponse(configKey, logLevel, response, elapsedTime);
      }
      return response;
    }

    @Override
    protected void log(String configKey, String format, Object... args) {
      logger.fine(String.format(methodTag(configKey) + format, args));
    }

    /**
     * helper that configures jul to sanely log messages at FINE level without additional
     * formatting.
     */
    public JavaLogger appendToFile(String logfile) {
      logger.setLevel(java.util.logging.Level.FINE);
      try {
        FileHandler handler = new FileHandler(logfile, true);
        handler.setFormatter(new SimpleFormatter() {
          @Override
          public String format(LogRecord record) {
            return String.format("%s%n", record.getMessage()); // NOPMD
          }
        });
        logger.addHandler(handler);
      } catch (IOException e) {
        throw new IllegalStateException("Could not add file handler.", e);
      }
      return this;
    }
  }

  public static class NoOpLogger extends Logger {

    @Override
    protected void logRequest(String configKey, Level logLevel, Request request) {
    }

    @Override
    protected Response logAndRebufferResponse(String configKey, Level logLevel, Response response,
                                              long elapsedTime) throws IOException {
      return response;
    }

    @Override
    protected void log(String configKey, String format, Object... args) {
    }
  }
}

```

## MethodMetadata.java

```java
package feign;

public final class MethodMetadata implements Serializable {

  private static final long serialVersionUID = 1L;
  private String configKey;
  private transient Type returnType;
  private Integer urlIndex;
  private Integer bodyIndex;
  private Integer headerMapIndex;
  private Integer queryMapIndex;
  private boolean queryMapEncoded;
  private transient Type bodyType;
  private RequestTemplate template = new RequestTemplate();
  private List<String> formParams = new ArrayList<String>();
  private Map<Integer, Collection<String>> indexToName =
      new LinkedHashMap<Integer, Collection<String>>();
  private Map<Integer, Class<? extends Expander>> indexToExpanderClass =
      new LinkedHashMap<Integer, Class<? extends Expander>>();
  private transient Map<Integer, Expander> indexToExpander;

  MethodMetadata() {
  }

  /**
   * @see Feign#configKey(Class, java.lang.reflect.Method)
   */
  public String configKey() {
    return configKey;
  }

  public MethodMetadata configKey(String configKey) {
    this.configKey = configKey;
    return this;
  }

  public Type returnType() {
    return returnType;
  }

  public MethodMetadata returnType(Type returnType) {
    this.returnType = returnType;
    return this;
  }

  public Integer urlIndex() {
    return urlIndex;
  }

  public MethodMetadata urlIndex(Integer urlIndex) {
    this.urlIndex = urlIndex;
    return this;
  }

  public Integer bodyIndex() {
    return bodyIndex;
  }

  public MethodMetadata bodyIndex(Integer bodyIndex) {
    this.bodyIndex = bodyIndex;
    return this;
  }

  public Integer headerMapIndex() {
    return headerMapIndex;
  }

  public MethodMetadata headerMapIndex(Integer headerMapIndex) {
    this.headerMapIndex = headerMapIndex;
    return this;
  }

  public Integer queryMapIndex() {
    return queryMapIndex;
  }

  public MethodMetadata queryMapIndex(Integer queryMapIndex) {
    this.queryMapIndex = queryMapIndex;
    return this;
  }

  public boolean queryMapEncoded() {
    return queryMapEncoded;
  }

  public MethodMetadata queryMapEncoded(boolean queryMapEncoded) {
    this.queryMapEncoded = queryMapEncoded;
    return this;
  }

  /**
   * Type corresponding to {@link #bodyIndex()}.
   */
  public Type bodyType() {
    return bodyType;
  }

  public MethodMetadata bodyType(Type bodyType) {
    this.bodyType = bodyType;
    return this;
  }

  public RequestTemplate template() {
    return template;
  }

  public List<String> formParams() {
    return formParams;
  }

  public Map<Integer, Collection<String>> indexToName() {
    return indexToName;
  }

  /**
   * If {@link #indexToExpander} is null, classes here will be instantiated by newInstance.
   */
  public Map<Integer, Class<? extends Expander>> indexToExpanderClass() {
    return indexToExpanderClass;
  }

  /**
   * After {@link #indexToExpanderClass} is populated, this is set by contracts that support
   * runtime injection.
   */
  public MethodMetadata indexToExpander(Map<Integer, Expander> indexToExpander) {
    this.indexToExpander = indexToExpander;
    return this;
  }

  /**
   * When not null, this value will be used instead of {@link #indexToExpander()}.
   */
  public Map<Integer, Expander> indexToExpander() {
    return indexToExpander;
  }
}

```

## Contract.java

```java
package feign;
/**
 * 解析接口上的注解
 */
public interface Contract {

  List<MethodMetadata> parseAndValidatateMetadata(Class<?> targetType);

  abstract class BaseContract implements Contract {

    @Override
    public List<MethodMetadata> parseAndValidatateMetadata(Class<?> targetType) {
      checkState(targetType.getTypeParameters().length == 0, "Parameterized types unsupported: %s",
                 targetType.getSimpleName());
      checkState(targetType.getInterfaces().length <= 1, "Only single inheritance supported: %s",
                 targetType.getSimpleName());
      if (targetType.getInterfaces().length == 1) {
        checkState(targetType.getInterfaces()[0].getInterfaces().length == 0,
                   "Only single-level inheritance supported: %s",
                   targetType.getSimpleName());
      }
      Map<String, MethodMetadata> result = new LinkedHashMap<String, MethodMetadata>();
      for (Method method : targetType.getMethods()) {
        if (method.getDeclaringClass() == Object.class ||
            (method.getModifiers() & Modifier.STATIC) != 0 ||
            Util.isDefault(method)) {
          continue;
        }
        MethodMetadata metadata = parseAndValidateMetadata(targetType, method);
        checkState(!result.containsKey(metadata.configKey()), "Overrides unsupported: %s",
                   metadata.configKey());
        result.put(metadata.configKey(), metadata);
      }
      return new ArrayList<MethodMetadata>(result.values());
    }

    /**
     * @deprecated use {@link #parseAndValidateMetadata(Class, Method)} instead.
     */
    @Deprecated
    public MethodMetadata parseAndValidatateMetadata(Method method) {
      return parseAndValidateMetadata(method.getDeclaringClass(), method);
    }

    /**
     * Called indirectly by {@link #parseAndValidatateMetadata(Class)}.
     */
    protected MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {
      MethodMetadata data = new MethodMetadata();
      data.returnType(Types.resolve(targetType, targetType, method.getGenericReturnType()));
      data.configKey(Feign.configKey(targetType, method));

      if(targetType.getInterfaces().length == 1) {
        processAnnotationOnClass(data, targetType.getInterfaces()[0]);
      }
      processAnnotationOnClass(data, targetType);


      for (Annotation methodAnnotation : method.getAnnotations()) {
        processAnnotationOnMethod(data, methodAnnotation, method);
      }
      checkState(data.template().method() != null,
                 "Method %s not annotated with HTTP method type (ex. GET, POST)",
                 method.getName());
      Class<?>[] parameterTypes = method.getParameterTypes();

      Annotation[][] parameterAnnotations = method.getParameterAnnotations();
      int count = parameterAnnotations.length;
      for (int i = 0; i < count; i++) {
        boolean isHttpAnnotation = false;
        if (parameterAnnotations[i] != null) {
          isHttpAnnotation = processAnnotationsOnParameter(data, parameterAnnotations[i], i);
        }
        if (parameterTypes[i] == URI.class) {
          data.urlIndex(i);
        } else if (!isHttpAnnotation) {
          checkState(data.formParams().isEmpty(),
                     "Body parameters cannot be used with form parameters.");
          checkState(data.bodyIndex() == null, "Method has too many Body parameters: %s", method);
          data.bodyIndex(i);
          data.bodyType(Types.resolve(targetType, targetType, method.getGenericParameterTypes()[i]));
        }
      }

      if (data.headerMapIndex() != null) {
        checkState(Map.class.isAssignableFrom(parameterTypes[data.headerMapIndex()]),
                "HeaderMap parameter must be a Map: %s", parameterTypes[data.headerMapIndex()]);
      }

      if (data.queryMapIndex() != null) {
        checkState(Map.class.isAssignableFrom(parameterTypes[data.queryMapIndex()]),
                "QueryMap parameter must be a Map: %s", parameterTypes[data.queryMapIndex()]);
      }

      return data;
    }

    /**
     * Called by parseAndValidateMetadata twice, first on the declaring class, then on the
     * target type (unless they are the same).
     *
     * @param data       metadata collected so far relating to the current java method.
     * @param clz        the class to process
     */
    protected void processAnnotationOnClass(MethodMetadata data, Class<?> clz) {
    }

    /**
     * @param data       metadata collected so far relating to the current java method.
     * @param annotation annotations present on the current method annotation.
     * @param method     method currently being processed.
     */
    protected abstract void processAnnotationOnMethod(MethodMetadata data, Annotation annotation,
                                                      Method method);

    /**
     * @param data        metadata collected so far relating to the current java method.
     * @param annotations annotations present on the current parameter annotation.
     * @param paramIndex  if you find a name in {@code annotations}, call {@link
     *                    #nameParam(MethodMetadata, String, int)} with this as the last parameter.
     * @return true if you called {@link #nameParam(MethodMetadata, String, int)} after finding an
     * http-relevant annotation.
     */
    protected abstract boolean processAnnotationsOnParameter(MethodMetadata data,
                                                             Annotation[] annotations,
                                                             int paramIndex);


    protected Collection<String> addTemplatedParam(Collection<String> possiblyNull, String name) {
      if (possiblyNull == null) {
        possiblyNull = new ArrayList<String>();
      }
      possiblyNull.add(String.format("{%s}", name));
      return possiblyNull;
    }

    /**
     * links a parameter name to its index in the method signature.
     */
    protected void nameParam(MethodMetadata data, String name, int i) {
      Collection<String>
          names =
          data.indexToName().containsKey(i) ? data.indexToName().get(i) : new ArrayList<String>();
      names.add(name);
      data.indexToName().put(i, names);
    }
  }

  class Default extends BaseContract {
    @Override
    protected void processAnnotationOnClass(MethodMetadata data, Class<?> targetType) {
      if (targetType.isAnnotationPresent(Headers.class)) {
        String[] headersOnType = targetType.getAnnotation(Headers.class).value();
        checkState(headersOnType.length > 0, "Headers annotation was empty on type %s.",
                   targetType.getName());
        Map<String, Collection<String>> headers = toMap(headersOnType);
        headers.putAll(data.template().headers());
        data.template().headers(null); // to clear
        data.template().headers(headers);
      }
    }

    @Override
    protected void processAnnotationOnMethod(MethodMetadata data, Annotation methodAnnotation,
                                             Method method) {
      Class<? extends Annotation> annotationType = methodAnnotation.annotationType();
      if (annotationType == RequestLine.class) {
        String requestLine = RequestLine.class.cast(methodAnnotation).value();
        checkState(emptyToNull(requestLine) != null,
                   "RequestLine annotation was empty on method %s.", method.getName());
        if (requestLine.indexOf(' ') == -1) {
          checkState(requestLine.indexOf('/') == -1,
              "RequestLine annotation didn't start with an HTTP verb on method %s.",
              method.getName());
          data.template().method(requestLine);
          return;
        }
        data.template().method(requestLine.substring(0, requestLine.indexOf(' ')));
        if (requestLine.indexOf(' ') == requestLine.lastIndexOf(' ')) {
          // no HTTP version is ok
          data.template().append(requestLine.substring(requestLine.indexOf(' ') + 1));
        } else {
          // skip HTTP version
          data.template().append(
              requestLine.substring(requestLine.indexOf(' ') + 1, requestLine.lastIndexOf(' ')));
        }

        data.template().decodeSlash(RequestLine.class.cast(methodAnnotation).decodeSlash());

      } else if (annotationType == Body.class) {
        String body = Body.class.cast(methodAnnotation).value();
        checkState(emptyToNull(body) != null, "Body annotation was empty on method %s.",
                   method.getName());
        if (body.indexOf('{') == -1) {
          data.template().body(body);
        } else {
          data.template().bodyTemplate(body);
        }
      } else if (annotationType == Headers.class) {
        String[] headersOnMethod = Headers.class.cast(methodAnnotation).value();
        checkState(headersOnMethod.length > 0, "Headers annotation was empty on method %s.",
                   method.getName());
        data.template().headers(toMap(headersOnMethod));
      }
    }

    @Override
    protected boolean processAnnotationsOnParameter(MethodMetadata data, Annotation[] annotations,
                                                    int paramIndex) {
      boolean isHttpAnnotation = false;
      for (Annotation annotation : annotations) {
        Class<? extends Annotation> annotationType = annotation.annotationType();
        if (annotationType == Param.class) {
          String name = ((Param) annotation).value();
          checkState(emptyToNull(name) != null, "Param annotation was empty on param %s.",
              paramIndex);
          nameParam(data, name, paramIndex);
          if (annotationType == Param.class) {
            Class<? extends Param.Expander> expander = ((Param) annotation).expander();
            if (expander != Param.ToStringExpander.class) {
              data.indexToExpanderClass().put(paramIndex, expander);
            }
          }
          isHttpAnnotation = true;
          String varName = '{' + name + '}';
          if (data.template().url().indexOf(varName) == -1 &&
              !searchMapValuesContainsExact(data.template().queries(), varName) &&
              !searchMapValuesContainsSubstring(data.template().headers(), varName)) {
            data.formParams().add(name);
          }
        } else if (annotationType == QueryMap.class) {
          checkState(data.queryMapIndex() == null, "QueryMap annotation was present on multiple parameters.");
          data.queryMapIndex(paramIndex);
          data.queryMapEncoded(QueryMap.class.cast(annotation).encoded());
          isHttpAnnotation = true;
        } else if (annotationType == HeaderMap.class) {
          checkState(data.headerMapIndex() == null, "HeaderMap annotation was present on multiple parameters.");
          data.headerMapIndex(paramIndex);
          isHttpAnnotation = true;
        }
      }
      return isHttpAnnotation;
    }

    private static <K, V> boolean searchMapValuesContainsExact(Map<K, Collection<V>> map,
                                                               V search) {
      Collection<Collection<V>> values = map.values();
      if (values == null) {
        return false;
      }

      for (Collection<V> entry : values) {
        if (entry.contains(search)) {
          return true;
        }
      }

      return false;
    }

    private static <K, V> boolean searchMapValuesContainsSubstring(Map<K, Collection<String>> map,
                                                                   String search) {
      Collection<Collection<String>> values = map.values();
      if (values == null) {
        return false;
      }

      for (Collection<String> entry : values) {
        for (String value : entry) {
          if (value.indexOf(search) != -1) {
            return true;
          }
        }
      }

      return false;
    }

    private static Map<String, Collection<String>> toMap(String[] input) {
      Map<String, Collection<String>>
          result =
          new LinkedHashMap<String, Collection<String>>(input.length);
      for (String header : input) {
        int colon = header.indexOf(':');
        String name = header.substring(0, colon);
        if (!result.containsKey(name)) {
          result.put(name, new ArrayList<String>(1));
        }
        result.get(name).add(header.substring(colon + 2));
      }
      return result;
    }
  }
}

```

## Client.java

```java
package feign;
/**
 * 执行请求的客户端
 */
public interface Client {

  Response execute(Request request, Options options) throws IOException;

}

```

## Retryer.java

```java
package feign;

import static java.util.concurrent.TimeUnit.SECONDS;

/**
 * 重试器
 */
public interface Retryer extends Cloneable {

  /**
   * if retry is permitted, return (possibly after sleeping). Otherwise propagate the exception.
   */
  void continueOrPropagate(RetryableException e);

  Retryer clone();

  public static class Default implements Retryer {

    private final int maxAttempts;
    private final long period;
    private final long maxPeriod;
    int attempt;
    long sleptForMillis;

    public Default() {
      this(100, SECONDS.toMillis(1), 5);
    }

    public Default(long period, long maxPeriod, int maxAttempts) {
      this.period = period;
      this.maxPeriod = maxPeriod;
      this.maxAttempts = maxAttempts;
      this.attempt = 1;
    }

    // visible for testing;
    protected long currentTimeMillis() {
      return System.currentTimeMillis();
    }

    public void continueOrPropagate(RetryableException e) {
      if (attempt++ >= maxAttempts) {
        throw e;
      }

      long interval;
      if (e.retryAfter() != null) {
        interval = e.retryAfter().getTime() - currentTimeMillis();
        if (interval > maxPeriod) {
          interval = maxPeriod;
        }
        if (interval < 0) {
          return;
        }
      } else {
        interval = nextMaxInterval();
      }
      try {
        Thread.sleep(interval);
      } catch (InterruptedException ignored) {
        Thread.currentThread().interrupt();
      }
      sleptForMillis += interval;
    }

    /**
     * Calculates the time interval to a retry attempt. <br> The interval increases exponentially
     * with each attempt, at a rate of nextInterval *= 1.5 (where 1.5 is the backoff factor), to the
     * maximum interval.
     *
     * @return time in nanoseconds from now until the next attempt.
     */
    long nextMaxInterval() {
      long interval = (long) (period * Math.pow(1.5, attempt - 1));
      return interval > maxPeriod ? maxPeriod : interval;
    }

    @Override
    public Retryer clone() {
      return new Default(period, maxPeriod, maxAttempts);
    }
  }

  /**
   * Implementation that never retries request. It propagates the RetryableException.
   */
  Retryer NEVER_RETRY = new Retryer() {

    @Override
    public void continueOrPropagate(RetryableException e) {
      throw e;
    }

    @Override
    public Retryer clone() {
      return this;
    }
  };
}

```

## Encoder.java

```java
package feign.codec;
/**
 * 编码器
 */
public interface Encoder {

  Type MAP_STRING_WILDCARD = Util.MAP_STRING_WILDCARD;

  void encode(Object object, Type bodyType, RequestTemplate template) throws EncodeException;

  class Default implements Encoder {

    @Override
    public void encode(Object object, Type bodyType, RequestTemplate template) {
      if (bodyType == String.class) {
        template.body(object.toString());
      } else if (bodyType == byte[].class) {
        template.body((byte[]) object, null);
      } else if (object != null) {
        throw new EncodeException(
            format("%s is not a type supported by this encoder.", object.getClass()));
      }
    }
  }
}

```

## Decoder.java

```java
package feign.codec;
/**
 * 解码器
 */
public interface Decoder {

  Object decode(Response response, Type type) throws IOException, DecodeException, FeignException;

  public class Default extends StringDecoder {

    @Override
    public Object decode(Response response, Type type) throws IOException {
      if (response.status() == 404) return Util.emptyValueOf(type);
      if (response.body() == null) return null;
      if (byte[].class.equals(type)) {
        return Util.toByteArray(response.body().asInputStream());
      }
      return super.decode(response, type);
    }
  }
}

```

## ErrorDecoder.java

```java
package feign.codec;
/**
 * 错误解码器
 */
public interface ErrorDecoder {

  public Exception decode(String methodKey, Response response);

  public static class Default implements ErrorDecoder {

    private final RetryAfterDecoder retryAfterDecoder = new RetryAfterDecoder();

    @Override
    public Exception decode(String methodKey, Response response) {
      FeignException exception = errorStatus(methodKey, response);
      Date retryAfter = retryAfterDecoder.apply(firstOrNull(response.headers(), RETRY_AFTER));
      if (retryAfter != null) {
        return new RetryableException(exception.getMessage(), exception, retryAfter);
      }
      return exception;
    }

    private <T> T firstOrNull(Map<String, Collection<T>> map, String key) {
      if (map.containsKey(key) && !map.get(key).isEmpty()) {
        return map.get(key).iterator().next();
      }
      return null;
    }
  }

  /**
   * Decodes a {@link feign.Util#RETRY_AFTER} header into an absolute date, if possible. <br> See <a
   * href="https://tools.ietf.org/html/rfc2616#section-14.37">Retry-After format</a>
   */
  static class RetryAfterDecoder {

    static final DateFormat
        RFC822_FORMAT =
        new SimpleDateFormat("EEE, dd MMM yyyy HH:mm:ss 'GMT'", US);
    private final DateFormat rfc822Format;

    RetryAfterDecoder() {
      this(RFC822_FORMAT);
    }

    RetryAfterDecoder(DateFormat rfc822Format) {
      this.rfc822Format = checkNotNull(rfc822Format, "rfc822Format");
    }

    protected long currentTimeMillis() {
      return System.currentTimeMillis();
    }

    /**
     * returns a date that corresponds to the first time a request can be retried.
     *
     * @param retryAfter String in <a href="https://tools.ietf.org/html/rfc2616#section-14.37"
     *                   >Retry-After format</a>
     */
    public Date apply(String retryAfter) {
      if (retryAfter == null) {
        return null;
      }
      if (retryAfter.matches("^[0-9]+$")) {
        long deltaMillis = SECONDS.toMillis(Long.parseLong(retryAfter));
        return new Date(currentTimeMillis() + deltaMillis);
      }
      synchronized (rfc822Format) {
        try {
          return rfc822Format.parse(retryAfter);
        } catch (ParseException ignored) {
          return null;
        }
      }
    }
  }
}

```

## InvocationHandlerFactory.java

```java

package feign;
/**
 * 反射处理器工厂
 */
public interface InvocationHandlerFactory {

  InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch);

  interface MethodHandler {

    Object invoke(Object[] argv) throws Throwable;
  }

  static final class Default implements InvocationHandlerFactory {

    @Override
    public InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch) {
      return new ReflectiveFeign.FeignInvocationHandler(target, dispatch);
    }
  }
}

```

### DefaultMethodHandler.java

```java
package feign;
/**
 * 默认的方法处理器，直接调用方法
 */
@IgnoreJRERequirement
final class DefaultMethodHandler implements MethodHandler {
  // Uses Java 7 MethodHandle based reflection.  As default methods will only exist when
  // run on a Java 8 JVM this will not affect use on legacy JVMs.
  // When Feign upgrades to Java 7, remove the @IgnoreJRERequirement annotation.
  private final MethodHandle unboundHandle;

  // handle is effectively final after bindTo has been called.
  private MethodHandle handle;

  public DefaultMethodHandler(Method defaultMethod) {
    try {
      Class<?> declaringClass = defaultMethod.getDeclaringClass();
      Field field = Lookup.class.getDeclaredField("IMPL_LOOKUP");
      field.setAccessible(true);
      Lookup lookup = (Lookup) field.get(null);

      this.unboundHandle = lookup.unreflectSpecial(defaultMethod, declaringClass);
    } catch (NoSuchFieldException ex) {
      throw new IllegalStateException(ex);
    } catch (IllegalAccessException ex) {
      throw new IllegalStateException(ex);
    }
  }

  /**
   * Bind this handler to a proxy object.  After bound, DefaultMethodHandler#invoke will act as if it was called
   * on the proxy object.  Must be called once and only once for a given instance of DefaultMethodHandler
   */
  public void bindTo(Object proxy) {
    if(handle != null) {
      throw new IllegalStateException("Attempted to rebind a default method handler that was already bound");
    }
    handle = unboundHandle.bindTo(proxy);
  }

  /**
   * Invoke this method.  DefaultMethodHandler#bindTo must be called before the first
   * time invoke is called.
   */
  @Override
  public Object invoke(Object[] argv) throws Throwable {
    if(handle == null) {
      throw new IllegalStateException("Default method handler invoked before proxy has been bound.");
    }
    return handle.invokeWithArguments(argv);
  }
}

```

### SynchronousMethodHandler.java

```java
package feign;

final class SynchronousMethodHandler implements MethodHandler {

  private static final long MAX_RESPONSE_BUFFER_SIZE = 8192L;

  private final MethodMetadata metadata;
  private final Target<?> target;
  private final Client client;
  private final Retryer retryer;
  private final List<RequestInterceptor> requestInterceptors;
  private final Logger logger;
  private final Logger.Level logLevel;
  private final RequestTemplate.Factory buildTemplateFromArgs;
  private final Options options;
  private final Decoder decoder;
  private final ErrorDecoder errorDecoder;
  private final boolean decode404;

  private SynchronousMethodHandler(Target<?> target, Client client, Retryer retryer,
                                   List<RequestInterceptor> requestInterceptors, Logger logger,
                                   Logger.Level logLevel, MethodMetadata metadata,
                                   RequestTemplate.Factory buildTemplateFromArgs, Options options,
                                   Decoder decoder, ErrorDecoder errorDecoder, boolean decode404) {
    this.target = checkNotNull(target, "target");
    this.client = checkNotNull(client, "client for %s", target);
    this.retryer = checkNotNull(retryer, "retryer for %s", target);
    this.requestInterceptors =
        checkNotNull(requestInterceptors, "requestInterceptors for %s", target);
    this.logger = checkNotNull(logger, "logger for %s", target);
    this.logLevel = checkNotNull(logLevel, "logLevel for %s", target);
    this.metadata = checkNotNull(metadata, "metadata for %s", target);
    this.buildTemplateFromArgs = checkNotNull(buildTemplateFromArgs, "metadata for %s", target);
    this.options = checkNotNull(options, "options for %s", target);
    this.errorDecoder = checkNotNull(errorDecoder, "errorDecoder for %s", target);
    this.decoder = checkNotNull(decoder, "decoder for %s", target);
    this.decode404 = decode404;
  }

  @Override
  public Object invoke(Object[] argv) throws Throwable {
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Retryer retryer = this.retryer.clone();
    while (true) {
      try {
        return executeAndDecode(template);
      } catch (RetryableException e) {
        retryer.continueOrPropagate(e);
        if (logLevel != Logger.Level.NONE) {
          logger.logRetry(metadata.configKey(), logLevel);
        }
        continue;
      }
    }
  }

  Object executeAndDecode(RequestTemplate template) throws Throwable {
    Request request = targetRequest(template);

    if (logLevel != Logger.Level.NONE) {
      logger.logRequest(metadata.configKey(), logLevel, request);
    }

    Response response;
    long start = System.nanoTime();
    try {
      response = client.execute(request, options);
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
      }
      throw errorExecuting(request, e);
    }
    long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);

    boolean shouldClose = true;
    try {
      if (logLevel != Logger.Level.NONE) {
        response =
            logger.logAndRebufferResponse(metadata.configKey(), logLevel, response, elapsedTime);
      }
      if (Response.class == metadata.returnType()) {
        if (response.body() == null) {
          return response;
        }
        if (response.body().length() == null ||
                response.body().length() > MAX_RESPONSE_BUFFER_SIZE) {
          shouldClose = false;
          return response;
        }
        // Ensure the response body is disconnected
        byte[] bodyData = Util.toByteArray(response.body().asInputStream());
        return Response.create(response.status(), response.reason(), response.headers(), bodyData);
      }
      if (response.status() >= 200 && response.status() < 300) {
        if (void.class == metadata.returnType()) {
          return null;
        } else {
          return decode(response);
        }
      } else if (decode404 && response.status() == 404) {
        return decoder.decode(response, metadata.returnType());
      } else {
        throw errorDecoder.decode(metadata.configKey(), response);
      }
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime);
      }
      throw errorReading(request, response, e);
    } finally {
      if (shouldClose) {
        ensureClosed(response.body());
      }
    }
  }

  long elapsedTime(long start) {
    return TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);
  }

  Request targetRequest(RequestTemplate template) {
    for (RequestInterceptor interceptor : requestInterceptors) {
      interceptor.apply(template);
    }
    return target.apply(new RequestTemplate(template));
  }

  Object decode(Response response) throws Throwable {
    try {
      return decoder.decode(response, metadata.returnType());
    } catch (FeignException e) {
      throw e;
    } catch (RuntimeException e) {
      throw new DecodeException(e.getMessage(), e);
    }
  }

  static class Factory {

    private final Client client;
    private final Retryer retryer;
    private final List<RequestInterceptor> requestInterceptors;
    private final Logger logger;
    private final Logger.Level logLevel;
    private final boolean decode404;

    Factory(Client client, Retryer retryer, List<RequestInterceptor> requestInterceptors,
            Logger logger, Logger.Level logLevel, boolean decode404) {
      this.client = checkNotNull(client, "client");
      this.retryer = checkNotNull(retryer, "retryer");
      this.requestInterceptors = checkNotNull(requestInterceptors, "requestInterceptors");
      this.logger = checkNotNull(logger, "logger");
      this.logLevel = checkNotNull(logLevel, "logLevel");
      this.decode404 = decode404;
    }

    public MethodHandler create(Target<?> target, MethodMetadata md,
                                RequestTemplate.Factory buildTemplateFromArgs,
                                Options options, Decoder decoder, ErrorDecoder errorDecoder) {
      return new SynchronousMethodHandler(target, client, retryer, requestInterceptors, logger,
                                          logLevel, md, buildTemplateFromArgs, options, decoder,
                                          errorDecoder, decode404);
    }
  }
}

```



## Target.java

```java
package feign;
/**
 * @param <T> type of the interface this target applies to.
 */
public interface Target<T> {

  /* The type of the interface this target applies to. ex. {@code Route53}. */
  Class<T> type();

  /* configuration key associated with this target. For example, {@code route53}. */
  String name();

  /* base HTTP URL of the target. For example, {@code https://api/v2}. */
  String url();

  /**
   * Targets a template to this target, adding the {@link #url() base url} and any target-specific
   * headers or query parameters. <br> <br> For example: <br>
   * <pre>
   * public Request apply(RequestTemplate input) {
   *     input.insert(0, url());
   *     input.replaceHeader(&quot;X-Auth&quot;, currentToken);
   *     return input.asRequest();
   * }
   * </pre>
   * <br> <br><br><b>relationship to JAXRS 2.0</b><br> <br> This call is similar to {@code
   * javax.ws.rs.client.WebTarget.request()}, except that we expect transient, but necessary
   * decoration to be applied on invocation.
   */
  public Request apply(RequestTemplate input);

  public static class HardCodedTarget<T> implements Target<T> {

    private final Class<T> type;
    private final String name;
    private final String url;

    public HardCodedTarget(Class<T> type, String url) {
      this(type, url, url);
    }

    public HardCodedTarget(Class<T> type, String name, String url) {
      this.type = checkNotNull(type, "type");
      this.name = checkNotNull(emptyToNull(name), "name");
      this.url = checkNotNull(emptyToNull(url), "url");
    }

    @Override
    public Class<T> type() {
      return type;
    }

    @Override
    public String name() {
      return name;
    }

    @Override
    public String url() {
      return url;
    }

    /* no authentication or other special activity. just insert the url. */
    @Override
    public Request apply(RequestTemplate input) {
      if (input.url().indexOf("http") != 0) {
        input.insert(0, url());
      }
      return input.request();
    }

    @Override
    public boolean equals(Object obj) {
      if (obj instanceof HardCodedTarget) {
        HardCodedTarget<?> other = (HardCodedTarget) obj;
        return type.equals(other.type)
               && name.equals(other.name)
               && url.equals(other.url);
      }
      return false;
    }

    @Override
    public int hashCode() {
      int result = 17;
      result = 31 * result + type.hashCode();
      result = 31 * result + name.hashCode();
      result = 31 * result + url.hashCode();
      return result;
    }

    @Override
    public String toString() {
      if (name.equals(url)) {
        return "HardCodedTarget(type=" + type.getSimpleName() + ", url=" + url + ")";
      }
      return "HardCodedTarget(type=" + type.getSimpleName() + ", name=" + name + ", url=" + url
             + ")";
    }
  }

  public static final class EmptyTarget<T> implements Target<T> {

    private final Class<T> type;
    private final String name;

    EmptyTarget(Class<T> type, String name) {
      this.type = checkNotNull(type, "type");
      this.name = checkNotNull(emptyToNull(name), "name");
    }

    public static <T> EmptyTarget<T> create(Class<T> type) {
      return new EmptyTarget<T>(type, "empty:" + type.getSimpleName());
    }

    public static <T> EmptyTarget<T> create(Class<T> type, String name) {
      return new EmptyTarget<T>(type, name);
    }

    @Override
    public Class<T> type() {
      return type;
    }

    @Override
    public String name() {
      return name;
    }

    @Override
    public String url() {
      throw new UnsupportedOperationException("Empty targets don't have URLs");
    }

    @Override
    public Request apply(RequestTemplate input) {
      if (input.url().indexOf("http") != 0) {
        throw new UnsupportedOperationException(
            "Request with non-absolute URL not supported with empty target");
      }
      return input.request();
    }

    @Override
    public boolean equals(Object obj) {
      if (obj instanceof EmptyTarget) {
        EmptyTarget<?> other = (EmptyTarget) obj;
        return type.equals(other.type)
               && name.equals(other.name);
      }
      return false;
    }

    @Override
    public int hashCode() {
      int result = 17;
      result = 31 * result + type.hashCode();
      result = 31 * result + name.hashCode();
      return result;
    }

    @Override
    public String toString() {
      if (name.equals("empty:" + type.getSimpleName())) {
        return "EmptyTarget(type=" + type.getSimpleName() + ")";
      }
      return "EmptyTarget(type=" + type.getSimpleName() + ", name=" + name + ")";
    }
  }
}

```



## Feign.java

```java
package feign;
/**
 * Feign的目的是简化REST开发，这个类的实现是一个生成HTTPAPI的工厂
 */
public abstract class Feign {

  public static Builder builder() {
    return new Builder();
  }

  public static String configKey(Class targetType, Method method) {
    StringBuilder builder = new StringBuilder();
    builder.append(targetType.getSimpleName());
    builder.append('#').append(method.getName()).append('(');
    for (Type param : method.getGenericParameterTypes()) {
      param = Types.resolve(targetType, targetType, param);
      builder.append(Types.getRawType(param).getSimpleName()).append(',');
    }
    if (method.getParameterTypes().length > 0) {
      builder.deleteCharAt(builder.length() - 1);
    }
    return builder.append(')').toString();
  }

  public abstract <T> T newInstance(Target<T> target);

  public static class Builder {

    private final List<RequestInterceptor> requestInterceptors =
        new ArrayList<RequestInterceptor>();
    private Logger.Level logLevel = Logger.Level.NONE;
    private Contract contract = new Contract.Default();
    private Client client = new Client.Default(null, null);
    private Retryer retryer = new Retryer.Default();
    private Logger logger = new NoOpLogger();
    private Encoder encoder = new Encoder.Default();
    private Decoder decoder = new Decoder.Default();
    private ErrorDecoder errorDecoder = new ErrorDecoder.Default();
    private Options options = new Options();
    private InvocationHandlerFactory invocationHandlerFactory =
        new InvocationHandlerFactory.Default();
    private boolean decode404;

    public Builder logLevel(Logger.Level logLevel) {
      this.logLevel = logLevel;
      return this;
    }

    public Builder contract(Contract contract) {
      this.contract = contract;
      return this;
    }

    public Builder client(Client client) {
      this.client = client;
      return this;
    }

    public Builder retryer(Retryer retryer) {
      this.retryer = retryer;
      return this;
    }

    public Builder logger(Logger logger) {
      this.logger = logger;
      return this;
    }

    public Builder encoder(Encoder encoder) {
      this.encoder = encoder;
      return this;
    }

    public Builder decoder(Decoder decoder) {
      this.decoder = decoder;
      return this;
    }

    public Builder decode404() {
      this.decode404 = true;
      return this;
    }

    public Builder errorDecoder(ErrorDecoder errorDecoder) {
      this.errorDecoder = errorDecoder;
      return this;
    }

    public Builder options(Options options) {
      this.options = options;
      return this;
    }

    public Builder requestInterceptor(RequestInterceptor requestInterceptor) {
      this.requestInterceptors.add(requestInterceptor);
      return this;
    }

    public Builder requestInterceptors(Iterable<RequestInterceptor> requestInterceptors) {
      this.requestInterceptors.clear();
      for (RequestInterceptor requestInterceptor : requestInterceptors) {
        this.requestInterceptors.add(requestInterceptor);
      }
      return this;
    }

    public Builder invocationHandlerFactory(InvocationHandlerFactory invocationHandlerFactory) {
      this.invocationHandlerFactory = invocationHandlerFactory;
      return this;
    }

    public <T> T target(Class<T> apiType, String url) {
      return target(new HardCodedTarget<T>(apiType, url));
    }

    public <T> T target(Target<T> target) {
      return build().newInstance(target);
    }

    public Feign build() {
      SynchronousMethodHandler.Factory synchronousMethodHandlerFactory =
          new SynchronousMethodHandler.Factory(client, retryer, requestInterceptors, logger,
                                               logLevel, decode404);
      ParseHandlersByName handlersByName =
          new ParseHandlersByName(contract, options, encoder, decoder,
                                  errorDecoder, synchronousMethodHandlerFactory);
      return new ReflectiveFeign(handlersByName, invocationHandlerFactory);
    }
  }
}
```

### ReflectiveFeign.java

```java
package feign;

public class ReflectiveFeign extends Feign {

  private final ParseHandlersByName targetToHandlersByName;
  private final InvocationHandlerFactory factory;

  ReflectiveFeign(ParseHandlersByName targetToHandlersByName, InvocationHandlerFactory factory) {
    this.targetToHandlersByName = targetToHandlersByName;
    this.factory = factory;
  }

  @SuppressWarnings("unchecked")
  @Override
  public <T> T newInstance(Target<T> target) {
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

    for (Method method : target.type().getMethods()) {
      if (method.getDeclaringClass() == Object.class) {
        continue;
      } else if(Util.isDefault(method)) {
        DefaultMethodHandler handler = new DefaultMethodHandler(method);
        defaultMethodHandlers.add(handler);
        methodToHandler.put(method, handler);
      } else {
        methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
      }
    }
    InvocationHandler handler = factory.create(target, methodToHandler);
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[]{target.type()}, handler);

    for(DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
      defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
  }

  static class FeignInvocationHandler implements InvocationHandler {

    private final Target target;
    private final Map<Method, MethodHandler> dispatch;

    FeignInvocationHandler(Target target, Map<Method, MethodHandler> dispatch) {
      this.target = checkNotNull(target, "target");
      this.dispatch = checkNotNull(dispatch, "dispatch for %s", target);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      if ("equals".equals(method.getName())) {
        try {
          Object
              otherHandler =
              args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
          return equals(otherHandler);
        } catch (IllegalArgumentException e) {
          return false;
        }
      } else if ("hashCode".equals(method.getName())) {
        return hashCode();
      } else if ("toString".equals(method.getName())) {
        return toString();
      }
      return dispatch.get(method).invoke(args);
    }

    @Override
    public boolean equals(Object obj) {
      if (obj instanceof FeignInvocationHandler) {
        FeignInvocationHandler other = (FeignInvocationHandler) obj;
        return target.equals(other.target);
      }
      return false;
    }

    @Override
    public int hashCode() {
      return target.hashCode();
    }

    @Override
    public String toString() {
      return target.toString();
    }
  }

  static final class ParseHandlersByName {

    private final Contract contract;
    private final Options options;
    private final Encoder encoder;
    private final Decoder decoder;
    private final ErrorDecoder errorDecoder;
    private final SynchronousMethodHandler.Factory factory;

    ParseHandlersByName(Contract contract, Options options, Encoder encoder, Decoder decoder,
                        ErrorDecoder errorDecoder, SynchronousMethodHandler.Factory factory) {
      this.contract = contract;
      this.options = options;
      this.factory = factory;
      this.errorDecoder = errorDecoder;
      this.encoder = checkNotNull(encoder, "encoder");
      this.decoder = checkNotNull(decoder, "decoder");
    }

    public Map<String, MethodHandler> apply(Target key) {
      List<MethodMetadata> metadata = contract.parseAndValidatateMetadata(key.type());
      Map<String, MethodHandler> result = new LinkedHashMap<String, MethodHandler>();
      for (MethodMetadata md : metadata) {
        BuildTemplateByResolvingArgs buildTemplate;
        if (!md.formParams().isEmpty() && md.template().bodyTemplate() == null) {
          buildTemplate = new BuildFormEncodedTemplateFromArgs(md, encoder);
        } else if (md.bodyIndex() != null) {
          buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder);
        } else {
          buildTemplate = new BuildTemplateByResolvingArgs(md);
        }
        result.put(md.configKey(),
                   factory.create(key, md, buildTemplate, options, decoder, errorDecoder));
      }
      return result;
    }
  }

  private static class BuildTemplateByResolvingArgs implements RequestTemplate.Factory {

    protected final MethodMetadata metadata;
    private final Map<Integer, Expander> indexToExpander = new LinkedHashMap<Integer, Expander>();

    private BuildTemplateByResolvingArgs(MethodMetadata metadata) {
      this.metadata = metadata;
      if (metadata.indexToExpander() != null) {
        indexToExpander.putAll(metadata.indexToExpander());
        return;
      }
      if (metadata.indexToExpanderClass().isEmpty()) {
        return;
      }
      for (Entry<Integer, Class<? extends Expander>> indexToExpanderClass : metadata
          .indexToExpanderClass().entrySet()) {
        try {
          indexToExpander
              .put(indexToExpanderClass.getKey(), indexToExpanderClass.getValue().newInstance());
        } catch (InstantiationException e) {
          throw new IllegalStateException(e);
        } catch (IllegalAccessException e) {
          throw new IllegalStateException(e);
        }
      }
    }

    @Override
    public RequestTemplate create(Object[] argv) {
      RequestTemplate mutable = new RequestTemplate(metadata.template());
      if (metadata.urlIndex() != null) {
        int urlIndex = metadata.urlIndex();
        checkArgument(argv[urlIndex] != null, "URI parameter %s was null", urlIndex);
        mutable.insert(0, String.valueOf(argv[urlIndex]));
      }
      Map<String, Object> varBuilder = new LinkedHashMap<String, Object>();
      for (Entry<Integer, Collection<String>> entry : metadata.indexToName().entrySet()) {
        int i = entry.getKey();
        Object value = argv[entry.getKey()];
        if (value != null) { // Null values are skipped.
          if (indexToExpander.containsKey(i)) {
            value = expandElements(indexToExpander.get(i), value);
          }
          for (String name : entry.getValue()) {
            varBuilder.put(name, value);
          }
        }
      }

      RequestTemplate template = resolve(argv, mutable, varBuilder);
      if (metadata.queryMapIndex() != null) {
        // add query map parameters after initial resolve so that they take
        // precedence over any predefined values
        template = addQueryMapQueryParameters(argv, template);
      }

      if (metadata.headerMapIndex() != null) {
        template = addHeaderMapHeaders(argv, template);
      }

      return template;
    }

    private Object expandElements(Expander expander, Object value) {
      if (value instanceof Iterable) {
        return expandIterable(expander, (Iterable) value);
      }
      return expander.expand(value);
    }

    private List<String> expandIterable(Expander expander, Iterable value) {
      List<String> values = new ArrayList<String>();
      for (Object element : (Iterable) value) {
        if (element!=null) {
          values.add(expander.expand(element));
        }
      }
      return values;
    }

    @SuppressWarnings("unchecked")
    private RequestTemplate addHeaderMapHeaders(Object[] argv, RequestTemplate mutable) {
      Map<Object, Object> headerMap = (Map<Object, Object>) argv[metadata.headerMapIndex()];
      for (Entry<Object, Object> currEntry : headerMap.entrySet()) {
        checkState(currEntry.getKey().getClass() == String.class, "HeaderMap key must be a String: %s", currEntry.getKey());

        Collection<String> values = new ArrayList<String>();

        Object currValue = currEntry.getValue();
        if (currValue instanceof Iterable<?>) {
          Iterator<?> iter = ((Iterable<?>) currValue).iterator();
          while (iter.hasNext()) {
            Object nextObject = iter.next();
            values.add(nextObject == null ? null : nextObject.toString());
          }
        } else {
          values.add(currValue == null ? null : currValue.toString());
        }

        mutable.header((String) currEntry.getKey(), values);
      }
      return mutable;
    }

    @SuppressWarnings("unchecked")
    private RequestTemplate addQueryMapQueryParameters(Object[] argv, RequestTemplate mutable) {
      Map<Object, Object> queryMap = (Map<Object, Object>) argv[metadata.queryMapIndex()];
      for (Entry<Object, Object> currEntry : queryMap.entrySet()) {
        checkState(currEntry.getKey().getClass() == String.class, "QueryMap key must be a String: %s", currEntry.getKey());

        Collection<String> values = new ArrayList<String>();

        Object currValue = currEntry.getValue();
        if (currValue instanceof Iterable<?>) {
          Iterator<?> iter = ((Iterable<?>) currValue).iterator();
          while (iter.hasNext()) {
            Object nextObject = iter.next();
            values.add(nextObject == null ? null : nextObject.toString());
          }
        } else {
          values.add(currValue == null ? null : currValue.toString());
        }

        mutable.query(metadata.queryMapEncoded(), (String) currEntry.getKey(), values);
      }
      return mutable;
    }

    protected RequestTemplate resolve(Object[] argv, RequestTemplate mutable,
                                      Map<String, Object> variables) {
      return mutable.resolve(variables);
    }
  }

  private static class BuildFormEncodedTemplateFromArgs extends BuildTemplateByResolvingArgs {

    private final Encoder encoder;

    private BuildFormEncodedTemplateFromArgs(MethodMetadata metadata, Encoder encoder) {
      super(metadata);
      this.encoder = encoder;
    }

    @Override
    protected RequestTemplate resolve(Object[] argv, RequestTemplate mutable,
                                      Map<String, Object> variables) {
      Map<String, Object> formVariables = new LinkedHashMap<String, Object>();
      for (Entry<String, Object> entry : variables.entrySet()) {
        if (metadata.formParams().contains(entry.getKey())) {
          formVariables.put(entry.getKey(), entry.getValue());
        }
      }
      try {
        encoder.encode(formVariables, Encoder.MAP_STRING_WILDCARD, mutable);
      } catch (EncodeException e) {
        throw e;
      } catch (RuntimeException e) {
        throw new EncodeException(e.getMessage(), e);
      }
      return super.resolve(argv, mutable, variables);
    }
  }

  private static class BuildEncodedTemplateFromArgs extends BuildTemplateByResolvingArgs {

    private final Encoder encoder;

    private BuildEncodedTemplateFromArgs(MethodMetadata metadata, Encoder encoder) {
      super(metadata);
      this.encoder = encoder;
    }

    @Override
    protected RequestTemplate resolve(Object[] argv, RequestTemplate mutable,
                                      Map<String, Object> variables) {
      Object body = argv[metadata.bodyIndex()];
      checkArgument(body != null, "Body parameter %s was null", metadata.bodyIndex());
      try {
        encoder.encode(body, metadata.bodyType(), mutable);
      } catch (EncodeException e) {
        throw e;
      } catch (RuntimeException e) {
        throw new EncodeException(e.getMessage(), e);
      }
      return super.resolve(argv, mutable, variables);
    }
  }
}
```



# end

