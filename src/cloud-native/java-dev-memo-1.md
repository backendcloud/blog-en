release time :2021-11-23 15:04

Record some problems encountered in developing projects in Java:

# @Pattern annotation
For the fields in the http request body, regular checks need to be done. For general regular checks that do not require conditional judgments, you can use @Pattern instead of writing regular checks in the method, which simplifies development.

In order to verify, it must start with "sftp://", written in the following form
@Pattern(regexp = "^sftp://")

But the content of the field is an sftp://10.10.10.10:33022/ftp_testerror
, and the @Pattern annotation will automatically add ^ and &. The correct spelling is
@Pattern(regexp = "^sftp://.*$")or@Pattern(regexp = "sftp://.*")

So the regular expression is improved and written in the following form:

```java
/**
 * ftpPath
 */
@JsonProperty("ftpServer")
@Pattern(regexp = "sftp://.+[:\\d+]?/.*", message = "ftpPath")
private String ftpPath;
/**
 * ftpUser
 */
@JsonProperty("ftpUser")
private String ftpUser;
/**
 * ftpPasswd
 */
@JsonProperty("ftpPasswd")
private String ftpPasswd;
```

# To enable https and https services at the same time
Open the http service separately, and write the configuration file of springboot

    server.port: 33021



Open the https service separately, use keytool to make a certificate, and the configuration file of springboot is written as follows:

    #https port
    server.port: 33021

    #https config
    server.ssl.key-store: classpath:keystore.p12
    #server.ssl.key-store: file:/Users/hanwei/xxx/yyy/api/src/main/resources/keystore.p12
    server.ssl.key-store-password: 123456
    server.ssl.keyStoreType: PKCS12
    server.ssl.keyAlias: tomcat

To enable http service and https service at the same time, the configuration file of springboot is written as follows:

    #https port
    server.port: 33021

    # http port
    http.port: 33011

    #https config
    server.ssl.key-store: classpath:keystore.p12
    #server.ssl.key-store: file:/Users/hanwei/xxx/yyy/api/src/main/resources/keystore.p12
    server.ssl.key-store-password: 123456
    server.ssl.keyStoreType: PKCS12

and create the following java file:

```java
import org.apache.catalina.connector.Connector;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.boot.web.servlet.server.ServletWebServerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;


@Configuration
public class SSLConfig {
    @Value("${http.port}")
    private Integer port;

    @Bean
    public ServletWebServerFactory servletContainer() {

        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory();
        Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
        connector.setPort(port);
        tomcat.addAdditionalTomcatConnectors(connector);
        return tomcat;
    }

}
```



# I want the null field of the http response body json to be displayed as an empty string""

```java
//demo
    @RequestMapping("/map")
    public Map<String, Object> getMap() {
        Map<String, Object> map = new HashMap<>(3);
        User user = new User(1, "xxx", null);
        map.put("auth", user);
        map.put("blog url", "http://www.yyy.com");
        map.put("phone number", null);
        return map;
    }
```

Call interface display: {"author information":{"id":1,"username":"xxx","password":null},"CSDN address":null,"blog address":" http://www .yyy.com"}

Create the following java file:

```java
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.databind.Module;
import com.fasterxml.jackson.databind.module.SimpleModule;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.http.converter.json.Jackson2ObjectMapperBuilder;

import java.io.IOException;

@Configuration
public class JacksonConfig {
    @Bean
    @Primary
    @ConditionalOnMissingBean(ObjectMapper.class)
    public ObjectMapper jacksonObjectMapper(Jackson2ObjectMapperBuilder builder) {
        ObjectMapper objectMapper = builder.createXmlMapper(false).build();
        objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, true);
        objectMapper.configure(MapperFeature.ALLOW_COERCION_OF_SCALARS, false);
        objectMapper.getSerializerProvider().setNullValueSerializer(new JsonSerializer<Object>() {
            @Override
            public void serialize(Object o, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException {
                jsonGenerator.writeString("");
            }
        });
        return objectMapper;
    }

    @Bean
    public Module customStringDeserializer() {
        SimpleModule module = new SimpleModule();
        module.addDeserializer(String.class, CustomStringDeserializer.instance);
        return module;
    }
}
```

Call interface display: {"author information":{"id":1,"username":"xxx","password":""},"CSDN address":"","blog address":" http:/ /www.yyy.com"}

# I want the null field of http response body json not to be displayed

@JsonInclude(JsonInclude.Include.NON_NULL)Annotations can be used , for example:

```java
import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.List;

@NoArgsConstructor
@Data
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ListPoolTenantsResponse {

    /**
     * tenantId
     */
    @JsonProperty("tenantId")
    private String tenantId;
    /**
     * poolTenants
     */
    @JsonProperty("poolTenants")
    private List<PoolTenantsDTO> poolTenants;

    /**
     * PoolTenantsDTO
     */
    @NoArgsConstructor
    @Data
    @JsonInclude(JsonInclude.Include.NON_NULL)
    public static class PoolTenantsDTO {
        /**
         * id
         */
        @JsonProperty("id")
        private String id;
        /**
         * name
         */
        @JsonProperty("name")
        private String name;
        /**
         * poolId
         */
        @JsonProperty("poolId")
        private String poolId;
        /**
         * poolName
         */
        @JsonProperty("poolName")
        private String poolName;
        /**
         * poolType
         */
        @JsonProperty("poolType")
        private Integer poolType;
        /**
         * description
         */
        @JsonProperty("description")
        private String description;
        /**
         * status
         */
        @JsonProperty("status")
        private String status;
        /**
         * quotaSpec
         */
        @JsonProperty("quotaSpec")
        private QuotaSpecDTO quotaSpec;

        /**
         * QuotaSpecDTO
         */
        @NoArgsConstructor
        @Data
        @JsonInclude(JsonInclude.Include.NON_NULL)
        public static class QuotaSpecDTO {
            /**
             * cpu
             */
            @JsonProperty("cpu")
            private Integer cpu;
            /**
             * memory
             */
            @JsonProperty("memory")
            private Integer memory;
        }
    }
}
```



# When Mybatis-Plus is used as ORM, I want to update a field in the database to null
By default, the field cannot be updated to null. Even if it is updated to null, the query database finds that the field is still the original field and has not been updated, because mybatis-plus FieldStrategy has three strategies:

* IGNORED: 0 Ignore
* NOT_NULL: 1 is not NULL, the default policy
* NOT_EMPTY: 2 is not empty



and the default update policy is NOT_NULL: not NULL; that is, when the data is NULL when updating data through the interface, it will not be updated into the database.

The default strategy of Mybatis-Plus provides great convenience for the update operation. For example, only the fields in the json body are updated when the http request is updated, and the fields that are not in the body will not be updated, which also meets the general requirements. It is a bit strange to update the fields that are not in the body to null. The general requirement is to keep fields that are null as-is.

If a field needs to be updated to null, it can be @TableField(updateStrategy = FieldStrategy.IGNORED)realized through annotations.

For example:

```java
import com.alibaba.fastjson.JSONObject;
import com.baomidou.mybatisplus.annotation.FieldStrategy;
import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import com.baomidou.mybatisplus.extension.handlers.JacksonTypeHandler;
import com.fasterxml.jackson.databind.node.ObjectNode;
import lombok.Data;

@Data
@TableName(value = "image", autoResultMap = true)
public class Image {
//    private String id;
    @TableId("name")
    private String name;
    private String description;
    private boolean publicView; //和Harbor一致
    private String type;
    private String registry; //镜像仓库地址，支持多镜像仓库
    private String repository; //"project/repository"的形式，和Harbor一致
    @TableField(typeHandler = JacksonTypeHandler.class, updateStrategy = FieldStrategy.IGNORED)
    private JSONObject lables; //"lables": {"gpu": "yes"}
}
```

# How does the json data type map between Java entity fields and database fields
The mapping between the json data type and the Java entity class is very common, and json nested json can also write a nested inner class in the Java entity class. The internal json data type corresponds to the internal class of the entity, but the current requirement is to associate it with a certain json type field in the database. It can be done in the following ways.

For example:

```java
CREATE TABLE image
(
    #id     int AUTO_INCREMENT comment 'id',
--     id   varchar(80),
    name varchar(80),
    description varchar(500),
    public_view tinyint(1) comment '0:非公开；1:公开',
    type varchar(80),
    registry varchar(80),
    repository varchar(80),
    lables JSON,
    primary key (name)
);
```

```java
import com.alibaba.fastjson.JSONObject;
import com.baomidou.mybatisplus.annotation.FieldStrategy;
import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import com.baomidou.mybatisplus.extension.handlers.JacksonTypeHandler;
import com.fasterxml.jackson.databind.node.ObjectNode;
import lombok.Data;

@Data
@TableName(value = "image", autoResultMap = true)
public class Image {
//    private String id;
    @TableId("name")
    private String name;
    private String description;
    private boolean publicView; //和Harbor一致
    private String type;
    private String registry; //记录仓库地址，支持多镜像仓库
    private String repository; //"project/repository"的形式，和Harbor一致
    @TableField(typeHandler = JacksonTypeHandler.class, updateStrategy = FieldStrategy.IGNORED)
    private JSONObject lables; //"lables": {"gpu": "yes"}
}
```

typeHandler: Field type handler for conversion between JavaType and JdbcType.

If you do not follow the above processing, the following error will be reported:

    com.mysql.cj.jdbc.exceptions.MysqlDataTruncation: Data truncation: Cannot create a JSON value from a string with CHARACTER SET 'binary'."

# The http response body field displays the time format on demand
Add annotations before the response body field:

    @JsonFormat(pattern="yyyy-MM-dd HH:mm:ss", timezone="GMT+8") Display the time of East Eight District
    @JsonFormat(pattern="yyyy-MM-dd'T'HH:mm: ss.SSS'Z'") displays UTC time, accurate to milliseconds, for example: 2017-06-01T00:00:00.000Z

# Synchronize data between servers
You can use the put method of com.jcraft.jsch.ChannelSftp to write a scheduled update task

```java
@Async("taskScheduler")
@Scheduled(cron = "0/15 * * * * ?")
public void update() {
```

Org.apache.camel can also be used to synchronize

```java
import org.apache.camel.builder.RouteBuilder;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class FtpRouteBuilder extends RouteBuilder {

    @Override
    public void configure() throws Exception {
        from("file:/tmp/ftp_test?recursive=true&noop=true").to("sftp://10.10.10.10:33022/ftp_test?username=root&password=123456@&binary=true");
    }
}
```


# Modification log packaging cycle
## At the same time, modify the packaging by unit time and size (take the modification of SystemLog as an example)

    <appender name="FILE-SFTP-SystemLog" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <File>${FILE_PATH_SFTP_SystemLog}</File>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <FileNamePattern>${FILE_PATH_SFTP_SystemLog}_%d{"yyyy-MM-dd_HH", UTC}_0.zip</FileNamePattern>
            <MaxFileSize>100MB</MaxFileSize>
            <totalSizeCap>1GB</totalSizeCap>
            <MaxHistory>2</MaxHistory>
        </rollingPolicy>

        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>INFO</level>
        </filter>
        <encoder>
            <pattern>
                {
                "time": "%d{yyyy-MM-dd HH:mm:ss.SSS}",
                "level": "%level",
                "thread": "%thread",
                "class": "%logger{40}",
                "message": "%message"
                }%n
            </pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

```bash
    <rollingPolicy ... /rollingPolicy>Several parameters are used to configure the log file splitting method:

    <FileNamePattern>The in yyyy-MM-dd_HH-mmmeans to split the file by hour, yyyy-MM-dd_HHmeans to split the file by hour, means to split the file yyyy-MM-ddby day, means to split the file by yyyy-MMmonth, and yyyymeans to split the file by year.
    <MaxFileSize>Indicates that the size of a packaged single file does not exceed this value.
    <totalSizeCap>Limit the size of log packaging.
    <MaxHistory>Limit the number of historical pack files.
```


## Modify Logback to be packaged in minutes within hours (take modifying SystemLog as an example) (the following time intervals are supported: 1, 2, 5, 10, 15, 20, 30 minutes)



    <appender name="FILE-SFTP-SystemLog" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <File>${FILE_PATH_SFTP_SystemLog}</File>
        <!--        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">-->
        <rollingPolicy
                class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--            <FileNamePattern>${FILE_PATH_SFTP_OperationLog}_%d{yyyy-MM-dd-HH-mm}_%i.zip</FileNamePattern>-->
            <FileNamePattern>${FILE_PATH_SFTP_SystemLog}_%d{"yyyy-MM-dd_HH-mm", UTC}_0.zip</FileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy
                    class="xxx.yyy.api.config.MyTimeBasedFileNamingAndTriggeringPolicy">
                <multiple>5</multiple>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--            <MaxFileSize>100MB</MaxFileSize>-->
            <!--            <totalSizeCap>1GB</totalSizeCap>-->
            <MaxHistory>90</MaxHistory>
        </rollingPolicy>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>INFO</level>
        </filter>
        <encoder>
            <pattern>
                {
                "time": "%d{yyyy-MM-dd HH:mm:ss.SSS}",
                "level": "%level",
                "thread": "%thread",
                "class": "%logger{40}",
                "message": "%message"
                }%n
            </pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

```bash
<rollingPolicy ... /rollingPolicy>Several parameters are used to configure the log file splitting method:

<FileNamePattern>The time format part in can only be fixed to yyyy-MM-dd_HH-mm.
<multiple>Indicates the packaging interval in minutes. The following intervals are supported: 1, 2, 5, 10, 12, 15, 20, 30 minutes.
<MaxHistory>Limit the time of historical pack files. That is to say, the number of historical pack files is <MaxHistory>divided by <multiple>.
```

The MyTimeBasedFileNamingAndTriggeringPolicy.java file is as follows:

```java
package xxx.yyy.api.config;

import ch.qos.logback.core.joran.spi.NoAutoStart;
import ch.qos.logback.core.rolling.DefaultTimeBasedFileNamingAndTriggeringPolicy;

@NoAutoStart
public class MyTimeBasedFileNamingAndTriggeringPolicy<E> extends DefaultTimeBasedFileNamingAndTriggeringPolicy<E> {
    //这个用来指定时间间隔
    private Integer multiple = 1;

    @Override
    protected void computeNextCheck() {
        nextCheck = rc.getEndOfNextNthPeriod(dateInCurrentPeriod, multiple).getTime();
    }

    public Integer getMultiple() {
        return multiple;
    }

    public void setMultiple(Integer multiple) {
        if (multiple > 1) {
            this.multiple = multiple;
        }
    }
}
```


