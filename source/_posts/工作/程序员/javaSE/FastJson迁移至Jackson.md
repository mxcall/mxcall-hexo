---
title: FastJson迁移至Jackson
date: 2020-05-29 09:00:00
toc: true
categories: [工作, 程序员, javaSE]
tags: [java, json]
index_img: https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200529122725.png
---

# FastJson迁移至Jackson

## 背景

Fastjson在短期内连续爆出高危漏洞, 导致每次都得改代码上线, 已经粉转黑, 决定迁移到SpringBoot自带的Jackson;

相关粉转黑文章参考:

https://zhuanlan.zhihu.com/p/138215906



## 替换方法

### 添加依赖

Jackson版本等根据springboot版本来选, 因为我用的springboot版本较低, 所以用的 jackson版本是2.8

```xml
<properties>
  <jackson.version>2.8.11</jackson.version>
</properties>
...
<dependencies>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-core</artifactId>
        <version>${jackson-version}</version>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>${jackson-version}</version>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-annotations</artifactId>
        <version>${jackson-version}</version>
    </dependency>
    <dependency>
         <groupId>com.fasterxml.jackson.datatype</groupId>
         <artifactId>jackson-datatype-json-org</artifactId>
         <version>2.9.0</version>
    </dependency>
    <dependency>
        <groupId>com.jayway.jsonpath</groupId>
        <artifactId>json-path</artifactId>
        <version>2.4.0</version>
    </dependency>
</dependencies>
```



### 工具类

因为已经习惯Fastjson的语法, 所以极其不习惯在序列号之前还要new Mapper, 所以写了此工具类;

此工具类能基本保证Fastjson的默认行为与Jackson默认行为(序列号和反序列化)保持一致 (如忽略属性);

此工具类相关方法名参考 Fastjson方法名, 避免多记一份方法名;

```java
package xxxx;

import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.MapperFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.datatype.jsonorg.JsonOrgModule;
import com.jayway.jsonpath.Configuration;
import com.jayway.jsonpath.DocumentContext;
import com.jayway.jsonpath.JsonPath;
import com.jayway.jsonpath.Option;
import com.jayway.jsonpath.spi.json.JacksonJsonProvider;
import com.jayway.jsonpath.spi.json.JsonProvider;
import com.jayway.jsonpath.spi.mapper.JacksonMappingProvider;
import com.jayway.jsonpath.spi.mapper.MappingProvider;
import org.json.JSONArray;
import org.json.JSONObject;

import java.io.IOException;
import java.util.EnumSet;
import java.util.List;
import java.util.Set;

/**
 * Jackson 高仿 fastjson 工具类
 *
 * @description: 参考网站
 * 1. 替换方法细节: https://blog.csdn.net/hujkay/article/details/97040048
 * 2. JSONObject替换方案1(低于2.11.0): https://github.com/FasterXML/jackson-datatype-json-org/tree/2.9
 * 3. JSONObject替换方案2(高于2.11.0): https://github.com/FasterXML/jackson-datatypes-misc
 * 4. JSONPath 工具类: https://github.com/json-path/JsonPath
 *
 * @author: weiqi
 * @create: 2020/05/28
 **/
public class JacksonUtil {



    /**
     * 单例内部类
     */
    private static class SingletonHolder {
        private static final ObjectMapper mapper = new ObjectMapper();
        private static Configuration jsonPathConf = null;
        static {
            //Jackson 支持org.json.JSONObject
            mapper.registerModule(new JsonOrgModule());

            //Jackson 反序列化配置
            // 属性在json有, entity有, 但标记为ignore注解, 不抛出异常
            mapper.configure(DeserializationFeature.FAIL_ON_IGNORED_PROPERTIES, false);
            // 属性在json有, entity没有,  不抛出异常
            mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
            // 支持json中的key无双引号
            mapper.configure(JsonParser.Feature.ALLOW_UNQUOTED_FIELD_NAMES, true);
            // 支持带单引号的key
            mapper.configure(JsonParser.Feature.ALLOW_SINGLE_QUOTES, true);
            // 支持0开头的整数, 如001
            mapper.configure(JsonParser.Feature.ALLOW_NUMERIC_LEADING_ZEROS, true);
            // 支持回车符号
            mapper.configure(JsonParser.Feature.ALLOW_UNQUOTED_CONTROL_CHARS, true);
            // int类型为null, 则抛出异常
            mapper.configure(DeserializationFeature.FAIL_ON_NULL_FOR_PRIMITIVES, true);
            // 枚举找不到值, 不抛出异常
            mapper.configure(DeserializationFeature.FAIL_ON_NUMBERS_FOR_ENUMS, false);


            //Jackson 序列化配置
            // 空值输出 字段名: null
            mapper.configure(SerializationFeature.WRITE_NULL_MAP_VALUES, true);
            // transient注解不输出字段
            mapper.configure(MapperFeature.PROPAGATE_TRANSIENT_MARKER, true);

            //JsonPath
            Configuration.setDefaults(new Configuration.Defaults() {
                private final JsonProvider jsonProvider = new JacksonJsonProvider(mapper);
                private final MappingProvider mappingProvider = new JacksonMappingProvider(mapper);

                @Override
                public JsonProvider jsonProvider() {
                    return jsonProvider;
                }
                @Override
                public MappingProvider mappingProvider() {
                    return mappingProvider;
                }
                @Override
                public Set<Option> options() {
                    return EnumSet.noneOf(Option.class);
                }
            });

            jsonPathConf = Configuration.defaultConfiguration()
            .addOptions(Option.DEFAULT_PATH_LEAF_TO_NULL)
            .addOptions(Option.SUPPRESS_EXCEPTIONS)
            ;

            //others
        }
    }

    /**
     * 单例get
     * @return
     */
    public static ObjectMapper getInstance(){
        return SingletonHolder.mapper;
    }

    /**
     * clone 一个可以改配置
     * @return
     */
    public static ObjectMapper copyInstance(){
        ObjectMapper copy = SingletonHolder.mapper.copy();
        return copy;
    }

    /**
     * 指定Obj 对象-> 字符串
     * @param obj
     * @param <T>
     * @return
     * @throws JsonProcessingException
     */
    public static <T> String toJSONString(T obj) throws JsonProcessingException {
        return getInstance().writeValueAsString(obj);
    }


    /**
     * 字符串 -> 指定Obj 对象
     * @param json
     * @param valueType
     * @param <T>
     * @return
     * @throws IOException
     */
    public static <T> T parseObject(String json, Class<T> valueType) throws IOException {
        return getInstance().readValue(json, valueType);
    }

    /**
     * 字符串 -> 通用类JSONObject
     * @param json
     * @return
     * @throws IOException
     */
    public static JSONObject parseObject(String json) throws IOException {
        return getInstance().readValue(json, JSONObject.class);

    }


    /**
     * 字符串 -> 指定数组类
     * @param json
     * @param valueType
     * @param <T>
     * @return
     * @throws IOException
     */
    public static <T> List<T> parseArray(String json, Class<T> valueType) throws IOException {
        return getInstance().readValue(json, new TypeReference<List<T>>(){});
    }


    /**
     * 字符串 -> 通用数组类 JSONArray
     * @param json
     * @param <T>
     * @return
     * @throws IOException
     */
    public static <T> JSONArray parseArray(String json) throws IOException {
        return getInstance().readValue(json, JSONArray.class);
    }


    /**
     * 高级: 获取JSONPath 帮助类
     * @param jsonStr
     * @return
     * @throws IOException
     */
    public static DocumentContext getDocumentContext(String jsonStr) throws IOException {
        DocumentContext dcHelp = JsonPath.using(SingletonHolder.jsonPathConf).parse(jsonStr);
        return dcHelp;
    }


    /**
     * 高级: 解析 JSONPath
     * @param dcHelp
     * @param path
     * @return
     * @throws IOException
     */
    public static Object evalPath(DocumentContext dcHelp, String path) throws IOException {
        // 参考链接 https://github.com/json-path/JsonPath
        Object result = dcHelp.read(path);
        return result;
    }


    /**
     * 高级: 默认配置解析 JSONPath
     * @param jsonStr
     * @param path
     * @return
     * @throws IOException
     */
    public static Object evalPath(String jsonStr, String path) throws IOException {
        DocumentContext dcHelp = getDocumentContext(jsonStr);
        Object o = evalPath(dcHelp, path);
        return  o;
    }

}

```







## 参考链接

1. 替换方法细节: https://blog.csdn.net/hujkay/article/details/97040048
2. JSONObject替换方案1(低于2.11.0): https://github.com/FasterXML/jackson-datatype-json-org/tree/2.9
3. JSONObject替换方案2(高于2.11.0): https://github.com/FasterXML/jackson-datatypes-misc
4. JSONPath 工具类: https://github.com/json-path/JsonPath