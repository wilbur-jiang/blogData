---
title: 日常填坑之如何优雅的解决dependency多版本冲突 
tags: gradle,maven,多版本冲突,dependency
grammar_cjkRuby: true
---

### 问题背景
今天为大家分享一个最近遇到的问题，引入了新的excel开源库*easy-excel*后，线上已有的excel导出功能出现了bug。
最后定位到问题是由于*easy-excel*使用的poi版本为==3.17==，而已有代码使用的poi版本为==3.9==。虽然看起来只是小版本的升级，但是很多3.9中的API已经废弃了，这也直接导致了已有功能的导出异常（真的是无力吐槽这个向下兼容性）。

既然定位到了问题，我们就要快速解决问题并重新上线，由于业务都有一定的复杂度，重写已有导出功能或者新增功能都是比较耗时的。那从问题本身来看有没有什么通用的解决方案呢？
### 问题的本质
首先我们来看问题本身，为何会出现冲突？我们看下在poi中比较常用的几个类的import语句如下。
``` java
import org.apache.poi.ss.usermodel.Cell;
import org.apache.poi.ss.usermodel.Row;
import org.apache.poi.ss.usermodel.Sheet;
```
由于3.9与3.17版本中Cell，Row，Sheet的包名和类名都是相同的，编译时只能选择其中一个。这一点Apache的commons包就做的比较好，直接通过包名将高版本与低版本做了区分。

``` java
import org.apache.commons.lang.StringUtils;
import org.apache.commons.lang3.StringUtils;
```

这也为我们解决问题提供了一个思路，可以通过修改包名的方式来区分poi版本，达到引入不同版本的poi的目的。
### 解决方案
1 手动修改包名，重新打包poi和easy-excel
通过上面的分析，我们可以将3.17版本的poi和easy-excel源码下载，修改包名后打包成新的jar，引入到项目中。
2 利用打包工具
Java程序猿比较常用的打包工具有maven、gradle，都提供了非常强大的功能和丰富的插件。对于依赖冲突的问题，也都提供了对应的解决方案。gradle中可以使用shadow插件，Maven可以使用Shade插件。
由于项目中使用gradle，这里主要介绍shadow的使用方案。
首先，我们创建一个新的module命名为common-excel，具体的gradle配置如下：

``` gradle
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies{
        classpath("com.github.jengelman.gradle.plugins:shadow:4.0.4")
    }
}

apply plugin: 'java'

//使用shadow插件
apply plugin: 'com.github.johnrengelman.shadow'



repositories {
    mavenCentral()
}

dependencies {
    compile 'com.alibaba:easyexcel:1.0.1'
}

//shadow配置，relocate包名
shadowJar {
    baseName = 'demo'
    version = null
    zip64 true
    relocate('org.apache.poi', 'demo.org.apache.poi')
    mergeServiceFiles()
}
```
配置很简单，我们通过relocate配置，将org.apache.poi映射成demo.org.apache.poi。Sahdow插件通过ASM类库来实现这个功能，将字节码中的org.apache.poi做了替换，并重新打包。这样不管是3.17版本中的poi，还是easy-excel中对poi的import都是使用新的包名。
最后，我们在项目中引入该模块，就实现了3.9与3.17版本poi的共存。

``` gradle
    compile project(path: ':common-excel',configuration:'shadow')
```

增加module的方式还是对源码有一定的侵入，我们也可以直接将打好的包放到内部的maven库中。

### 总结
其实上面提到是一种救火方案，我们应该在平常就确定一套通用的工具类库，如果觉得工具类库太low或者不能满足需求，应该提前评估工具替换可能带来的影响，并通知到组内同事。