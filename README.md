# 使用 Lint 进行代码检查

## 背景
为了进一步规范协同合作中的代码规范，避免写低级 Bug 和对代码进行规约，在调研了多种方案后，决定使用该种方案对代码进行自动化检查和规约

### 使用

Github 开源地址：https://github.com/PPTing/AwesomeLint

1. 先在配置文件(`local.properties`)中配置 `nexusUrl` 的值为本地仓库的地址
2. 复制 git hook 脚本<br>

2.1 在根目录下执行执行  
Windows:

  2.1.1 修改 `post-commit-windows` 文件中的第一行代码中的`D:/Git`为自己本机电脑的 `Git` 安装目录，即第一行代码为声明 `sh.exe` 的目录
  2.1.2 执行 `gradle installGitHooks`

Mac OS/Linux:  `./gradlew installGitHooks`<br>

或者在 `Android Studio` 中右边面板上找到 `root-Tasks-other-installGitHooks` 并双击执行

2.2 赋予执行脚本可执行权限 `chmod +x .git/hooks/post-commit`<br>

### 环境准备

使用<strong>git提交增量检查</strong>时需要配置ANDROID_HOME环境变量(需要以ANDROID_HOME命名并加入到path中，因为在Lint框架中执行Lint检查时需要获取Android环境变量)

```txt 
Windows环境：在电脑->属性->环境变量中编辑即可
```

```sh
Mac OS/Linux环境：编辑 ~/.bashrc即可
vi ~./bashrc
export ANDROID_HOME=$HOME/{Android SDK 路径}
export PATH=$PATH:$ANDROID_HOME/tools
```





### 实时提示

> 在 Android Studio 中进行实时提示

#### 进行配置

**方法一**

* 将自定义 Lint 规则的 aar 包放入项目中，例如 `${root}/libs` 目录①
* 在项目的根目录下的 `build.gradle` 文件内的 `repositories` block 中为所有的 module 添加 aar 的路径(主要是便于管理，不需要所有 module 的 build.gradle 文件都写一遍 aar 路径)

eg

```
    allprojects{
        repositories{
            flatDir{
                dirs "../${root}/libs"
            }
        }
    }
```


* 在每个 module 中的 `dependencies` 中加入 `implementation (name:'AAR的名字', ext:'aar')` 即可

**方法二**

将 `lintRules` 发布到仓库中，再通过远程依赖在每个 module 中进行依赖
即 `implementation 'me.ppting.plugin:lintPreview:1.0.0.beta'`

##### 效果
> 目前只自定义了几个规则，例如
* 类名和方法名的命名需要符合驼峰命名法

效果如下 ![Lint 检测后的展示效果](https://raw.githubusercontent.com/PPTing/AwesomeLint/master/images/Xnip2019-09-16_16-00-10.png)

在 IDE 中会根据错误级别进行相应的提示，在 `Darcula` 主题下默认会以黄色的前景色进行提示，将鼠标移至该代码块则会有浮窗提示问题

#### 去除提示(SuppressLint)

假如某个 Lint 规则提示代码有误，但实际上是因为自定义规则的检测有误导致误报，或者代码并没有错误，可以使用 `@SuppressLint` 对该代码段进行注解，注解的参数填写 Lint 规则的 id，一般会在浮窗中展示该规则 id。

如果实在不知道该规则 id ，可以使用 `@SuppressLint("All")` 忽略所有的规则，但这样就会导致该方法内新增的代码也无法进行 Lint 规则检测，请***谨慎使用***

这样该方法就会忽略该规则的检测



#### 自定义规则


#### 编写规则
在 lintRules module 中编写规则

* 创建一个`继承 Detector 的类`
* 类中创建一个 ISSUE 表示该类用来检测的问题
* 在 Register 类中对该 issue 进行注册

> 具体参考现有的代码

#### 测试

* 执行 lintAAR` 中的 `copyAAR` 的 task 即可编译出 aar 文件并复制到 app module 中，
* Sync Project With Gradle Files (点击 Gradle 的同步按钮)

即可在 app module 中测试各个规则是否生效

#### 发布

测试完毕，将 aar 文件复制到其他应用到它的地方即可，并同步即可生效

> PS. 如果不生效，尝试重启 Android Studio 


### COMMIT HOOKS

> 为了强制进行一些编码规范等的执行，会在 git hooks 在进行 commit 后做检查，如果检测不通过，则会触发 git reset 进行回滚此次提交，并将错误提示日志打印到 `lint-check-result.log` 文件中

#### 大概原理

i. 在项目中应用 gradle plugin，在每次 git commit 后，在 `.git/hooks` 中的 `post-commit` hook 会自动执行，会到项目根目录下执行 `./gradlew lintCheck -PisLintCheck`

ii. lintCheck 会通过 `git diff` 获取本次提交和上一次提交之间的代码差异，并记录其行数和文件名

iii. 将所有的改动的文件进行 lint 操作，并记录其 issue，如果某个 issue 对应的代码行正好是改动的代码行，则将记录数(记为 K)加一

iiii. 当结束 lintCheck 后，如果 K > 0 ，则将代码回滚(`git reset HEAD~1`)，即将该 commit 撤销

PS. 如果该 `Issue` 是 `Warning` 级别的，Android Studio 不会进行提示，如果该 `Issue` 是 `Error` 级别的，Android Studio 会有错误提示弹窗



#### 应用
1. 在项目根目录下的 `build.gradle` 中应用插件 

    ```
    apply plugin:"me.ppting.plugin.lint"
    ```

2. 设置 lintCheck 配置

    在 根目录下的 `build.gradle` 中添加 lintCheck 配置

    eg.

    ```
    lintConfig {
        //配置Lint检查文件的类型
        lintCheckFileType = ".java,.xml,.kt"
        //是否将检查文件的所有扫描结果都输出
        lintReportAll = true
        //是否进行 lint 检查，默认为 true
        isOpenLint = false
    }
    ```

3. 复制 `post-commit` 脚本到本地的 `.git/hooks` 目录下

    在根目录下执行执行  
    > Mac/Linux: `./gradlew installGitHooks`<br>
    > Windows: `gradle installGitHooks`

    或者在 `Android Studio` 中右边面板上找到 `root-Tasks-other-installGitHooks` 并双击执行


PS. 如果无法执行 `post-commit` 脚本，可能是权限问题，给该脚本加上可执行权限

`chmod +x .git/hooks/post-commit`

PSS. 如果要强制关闭在 commit 之后的 hook 操作，可以使用 `deleteGitHooks` task 将 `.git/hooks/post-commit` 文件删除




### 遇到的一些坑和解决方案

***坑 1：*** 由于 lint 的 api 依赖于 kotlin-compiler ，而这会造成 Android Studio 的 Kotlin-Plugin 冲突，而导致无法编译

**错误提示：**
```
FAILURE: Build failed with an exception.

* What went wrong:
org.jetbrains.kotlin.cli.common.arguments.K2JVMCompilerArguments.setAllowNoSourceFiles(Z)V

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output. Run with --scan to get full insights.

* Get more help at https://help.gradle.org

Deprecated Gradle features were used in this build, making it incompatible with Gradle 6.0.
Use '--warning-mode all' to show the individual deprecation warnings.
See https://docs.gradle.org/5.4.1/userguide/command_line_interface.html#sec:command_line_warnings

BUILD FAILED in 20s
```

**解决方法：**

在 buildSrc/build.gradle 文件中控制 lint api 的依赖

```
    configurations {
        all*.exclude group: 'com.android.tools.external.com-intellij',module:"kotlin-compiler"
    }
```


这样可以编译，但会导致 lintCheck task 失败
于是我们使用一个变量 `isLintCheck` 来控制 lint api 是否要 exclude 掉 kotlin-compiler 的依赖
    
如下

```
    /**
    * lintCheck 需要有 kotlin-compiler
    * run 不需要 kotlin-compiler
    */
    configurations {
        boolean isLintCheck = project.hasProperty("isLintCheck")
        if (!isLintCheck) {
            all*.exclude group: 'com.android.tools.external.com-intellij',module:"kotlin-compiler"
        }
    }
```

所以要求我们在执行 lintCheck 任务时加上参数 -PisLintCheck 
    
***坑 2：*** 将 plugin 发布到仓库中作为插件使用

>当尝试将 plugin 发布到仓库中，并在项目中使用 classpath 进行依赖该插件，会导致无法编译

**错误提示：**

```
org.jetbrains.kotlin.resolve.diagnostics.DiagnosticSuppressor$Companion.getEP_NAME()Lcom/intellij/openapi/extensions/ExtensionPointName;
```

**解决方案：**

只能将插件项目通过 buildSrc module 的方式引入到需要使用 lint 的项目中


***坑 3：*** walle 报错

>由于升级了 Gradle 的版本，项目中也使用了美团的 walle 作为多渠道打包的方案，在编译阶段会报错

**错误提示：**
```
API 'variantOutput.getPackageApplication()' is obsolete and has been replaced with 'variant.getPackageApplicationProvider()'. It will be removed at the end of 2019.
```

**解决方案：**

walle 已经修复了该[问题](https://github.com/Meituan-Dianping/walle/pull/285)，详情请查阅 [fixAPI 'variant.getAssemble()' is obsolete and has been replaced with…](https://github.com/Meituan-Dianping/walle/commit/c3869bbce43254c2fd44d67edf81fc9ea925b037)<br>
但 walle 并未将修复的版本发布(摊手)，可以自行下载[源码](https://github.com/Meituan-Dianping/walle)编译到自己的仓库中引用

## 如何生产进行实时 lintRules 的 aar 

```    
发布 aar，直接运行 `lintRules-uploadArchives` 的 task 即可发布到仓库中
```

### 备注
① ${root} 即项目的根地址文件夹名，请按需修改

### 感谢
本文是在巨人的肩膀上进行探索并实践的，感谢 <br>
①[lsc1993](https://github.com/lsc1993) 的[AwesomeLint](https://github.com/lsc1993/AwesomeLint) <br>
②[GitCode8](https://juejin.im/user/5995c9f2f265da248c3934a5) 的 [代码洁癖症的我，学习Lint学到心态爆炸](https://juejin.im/post/5d307615f265da1b6b1d0dd9)<br>
本实践也是在该基础上进行改进并应用到项目中的，代码也是 fork 自该项目。在其基础上进行添加了对 Kotlin 的支持，并在应用到项目中时踩了很多坑也将其填完了。借此也将填坑经验分享出来，以供借鉴<br>

**诚惶诚恐，若有错误，不吝赐教**
