# MybatisPlusGenerator

## 一、前言

最近在公司写项目的时候需要根据新的需求设计数据库表以及通过mybatisplus生成对应的controller等类，刚开始还以为需要写一下xml来配置generator，结果发现已经公司已经有大神写好了插件，只需要在pom里添加需要生成的表名即可，实在是太方便了……

后来还被组长叫去写了该插件的升级版以支持mybatisplus3.x，无论是maven插件和mybatis plus generator的实现原理，都挺有趣，因此开一篇文章来仔细讲讲mybatis plus generator



## 二、AutoGenerator

mybatis plus是国人编写的，因此阅读源码还是比较方便的，AutoGenerator作为整个mybatisPlusGenerator的核心类，非常的关键

```java
/*
 * Copyright (c) 2011-2020, baomidou (jobob@qq.com).
 * <p>
 * Licensed under the Apache License, Version 2.0 (the "License"); you may not
 * use this file except in compliance with the License. You may obtain a copy of
 * the License at
 * <p>
 * https://www.apache.org/licenses/LICENSE-2.0
 * <p>
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 * WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
 * License for the specific language governing permissions and limitations under
 * the License.
 */
package com.baomidou.mybatisplus.generator;

import com.baomidou.mybatisplus.annotation.TableLogic;
import com.baomidou.mybatisplus.annotation.TableName;
import com.baomidou.mybatisplus.annotation.Version;
import com.baomidou.mybatisplus.core.toolkit.StringUtils;
import com.baomidou.mybatisplus.extension.activerecord.Model;
import com.baomidou.mybatisplus.generator.config.*;
import com.baomidou.mybatisplus.generator.config.builder.ConfigBuilder;
import com.baomidou.mybatisplus.generator.config.po.TableInfo;
import com.baomidou.mybatisplus.generator.engine.AbstractTemplateEngine;
import com.baomidou.mybatisplus.generator.engine.VelocityTemplateEngine;
import lombok.AccessLevel;
import lombok.Data;
import lombok.Getter;
import lombok.Setter;
import lombok.experimental.Accessors;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.Serializable;
import java.util.List;

/**
 * 生成文件
 *
 * @author YangHu, tangguo, hubin
 * @since 2016-08-30
 */
@Data
@Accessors(chain = true)
public class AutoGenerator {
    private static final Logger logger = LoggerFactory.getLogger(AutoGenerator.class);

    /**
     * 配置信息
     */
    protected ConfigBuilder config;
    /**
     * 注入配置
     */
    @Getter(AccessLevel.NONE)
    @Setter(AccessLevel.NONE)
    protected InjectionConfig injectionConfig;
    /**
     * 数据源配置
     */
    private DataSourceConfig dataSource;
    /**
     * 数据库表配置
     */
    private StrategyConfig strategy;
    /**
     * 包 相关配置
     */
    private PackageConfig packageInfo;
    /**
     * 模板 相关配置
     */
    private TemplateConfig template;
    /**
     * 全局 相关配置
     */
    private GlobalConfig globalConfig;
    /**
     * 模板引擎
     */
    private AbstractTemplateEngine templateEngine;

    /**
     * 生成代码
     */
    public void execute() {
        logger.debug("==========================准备生成文件...==========================");
        // 初始化配置
        if (null == config) {
            config = new ConfigBuilder(packageInfo, dataSource, strategy, template, globalConfig);
            if (null != injectionConfig) {
                injectionConfig.setConfig(config);
            }
        }
        if (null == templateEngine) {
            // 为了兼容之前逻辑，采用 Velocity 引擎 【 默认 】
            templateEngine = new VelocityTemplateEngine();
        }
        // 模板引擎初始化执行文件输出
        templateEngine.init(this.pretreatmentConfigBuilder(config)).mkdirs().batchOutput().open();
        logger.debug("==========================文件生成完成！！！==========================");
    }

    /**
     * 开放表信息、预留子类重写
     *
     * @param config 配置信息
     * @return ignore
     */
    protected List<TableInfo> getAllTableInfoList(ConfigBuilder config) {
        return config.getTableInfoList();
    }

    /**
     * 预处理配置
     *
     * @param config 总配置信息
     * @return 解析数据结果集
     */
    protected ConfigBuilder pretreatmentConfigBuilder(ConfigBuilder config) {
        /*
         * 注入自定义配置
         */
        if (null != injectionConfig) {
            injectionConfig.initMap();
            config.setInjectionConfig(injectionConfig);
        }
        /*
         * 表信息列表
         */
        List<TableInfo> tableList = this.getAllTableInfoList(config);
        for (TableInfo tableInfo : tableList) {
            /* ---------- 添加导入包 ---------- */
            if (config.getGlobalConfig().isActiveRecord()) {
                // 开启 ActiveRecord 模式
                tableInfo.setImportPackages(Model.class.getCanonicalName());
            }
            if (tableInfo.isConvert()) {
                // 表注解
                tableInfo.setImportPackages(TableName.class.getCanonicalName());
            }
            if (config.getStrategyConfig().getLogicDeleteFieldName() != null && tableInfo.isLogicDelete(config.getStrategyConfig().getLogicDeleteFieldName())) {
                // 逻辑删除注解
                tableInfo.setImportPackages(TableLogic.class.getCanonicalName());
            }
            if (StringUtils.isNotEmpty(config.getStrategyConfig().getVersionFieldName())) {
                // 乐观锁注解
                tableInfo.setImportPackages(Version.class.getCanonicalName());
            }
            boolean importSerializable = true;
            if (StringUtils.isNotEmpty(config.getSuperEntityClass())) {
                // 父实体
                tableInfo.setImportPackages(config.getSuperEntityClass());
                importSerializable = false;
            }
            if (config.getGlobalConfig().isActiveRecord()) {
                importSerializable = true;
            }
            if (importSerializable) {
                tableInfo.setImportPackages(Serializable.class.getCanonicalName());
            }
            // Boolean类型is前缀处理
            if (config.getStrategyConfig().isEntityBooleanColumnRemoveIsPrefix()) {
                tableInfo.getFields().stream().filter(field -> "boolean".equalsIgnoreCase(field.getPropertyType()))
                    .filter(field -> field.getPropertyName().startsWith("is"))
                    .forEach(field -> {
                        field.setConvert(true);
                        field.setPropertyName(StringUtils.removePrefixAfterPrefixToLower(field.getPropertyName(), 2));
                    });
            }
        }
        return config.setTableInfoList(tableList);
    }

    public InjectionConfig getCfg() {
        return injectionConfig;
    }

    public AutoGenerator setCfg(InjectionConfig injectionConfig) {
        this.injectionConfig = injectionConfig;
        return this;
    }
}

```

将AutoGenerator类逐步分解，先来看看他的属性

### 1.属性

```java
/**
     * 配置信息
     */
    protected ConfigBuilder config;
    /**
     * 注入配置
     */
    protected InjectionConfig injectionConfig;
    /**
     * 数据源配置
     */
    private DataSourceConfig dataSource;
    /**
     * 数据库表配置
     */
    private StrategyConfig strategy;
    /**
     * 包 相关配置
     */
    private PackageConfig packageInfo;
    /**
     * 模板 相关配置
     */
    private TemplateConfig template;
    /**
     * 全局 相关配置
     */
    private GlobalConfig globalConfig;
    /**
     * 模板引擎
     */
    private AbstractTemplateEngine templateEngine;

```

#### ConfigBuilder

顾名思义，就是一个buidler，用来构建AutoGenerator中的配置

#### InjectionConfig

mpg默认只支持生成entity、controller、service、serviceimpl、mapper、mapper.xml，如果需要生成其他文件，则需要通过InjectionConfig来注入。

#### DataSourceConfig

配置数据库连接的想关信息

#### StrategyConfig

配置数据库表，如驼峰名称的转换、是否大写命名等

#### PackageConfig

配置包相关的信息，即配置需要生成文件所在的包名、子包和模块名等

#### TemplateConfig

配置模板的相关信息，一般在/template路径下，默认为上面说过的几种文件类型

#### GlobalConfig

配置一些全局信息，如文件的输出路径，是否开启swagger等

#### AbstractTemplateEngine

设置模板，如velocity、freemarker等



### 2.方法

#### execute

AutoGenerator的方法不多，最终的在于exectute，即执行方法。

```java
public void execute() {
        logger.debug("==========================准备生成文件...==========================");
        // 初始化配置
        if (null == config) {
            config = new ConfigBuilder(packageInfo, dataSource, strategy, template, globalConfig);
            if (null != injectionConfig) {
                injectionConfig.setConfig(config);
            }
        }
        if (null == templateEngine) {
            // 为了兼容之前逻辑，采用 Velocity 引擎 【 默认 】
            templateEngine = new VelocityTemplateEngine();
        }
        // 模板引擎初始化执行文件输出
        templateEngine.init(this.pretreatmentConfigBuilder(config)).mkdirs().batchOutput().open();
        logger.debug("==========================文件生成完成！！！==========================");
    }
```

很好理解，作者做了很详细的中文注释



## 三、举个栗子

看了上面的内容想必就不难发现了，要实现一个mybatis plus generator，只需要一个AutoGenerator->配置属性->编写模板即可，随后这个插件就能运用到各种项目中了,

```java
AutoGenerator generator = new AutoGenerator();
injectionConfig(generator);
dataSourceConfig(generator);
strategyConfig(generator);
packageConfig(generator);
templateConfig(generator);
globalConfig(generator);
templateEngineConfig(generator);
generator.execute();
```

上面是一个模板，每一个方法都对应一个AutoGenerator的属性，将autoGenerator作为参数传入并对其进行配置，最后执行execute方法。下面来详细介绍一下每一个配置具体的编写方法



## 四、InjectionConfig

简单看一眼InjectionConfig类里有什么

#### InjectionConfig

```java

public abstract class InjectionConfig {
    //全局配置
    private ConfigBuilder config;
    //自定义返回配置 Map 对象
    private Map<String, Object> map;
    //自定义输出文件
    private List<FileOutConfig> fileOutConfigList;
	//自定义判断是否创建文件
    private IFileCreate fileCreate;
    //注入自定义 Map 对象
    public abstract void initMap();
    //模板待渲染 Object Map 预处理
    //getObjectMap 结果处理
    public Map<String, Object> prepareObjectMap(Map<String, Object> objectMap) {
        return objectMap;
    }
}
```

关注fileOutConfigList属性，这个属性用于指定需要额外生成那些文件，这些文件将会用一个FileOutConfig类对象来存储，因此需要看看FileOutConfig类长什么样：

#### FileOutConfig

```java
public abstract class FileOutConfig {

    ///模板
    private String templatePath;

    public FileOutConfig() {
        // to do nothing
    }
	
	//构造器，传入模板路径
    public FileOutConfig(String templatePath) {
        this.templatePath = templatePath;
    }

    //输出文件
    public abstract String outputFile(TableInfo tableInfo);
}
```

嗯，大概懂了，要让一个文件能顺利注入到AutoGenerator中，需要创建一个FileConfig对象，并重写outputfile方法，随后将其绑定到InjectionConfig的list中就可以了，因此就可以写injectionConfig方法了

#### injectionConfig(AutoGenerator autoGenerator)

```java
private void injectionConfig(AutoGenerator autoGenerator) {
		//创建injectionConfig对象
        InjectionConfig cfg = new InjectionConfig() {
            @Override
            public void initMap() {
            }
        };
        List<FileOutConfig> focList = new ArrayList<>();
        // 添加一个FileOutConfig，传入模板路径
        focList.add(new FileOutConfig(BASE_TEMPLATE_PATH + "mapper.xml.vm") {			
            //重写outputfile方法，返回文件输出路径
            @Override
            public String outputFile(TableInfo tableInfo) {
                tableInfo.getFields().forEach(tableField -> tableField.setConvert(true));
                return System.getProperty("user.dir") + "/src/main/resources/mapper/" + tableInfo.getEntityName() + "Mapper.xml";
            }
        });
    	//配置到InjectionConfig和AutoGenerator
    	cfg.setFileOutConfigList(focList);
        autoGenerator.setCfg(cfg);
}
```



## 五、DataSourceConfig

```java
private void dataSourceConfig(AutoGenerator autoGenerator) {
        DataSourceConfig dsc = new DataSourceConfig();
        dsc.setDbType(DbType.MYSQL);
        // 默认driver
        String driver = "com.mysql.jdbc.Driver";
        dsc.setDriverName(driver);
        dsc.setUsername("username");
        dsc.setPassword("password");
        dsc.setUrl("url");
        autoGenerator.setDataSource(dsc);
    }
```

datasourceConfig的配置也是极为简单，唯一的缺点在于这些配置都是硬编码的，无法维护。但是解决方法也多种多样，可以通过传参的方法将参数带入，也可以通过读取application.yml文件获取，这里就不详细讲了。



## 六、StrategyConfig

```java
private void strategyConfig(generator){
	StrategyConfig strategy = new StrategyConfig();
   		//开启lombok
        strategy.setEntityLombokModel(true);
        strategy.setNaming(NamingStrategy.underline_to_camel);
        strategy.setInclude(tables);
        if (useCommonsWeb) {
		//配置可能会用到的superclass           
            strategy.setSuperServiceClass("com.kyle.example.service.Service");
            strategy.setSuperServiceImplClass("com.kyle.example..service.impl.ServiceImpl");
            strategy.setSuperControllerClass("com.kyle.example.controller.CommonController");
        }
        autoGenerator.setStrategy(strategy);   
}
```



## 七、PackageConfig

```java
private void packageConfig(generator){
        PackageConfig pc = new PackageConfig();
    	//设定父包名
        pc.setParent("com.kyle.example");
    	//设定子包名
        pc.setEntity("config");
        pc.setMapper("dao");
        pc.setService("config");
        pc.setServiceImpl("service.impl");
        pc.setController("controller");
        autoGenerator.setPackageInfo(pc);
}
```



## 八、TemplateConfig

```java
private void templateConfig(AutoGenerator autoGenerator) {
    	String BASE_TEMPLATE_PATH = "com/kyle/example/templates/";
    	//指定模板文件的路径
        TemplateConfig tc = new TemplateConfig();
        tc.setEntity(BASE_TEMPLATE_PATH + "entity.java.vm");
        tc.setService(BASE_TEMPLATE_PATH + "service.java.vm");
        tc.setServiceImpl(BASE_TEMPLATE_PATH + "serviceImpl.java.vm");
        tc.setMapper(BASE_TEMPLATE_PATH + "mapper.java.vm");
        tc.setController(BASE_TEMPLATE_PATH + "controller.java.vm");
        tc.setXml(null);
        autoGenerator.setTemplate(tc);
}
```



## 九、GlobalConfig

```java
private void globalConfig(AutoGenerator autoGenerator) {
    GlobalConfig gc = new GlobalConfig();
    //设定文件输出路径
    gc.setOutputDir(System.getProperty("user.dir") + "/src/main/java/");
    //设定文件重写
    gc.setFileOverride(true);
    gc.setSwagger2(true);
    gc.setActiveRecord(true);
    gc.setEnableCache(false);
    gc.setOpen(false);
    gc.setBaseResultMap(true);
    gc.setBaseColumnList(false);
    gc.setAuthor(this.author);
    gc.setServiceName("%sService");
    autoGenerator.setGlobalConfig(gc);
}
```



## 十、TemplateEngineConfig

```
private void templateEngineConfig(AutoGenerator autoGenerator) {
        VelocityTemplateEngine velocityEngine = new VelocityTemplateEngine();
        autoGenerator.setTemplateEngine(velocityEngine);
    }
```



## 十一、总结

AutoGenerator的配置大概先写到这里，后面是编写模板和相关配置的升级。