---
layout:     post
title:      使用Gradle开发IDEA插件
subtitle:   IDEA插件
date:       2019-03-17
author:     axiomaster
header-img: img/post-bg-desk.jpg
catalog: true
tags:
    - IDEA插件
    - Gradle
---

最近想试着开发一个IDEA的插件，**[官方指导](http://www.jetbrains.org/intellij/sdk/docs/basics/getting_started.html)** 中有两种方式: 

- 使用Gradle
- 使用DevKit

而且官方推荐Gradle的方式，称DevKit的工作流已经过时了。但百度出来的全都是DevKit的方式。而且Android Studio也是gradle，那就试试gradle的方式开发一个插件。

# Gradle配置

使用gradle创建工程之后，需要下载一些文件，因为GWF的原因这一步会特别慢，不使用VPN基本都失败了。网上找了使用aliyun源的方式，搞定了这一步。

## 配置 ```build.gradle```

将工程下build.gradle中的repositories替换为如下内容

```bash
repositories {
    maven{ url 'http://maven.aliyun.com/nexus/content/groups/public/'}
}
```

参考链接如下，原文里提到```buildscript```和```allprojects```配置，但我没找到，就按上面修改了。

引用：[处理Gradle中的这个文件下载慢](https://www.zhihu.com/question/37810416/answer/153168766)

## 配置```init.gradle```

在 ```C:\Users\用户名\.gradle```目录下创建```init.gradle```文件，内容如下：

```
gradle.projectsLoaded {
    rootProject.allprojects {
        buildscript {
            repositories {
                def REPOSITORY_URL = 'http://maven.aliyun.com/nexus/content/repositories/jcenter'
                all { ArtifactRepository repo ->
                    if (repo instanceof MavenArtifactRepository) {
                        def url = repo.url.toString()
                        if (url.startsWith('https://repo1.maven.org/maven2')
                            || url.startsWith('https://jcenter.bintray.com/')) {
                            project.logger.lifecycle "Repository ${repo.url} replaced by $REPOSITORY_URL."
                            println("buildscript ${repo.url} replaced by $REPOSITORY_URL.")
                            remove repo
                        }
                    }
                }
                jcenter {
                    url REPOSITORY_URL
                }
            }
        }
        repositories {
            def REPOSITORY_URL = 'http://maven.aliyun.com/nexus/content/repositories/jcenter'
            all { ArtifactRepository repo ->
                if (repo instanceof MavenArtifactRepository) {
                    def url = repo.url.toString()
                    if (url.startsWith('https://repo1.maven.org/maven2')
                        || url.startsWith('https://jcenter.bintray.com/')) {
                        project.logger.lifecycle "Repository ${repo.url} replaced by $REPOSITORY_URL."
                        println("allprojects ${repo.url} replaced by $REPOSITORY_URL.")
                        remove repo
                    }
                }
            }
            jcenter {
                url REPOSITORY_URL
            }
        }
    }
}
```

## 下载IDEA-sources.jar

以上两个配置之后，会很快的下载好gradle相关的依赖，但开发IDEA插件，会从jetbrains官网下载```ideaIC-2018.3.5-sources.jar```文件，估计是和插件调试有关。这个文件aliyun肯定不会有，下载巨慢。

解决方法就是手动下载好，放在如下临时目录下
```bash
C:\Users\用户名\.gradle\caches\modules-2\files-2.1\com.jetbrains.intellij.idea\ideaIC\2018.3.5\临时目录\
```
具体文件和路径与IDEA版本相关。

放置之后，重启IDEA就好了。

# 使用gradle开发插件

配置好gradle之后，按照官网的指导开发一个Hello World。点击运行之后，会打开一个新的IDEA窗口，随便选择一个项目进入之后，就可以看到效果了。如下：

![idea](../img/IDEA/h1.png)

![idea](../img/IDEA/h2.png)
