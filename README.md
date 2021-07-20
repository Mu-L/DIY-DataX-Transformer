# DIY-DataX-Transformer
早该知道的一种更高级的 transformer 自定义实现方法。

让 DataX 运行时加载自定义 transformer 插件。

详情链接：[我的 CSDN 博客](https://blog.csdn.net/landstream/article/details/88878172)

## 前言

之前的文章有介绍过通过自定义 transformer 在 DataX 上实现 ETL(Extract Transform Load) 过程中定制化的数据处理，当时的实现方法是自定义插件并手写代码注册到`com/alibaba/datax/core/transport/transformer/TransformerRegistry.java`文件中。

如下所示：

```java
 registTransformer(new SubstrTransformer());
 registTransformer(new PadTransformer());
 registTransformer(new ReplaceTransformer());
 registTransformer(new FilterTransformer());
 registTransformer(new GroovyTransformer());
```

这种方法有明显的缺点：当我们对 transformer 进行修改后，需要重新编译 dataX 的核心模块。显然这种做法的系统耦合性过大，完全不符合 DataX 系统重视插件机制的设计理念。

那么，有没有更好的方法呢？

[官方文档](https://github.com/alibaba/DataX/blob/master/dataxPluginDev.md)提到了 Reader 和 Writer 的加载原理，依据相关做法，开发者可以仅编写编译自己要使用的插件，然后按要求包装后放到指定的位置，DataX 就会在启动后加载相关插件。

原代码告诉我，Transformer 也支持类似的加载方法。

核心的功能实现是由下面的 *TransformerRegistry* 类来完成的：

```java
/* 
	com/alibaba/datax/core/transport/transformer/TransformerRegistry.java
*/
public static void loadTransformer(String each) {
    String transformerPath = CoreConstant.DATAX_STORAGE_TRANSFORMER_HOME + File.separator + each;
    Configuration transformerConfiguration;
    try {
        transformerConfiguration = loadTransFormerConfig(transformerPath);
    } catch (Exception e) {
        LOG.error(String.format("skip transformer(%s),load transformer.json error, path = %s, ", each, transformerPath), e);
        return;
    }

    String className = transformerConfiguration.getString("class");
    if (StringUtils.isEmpty(className)) {
        LOG.error(String.format("skip transformer(%s),class not config, path = %s, config = %s", each, transformerPath, transformerConfiguration.beautify()));
        return;
    }

    String funName = transformerConfiguration.getString("name");
    if (!each.equals(funName)) {
        LOG.warn(String.format("transformer(%s) name not match transformer.json config name[%s], will ignore json's name, path = %s, config = %s", each, funName, transformerPath, transformerConfiguration.beautify()));
    }
    JarLoader jarLoader = new JarLoader(new String[]{transformerPath});

    try {
        Class<?> transformerClass = jarLoader.loadClass(className);
        Object transformer = transformerClass.newInstance();
        if (ComplexTransformer.class.isAssignableFrom(transformer.getClass())) {
            ((ComplexTransformer) transformer).setTransformerName(each);
            registComplexTransformer((ComplexTransformer) transformer, jarLoader, false);
        } else if (Transformer.class.isAssignableFrom(transformer.getClass())) {
            ((Transformer) transformer).setTransformerName(each);
            registTransformer((Transformer) transformer, jarLoader, false);
        } else {
            LOG.error(String.format("load Transformer class(%s) error, path = %s", className, transformerPath));
        }
    } catch (Exception e) {
        //错误funciton跳过
        LOG.error(String.format("skip transformer(%s),load Transformer class error, path = %s ", each, transformerPath), e);
    }
}
```

## 你该怎么做

### 独立创建一个 transformer 项目

* 继承并实现 `com.alibaba.datax.transformer.Transformer` 或 `com.alibaba.datax.transformer.ComplexTransformer`

  注意：需要手写两个 json 文件，文件命名和内容有讲究：

  transformer.json

  plugin_job_template.json （非必须）

* 打包

  以 hiding_transformer 为例

```xml
<assembly
        xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.0 http://maven.apache.org/xsd/assembly-1.1.0.xsd">
    <id></id>
    <formats>
        <format>dir</format>
    </formats>
    <includeBaseDirectory>false</includeBaseDirectory>
    <fileSets>
        <fileSet>
            <directory>src/main/resources</directory>
            <includes>
                <include>transformer.json</include>
                <include>plugin_job_template.json</include>
            </includes>
            <outputDirectory>plugin/transformer/hiding_transformer</outputDirectory>
        </fileSet>
        <fileSet>
            <directory>target/</directory>
            <includes>
                <include>hidingtransformer-1.0-SNAPSHOT.jar</include>
            </includes>
            <outputDirectory>plugin/transformer/hiding_transformer</outputDirectory>
        </fileSet>
    </fileSets>

    <dependencySets>
        <dependencySet>
            <useProjectArtifact>false</useProjectArtifact>
            <outputDirectory>plugin/transformer/hiding_transformer/libs</outputDirectory>
            <scope>runtime</scope>
        </dependencySet>
    </dependencySets>
</assembly>
```

> mvn clean package -DskipTests assembly:assembly

* 把打好的包放到 dataX 程序目录下一个特别的位置（自己动手mkdir）

  datax/local_storage/transformer

```dir
└─datax
    ├─bin
    ├─conf
    ├─job
    ├─lib
    ├─local_storage <-----------------这个！
    │  └─transformer
    │      └─hiding_transformer
    │          └─libs
    ├─log
    │  └─2019-03-28
    ├─log_perf
    │  └─2019-03-28
    ├─plugin
    │  ├─reader
    │  │  ├─drdsreader
    │  │  │  └─libs
    │  │  ├─hbase20xsqlreader
    │  │  │  └─libs
    │  │  ├─mysqlreader
    │  │  │  └─libs
    │  │  ├─odpsreader
    │  │  │  └─libs
    │  │  ├─oraclereader
    │  │  │  └─libs
    │  │  ├─otsreader
    │  │  │  └─libs
    │  │  ├─postgresqlreader
    │  │  │  └─libs
    │  │  ├─sqlserverreader
    │  │  │  └─libs
    │  │  └─streamreader
    │  │      └─libs
    │  └─writer
    │      ├─hbase20xsqlwriter
    │      │  └─libs
    │      └─streamwriter
    │          └─libs
    ├─script
    └─tmp
```

