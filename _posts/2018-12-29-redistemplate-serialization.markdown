---
layout:     post
title:      "RedisTemplate序列化StringRedisSerializer只能支持String的坑"
subtitle:   "RedisTemplate StringRedisSerializer"
date:       2018-12-29 08:12:00
author:     "XYZ"
header-img: "img/article-bg1.jpg"
tags:
    - 工作笔记
---
* 项目使用了spring-data-redis包的RedisTemplate类进行redis操作，在配置value的序列化类使用了StringRedisSerializer，如下：
```
private RedisTemplate<String, Object> buildRedisTemplate(JedisConnectionFactory connectionFactory) {
       RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
       redisTemplate.setKeySerializer(new StringRedisSerializer());
       redisTemplate.setValueSerializer(new StringRedisSerializer());
       redisTemplate.setHashKeySerializer(new StringRedisSerializer());
       redisTemplate.setHashValueSerializer(new Jackson2JsonRedisSerializer<>(Object.class));
       redisTemplate.setConnectionFactory(connectionFactory);
       return redisTemplate;
   }
```

redisTemplate.setValueSerializer(new StringRedisSerializer());
* StringRedisSerializer只能存储类型为String的value，当存储其他类型的数据，如double，则无法将value保存到redis，必须转为String。

```
Double value = 12.0;
BoundValueOperations<String, Object> valOps = redisTemplate.boundValueOps(key);
valOps.set(value);

Double value = 12.0;
BoundValueOperations<String, Object> valOps = redisTemplate.boundValueOps(key);
valOps.set(value.toString());
```
StringRedisSerializer源码如下：大概意思只能在String和byte[]之间序列化和反序列化。
```
/**
 * Simple String to byte[] (and back) serializer. Converts Strings into bytes and vice-versa using the specified charset
 * (by default UTF-8).
 * <p>
 * Useful when the interaction with the Redis happens mainly through Strings.
 * <p>
 * Does not perform any null conversion since empty strings are valid keys/values.
 * 
 * @author Costin Leau
 */
public class StringRedisSerializer implements RedisSerializer<String> {

	private final Charset charset;

	public StringRedisSerializer() {
		this(Charset.forName("UTF8"));
	}

	public StringRedisSerializer(Charset charset) {
		Assert.notNull(charset);
		this.charset = charset;
	}

	public String deserialize(byte[] bytes) {
		return (bytes == null ? null : new String(bytes, charset));
	}

	public byte[] serialize(String string) {
		return (string == null ? null : string.getBytes(charset));
	}
}
```
* 故需要改成Jackson2JsonRedisSerializer，才能存储Double类型的数据。如下：
```
redisTemplate.setValueSerializer(new Jackson2JsonRedisSerializer<>(Object.class));
```
但是马上报错了：
```
org.springframework.data.redis.serializer.SerializationException: Could not read JSON: Unrecognized token 'NIO': was expecting 'null', 'true', 'false' or NaN
 at [Source: [B@6bb7cce7; line: 1, column: 7]; nested exception is com.fasterxml.jackson.core.JsonParseException: Unrecognized token 'NIO': was expecting 'null', 'true', 'false' or NaN
 at [Source: [B@6bb7cce7; line: 1, column: 7]
	at org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer.deserialize(Jackson2JsonRedisSerializer.java:73)
	at org.springframework.data.redis.serializer.SerializationUtils.deserializeValues(SerializationUtils.java:50)
	at org.springframework.data.redis.serializer.SerializationUtils.deserialize(SerializationUtils.java:58)
	at org.springframework.data.redis.core.AbstractOperations.deserializeValues(AbstractOperations.java:202)
	at org.springframework.data.redis.core.DefaultSetOperations.members(DefaultSetOperations.java:134)
	at org.springframework.data.redis.core.DefaultBoundSetOperations.members(DefaultBoundSetOperations.java:91)
```
异常大概的意思：换成后无法将之前序列化保存的数据反序列化出来了。具体为如下调用members获取set的数据时，无法反序列化出来，坑就在这里，

```
写入set：
BoundSetOperations<String, Object> setOps = redisTemplate.boundSetOps(key);
setOps.add(symbol);
读出set：
BoundSetOperations<String, Object> setOps = redisTemplate.boundSetOps(key);
return setOps.members();
```

故在设计之初要使用：
```
redisTemplate.setValueSerializer(new Jackson2JsonRedisSerializer<>(Object.class));
```
才能支持多种数据类型的存储，避免存储了一些数据之后再来换，就需要对历史数据进行处理了，否则无法进行反序列化。
Jackson2JsonRedisSerializer的源码：使用FastJSON来进行序列化和反序列化。
```
/**
 * {@link RedisSerializer} that can read and write JSON using <a
 * href="https://github.com/FasterXML/jackson-core">Jackson's</a> and <a
 * href="https://github.com/FasterXML/jackson-databind">Jackson Databind</a> {@link ObjectMapper}.
 * <p>
 * This converter can be used to bind to typed beans, or untyped {@link java.util.HashMap HashMap} instances.
 * <b>Note:</b>Null objects are serialized as empty arrays and vice versa.
 * 
 * @author Thomas Darimont
 * @since 1.2
 */
public class Jackson2JsonRedisSerializer<T> implements RedisSerializer<T> {

	public static final Charset DEFAULT_CHARSET = Charset.forName("UTF-8");

	private final JavaType javaType;

	private ObjectMapper objectMapper = new ObjectMapper();
	...
}
```
推荐阅读：
https://blog.csdn.net/jinzhencs/article/details/75123631
https://stackoverflow.com/questions/46233296/can-not-store-non-string-object-when-using-stringredisserializer-for-hashvaluese
