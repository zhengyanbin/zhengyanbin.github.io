---
path: prevent-xss-with-antisamy-in-java
title: 使用antisamy防止xss注入
date: 2017-06-10 10:37:11
updated: 2017-06-10 10:37:11
categories:
- Java
tags:
- java
- xss
- antisamy
---

Java中处理xss的方式有多种，一般使用html entity转义，校验输入，清理有危害代码片段等方法。
+ 转义字符串长度会发生变化，可能影响到持久化操作。如果页面中不允许编辑富文本，可以采用此方式；相反，这种方式会影响html展示。
+ 校验输入可以在数据提交前防止用户添加特殊字符，但不是主要防御方法，只是有助于减少xss漏洞。
+ 清理是一种强有力的防御措施，清除可能有害的标记，将不可接受的用户输入更改为可接受的格式，以确保接收到的数据不会对用户以及数据库造成损害。在允许使用html的站点上，这种方式效果较好。

实际处理中，一般采用以上几种方式组合使用，但仍不能涵盖所有xss攻击，安全测试必不可少！
下面就清理有害代码的来做一次实践，利用了antisamy来完成有害代码的清理，jsoup等其他工具也可以做到。

1. 添加maven依赖
	``` xml
	<dependency>
		<groupId>org.owasp.antisamy</groupId>
		<artifactId>antisamy</artifactId>
		<version>1.5.8</version>
	</dependency>
	```
1. 根据antisamy api编写相关工具类
	``` java
	import java.io.IOException;

	import org.owasp.esapi.ESAPI;
	import org.owasp.validator.html.AntiSamy;
	import org.owasp.validator.html.CleanResults;
	import org.owasp.validator.html.Policy;
	import org.owasp.validator.html.PolicyException;
	import org.owasp.validator.html.ScanException;
	import org.slf4j.Logger;
	import org.slf4j.LoggerFactory;
	import org.springframework.core.io.DefaultResourceLoader;
	import org.springframework.core.io.Resource;
	import org.springframework.core.io.ResourceLoader;

	public class XssUtils {

		private final static Logger logger = LoggerFactory.getLogger(XssUtils.class);

		private static Policy policy = null; 

		static{ 
			ResourceLoader resourceLoader = new DefaultResourceLoader();
			//加载规则文件，antisamy提供了多份文件供参考
			Resource resource = resourceLoader.getResource("classpath:/antisamy.xml"); 
			 try { 
				policy = Policy.getInstance(resource.getURL().getPath()); 
			} catch (PolicyException e) { 
				e.printStackTrace(); 
			} catch (IOException e) {
				e.printStackTrace();
			} 
		} 

		public static String encode(String value){
			if(value == null || value.length() == 0) {
				return value;
			}
			value = ESAPI.encoder().canonicalize(value);
			return ESAPI.encoder().encodeForHTML(value);
		}

		public static String[] encode(String[] values){
			if(values == null || values.length == 0){
				return values;
			}
			int len = values.length;
			String[] _values = new String[len];
			for(int i = 0; i < len; i++){
				_values[i] = encode(values[i]);
			}
			return _values;
		}

		public static String clean(String value){
			AntiSamy antiSamy = new AntiSamy(); 
			try { 
				final CleanResults cr = antiSamy.scan(value, policy); 
				return cr.getCleanHTML(); 
			} catch (ScanException | PolicyException e) { 
				logger.error("invoke xss clean error", e);
			} 
			return value;
		}

		public static String[] clean(String[] values){
			if(values == null || values.length == 0){
				return values;
			}
			int len = values.length;
			String[] _values = new String[len];
			for(int i = 0; i < len; i++){
				_values[i] = clean(values[i]);
			}
			return _values;
		}

	}
	```
1. 使用filter处理HttpServletRequest，依次编写以下代码

	1. 自定义HttpServletRequestWrapper
		``` java
		import java.util.HashMap;
		import java.util.Map;

		import javax.servlet.http.HttpServletRequest;
		import javax.servlet.http.HttpServletRequestWrapper;

		import com.sw.busi.utils.XssUtils;

		public class XssCleanHttpServletRequestWrapper extends HttpServletRequestWrapper {

			public XssCleanHttpServletRequestWrapper(HttpServletRequest request) {
				super(request);
			}

			public String getQueryString() {
				String value = super.getQueryString();
				if (value != null) {
					value = XssUtils.clean(value);
				}
				return value;
			}

			/**
			 * 覆盖getParameter方法，将参数名和参数值都做xss过滤。<br/>
			 * 如果需要获得原始的值，则通过super.getParameterValues(name)来获取<br/>
			 * getParameterNames,getParameterValues和getParameterMap也可能需要覆盖
			 */
			public String getParameter(String name) {
				String value = super.getParameter(XssUtils.clean(name));
				if (value != null) {
					value = XssUtils.clean(value);
				}
				return value;
			}

			public String[] getParameterValues(String name) {
				String[]parameters=super.getParameterValues(name);
				if (parameters==null||parameters.length == 0) {
					return null;
				}
				for (int i = 0; i < parameters.length; i++) {
					parameters[i] = XssUtils.clean(parameters[i]);
				}
				return parameters;
			}

			public Map<String, String[]> getParameterMap() {
				Map<String, String[]> params = super.getParameterMap();
				if (params==null || params.size() == 0) {
					return params;
				}
				int capacity = (int) ((float) params.size() / 0.75F + 1.0F);
				Map<String, String[]> _params = new HashMap<String, String[]>(capacity);
				for (Map.Entry<String, String[]> e : params.entrySet()) {
					String key = e.getKey();
					String[] values = e.getValue();
					for (int i = 0; i < values.length; i++) {
						values[i] = XssUtils.clean(values[i]);
					}
					_params.put(key, values);
				}
				return _params;
			}

			/**
			 * 覆盖getHeader方法，将参数名和参数值都做xss过滤。<br/>
			 * 如果需要获得原始的值，则通过super.getHeaders(name)来获取<br/> getHeaderNames 也可能需要覆盖
			 */
			public String getHeader(String name) {

				String value = super.getHeader(XssUtils.clean(name));
				if (value != null) {
					value = XssUtils.clean(value);
				}
				return value;
			}

		}
		```


	1. 自定义filter
		``` java
		import java.io.IOException;

		import javax.servlet.Filter;
		import javax.servlet.FilterChain;
		import javax.servlet.FilterConfig;
		import javax.servlet.ServletException;
		import javax.servlet.ServletRequest;
		import javax.servlet.ServletResponse;
		import javax.servlet.http.HttpServletRequest;

		public class XssFilter implements Filter {

			protected FilterConfig filterConfig = null;

			public void init(FilterConfig filterConfig) throws ServletException {
				this.filterConfig = filterConfig;
			}

			public void destroy() {
				this.filterConfig = null;
			}

			public void doFilter(ServletRequest request, ServletResponse response,FilterChain chain) throws IOException, ServletException {
				chain.doFilter(new XssCleanHttpServletRequestWrapper((HttpServletRequest) request), response);
			}
		}
		```

1. 若使用spring mvc，且使用了@RequestBody的方式绑定参数（spring内部封装了httpservletrequest的装配器，使用了getInputStream的方式获取请求体，并转换为对应的实体），通过filter的方式就无法满足了，笔者对spring mvc的序列化与反序列化进行了修改，这里以jackson为示例，编写代码。
	1. 为MappingJackson2HttpMessageConverter的objectMapper注入自定义的实例对象
	1. 编写自定义ObjectMapper的实现
		``` java
		import com.fasterxml.jackson.databind.ObjectMapper;
		import com.fasterxml.jackson.databind.module.SimpleModule;
		import com.sw.busi.json.jackson.databind.deser.StringDeserializer;
		import com.sw.busi.json.jackson.databind.ser.StringSerializer;

		public class CustomObjectMapper extends ObjectMapper {
			private static final long serialVersionUID = -8543006375974774016L;

			public CustomObjectMapper(){
				SimpleModule simpleModule = new SimpleModule();
				//序列化与反序列化字符串，使用了xss clean
				simpleModule.addSerializer(String.class, StringSerializer.instance);
				simpleModule.addDeserializer(String.class, StringDeserializer.instance);

				this.registerModule(simpleModule);
			}
		}
		```
	1. 编写自定义StringSerializer  
		``` java
		import java.io.IOException;
		import java.lang.reflect.Type;

		import com.fasterxml.jackson.core.*;

		import com.fasterxml.jackson.databind.JavaType;
		import com.fasterxml.jackson.databind.JsonMappingException;
		import com.fasterxml.jackson.databind.JsonNode;
		import com.fasterxml.jackson.databind.SerializerProvider;
		import com.fasterxml.jackson.databind.jsonFormatVisitors.JsonFormatVisitorWrapper;
		import com.fasterxml.jackson.databind.ser.std.NonTypedScalarSerializerBase;
		import com.sw.busi.utils.XssUtils;

		/**
		 * This is the special serializer for regular {@link java.lang.String}s.
		 *<p>
		 * Since this is one of "native" types, no type information is ever
		 * included on serialization (unlike for most scalar types as of 1.5)
		 */
		public final class StringSerializer extends NonTypedScalarSerializerBase<String> {
			private static final long serialVersionUID = 1L;

			public static final StringSerializer instance = new StringSerializer();

			public StringSerializer() {
				super(String.class);
			}

			/**
			 * For Strings, both null and Empty String qualify for emptiness.
			 */
			@Override
			@Deprecated
			public boolean isEmpty(String value) {
				return (value == null) || (value.length() == 0);
			}

			@Override
			public boolean isEmpty(SerializerProvider prov, String value) {
				return (value == null) || (value.length() == 0);
			}

			@Override
			public void serialize(String value, JsonGenerator jgen, SerializerProvider provider) throws IOException {
				jgen.writeString(XssUtils.clean(value));
			}

			@Override
			public JsonNode getSchema(SerializerProvider provider, Type typeHint) {
				return createSchemaNode("string", true);
			}

			@Override
			public void acceptJsonFormatVisitor(JsonFormatVisitorWrapper visitor, JavaType typeHint)
					throws JsonMappingException {
				if (visitor != null)
					visitor.expectStringFormat(typeHint);
			}
		}
		```
	1. 编写StringDeserializer
		``` java
		import java.io.IOException;

		import com.fasterxml.jackson.core.Base64Variants;
		import com.fasterxml.jackson.core.JsonParser;
		import com.fasterxml.jackson.core.JsonToken;
		import com.fasterxml.jackson.databind.DeserializationContext;
		import com.fasterxml.jackson.databind.DeserializationFeature;
		import com.fasterxml.jackson.databind.deser.std.StdScalarDeserializer;
		import com.fasterxml.jackson.databind.jsontype.TypeDeserializer;

		import com.sw.busi.utils.XssUtils;

		//去掉注解，防止自定义反序列化器被认为是标准实现，从而反序列化String[] collection<String> Map<*, String> 无效
		// com.fasterxml.jackson.databind.util.ClassUtil.isJacksonStdImpl(Object)
		//@JacksonStdImpl
		public class StringDeserializer extends StdScalarDeserializer<String> {
			private static final long serialVersionUID = 1L;

			public final static StringDeserializer instance = new StringDeserializer();

			public StringDeserializer() {
				super(String.class);
			}

			// since 2.6, slightly faster lookups for this very common type
			@Override
			public boolean isCachable() {
				return true;
			}

			public String deserialize(JsonParser jp, DeserializationContext ctxt) throws IOException {
				JsonToken curr = jp.getCurrentToken();
				if (curr == JsonToken.VALUE_STRING) {
					return XssUtils.clean(jp.getText());
				}

				// Issue#381
				if (curr == JsonToken.START_ARRAY && ctxt.isEnabled(DeserializationFeature.UNWRAP_SINGLE_VALUE_ARRAYS)) {
					jp.nextToken();
					final String parsed = _parseString(jp, ctxt);
					if (jp.nextToken() != JsonToken.END_ARRAY) {
						throw ctxt.wrongTokenException(jp, JsonToken.END_ARRAY,
								"Attempted to unwrap single value array for single 'String' value but there was more than a single value in the array");
					}
					return XssUtils.clean(parsed);
				}
				// [JACKSON-330]: need to gracefully handle byte[] data, as base64
				if (curr == JsonToken.VALUE_EMBEDDED_OBJECT) {
					Object ob = jp.getEmbeddedObject();
					if (ob == null) {
						return null;
					}
					if (ob instanceof byte[]) {
						return XssUtils.clean(Base64Variants.getDefaultVariant().encode((byte[]) ob, false));
					}
					// otherwise, try conversion using toString()...
					return XssUtils.clean(ob.toString());
				}
				// allow coercions for other scalar types
				String text = jp.getValueAsString();
				if (text != null) {
					return XssUtils.clean(text);
				}
				throw ctxt.mappingException(_valueClass, curr);
			}

			// Since we can never have type info ("natural type"; String, Boolean,
			// Integer, Double):
			// (is it an error to even call this version?)
			@Override
			public String deserializeWithType(JsonParser p, DeserializationContext ctxt, TypeDeserializer typeDeserializer)
					throws IOException {
				return deserialize(p, ctxt);
			}
		}
		```
