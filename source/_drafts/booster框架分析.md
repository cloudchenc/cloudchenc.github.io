---
title: booster框架分析
tags:
    - booster
    - Gradle
    - SPI
categories: Gradle
---

一、简介
在去年做包体优化的时候我们使用到了didi的booster开源框架，这个框架具体起到了什么作用，大家应该是不知道的，所以这里就做下其原理的分享

首先，booster是一款质量优化框架，是为了解决随着APP复杂度的提升而带来的性能、稳定性、包体积等质量问题。
这些描述是官方的，简单来说就是插件平台，针对APP的质量问题，独立出了各个插件

二、功能特性
性能优化
● 静态分析，booster-task-analyser - 静态分析工具
● 多线程优化，booster-transform-thread - 多线程优化
● SharedPreferences优化，booster-transform-shared-preferences - SharedPreferences 优化
● WebView预加载，booster-transform-webview - WebView 预加载
包体瘦身
2.1 图片压缩
booster-task-compression-cwebp - 采用 cwebp 对资源进行压缩
booster-task-compression-pngquant - 采用 pngquant 对资源进行压缩
2.2 ZIP文件压缩 
booster-task-compression-processed-res - ap_ 文件压缩
2.3 索引内联
booster-transform-r-inline - 资源索引内联
booster-transform-br-inline - DataBinding BR索引内联
2.4 移除冗余资源
booster-task-resource-deredundancy - 去冗余资源
修复系统bug
3.1 系统崩溃兜底
booster-transform-activity-thread - 处理系统 Crash
booster-transform-media-player - 修复 MediaPlayer 崩溃
3.2 Finalizer导致的TimeoutException
booster-transform-finalizer-watchdog-daemon - 修复 finalizer 导致的 TimeoutException
3.3 资源为null的问题
booster-transform-res-check - 检查覆盖安装导致的 Resources 和 Assets 未加载的 Bug
3.4 Android 7.1 Toast 崩溃
booster-transform-toast - 修复 Toast 在 Android 7.1 上的 Bug
三、框架分析
设计亮点：
● 利用 SPI 实现 Feature 模块化 & 插件化
● 自定义 Transfomer 实现可扩展性
● 支持注入复杂的类库

网上大多是根据插件功能相应的架构图，这是我根据自己理解画的一张工程整体的架构图


在分析框架前，需要事先了解以下知识点：
Gradle构建
Gradle的构建生命周期分为三个过程：Initialization、Configuration、Execution。
https://docs.gradle.org/7.3/userguide/build_lifecycle.html1.1 Initialization
在初始化阶段，gradle要确定的一件比较重要的事是:这次构建是单工程构建还是多工程。即gradle会去寻找setting.gradle文件, 找的大致逻辑是：
1. 当前目录查找setting.gradle文件，查找到后开始构建
2. 如果当前目录没有，则找当前目录的父目录有没有，如果有，则按照父目录的setting.gradle开始构建
3. 如果没有，则如果当前目录存在build.gradle,进行单工程构建
确定好是单工程构建还是多工程构建后就会把所有参与构建的工程生成对应的Project对象。

1.2 Configuration
在这个阶段会对build.gralde编写的脚本进行逐行执行，会进行如下任务
1. 加载插件
2. 加载依赖
3. 加载task
4. 执行脚本，自定义的插件DSL代码，本身gradle支持的API等
5. 构建好所有task的拓扑图（有向无环图）

1.3 Execution
根据拓扑图，执行具体的的task及其依赖task

Gradle生命周期的回调监听
project.beforeEvaluate{
    // 配置阶段开始前回调监听
}

project.afterEvaluate{
    // 配置阶段完成后回调监听
}

project.gradle.buildFinished{
    // gradle执行完毕后的监听回调
}

2. SPI与ServiceLoader
要了解booster，首先要从SPI（Service Provider Interface）开始，它是JDK内置的API，在运行时自动加载接口对应的实现，所以booster用它来开发可拓展程序是非常方便的。

像JDK本身也是很多地方使用到了SPI，至于它的原理，其实就是从JAR包中的 ‘META-INF/services/’ 目录下，有一系列以接口类名命名的文件，文件的内容则是该接口的实现类，所以在SPI加载接口实现时，会通过ServiceLoader实际调用到 ClassLoader 中的 getResourcesXxx(String) 相关的方法，一般使用Google开源的 autoservice APT（Annotation Processing Tool）工具，在编译期间自动生成配置文件到 ‘META-INF/services/’ 目录

如下图就是 yuer v5.29.5版本APK解压后的‘META-INF/services/’ 目录

以‘kotlinx.coroutines.CoroutineExceptionHandler’为例，打开文件会看到真正的实现是在‘kotlinx.coroutines.android.AndroidExceptionPreHandler’


3. booster-gradle-plugin
好了，聊完上面的知识点就开始正式进入booster框架，而一个gradle插件的开发是从实现Plugin类开始的
3.1 BoosterPlugin.kt
class BoosterPlugin : Plugin<Project> {

    override fun apply(project: Project) {
        project.gradle.addListener(BoosterTransformTaskExecutionListener(project))

        when {
            project.plugins.hasPlugin("com.android.application") || project.plugins.hasPlugin("com.android.dynamic-feature") -> project.getAndroid<AppExtension>().let { android ->
                android.registerTransform(BoosterTransform.newInstance(project))
                project.afterEvaluate {
                    loadVariantProcessors(project).let { processors ->
                        android.applicationVariants.forEach { variant ->
                            processors.forEach { processor ->
                                processor.process(variant)
                            }
                        }
                    }
                }
            }
            project.plugins.hasPlugin("com.android.library") -> project.getAndroid<LibraryExtension>().let { android ->
                android.registerTransform(BoosterTransform.newInstance(project))
                project.afterEvaluate {
                    loadVariantProcessors(project).let { processors ->
                        android.libraryVariants.forEach { variant ->
                            processors.forEach { processor ->
                                processor.process(variant)
                            }
                        }
                    }
                }
            }
        }
    }
}
根据android app和library模块，注册BoosterTransform，同时设置回调在gradle进行到afterEvaluate生命周期时，利用Java的ServiceLoader，运行时（Gradle本身是在JVM运行）动态查找VariantProcessor的实现对象列表，遍历并调用processor()方法。

3.2 BoosterTransform.kt
open class BoosterTransform protected constructor(val project: Project) : Transform() {

    internal val transformers = loadTransformers(project.buildscript.classLoader).sortedBy {
        it.javaClass.getAnnotation(Priority::class.java)?.value ?: 0
    }
    
    override fun getName() = "booster"

    override fun isIncremental() = !verifyEnabled

    override fun isCacheable() = !verifyEnabled

    override fun getInputTypes(): MutableSet<QualifiedContent.ContentType> = TransformManager.CONTENT_CLASS

    final override fun transform(invocation: TransformInvocation) {
        BoosterTransformInvocation(invocation, this).apply {
            if (isIncremental) {
                doIncrementalTransform()
            } else {
                outputProvider?.deleteAll()
                doFullTransform()
            }
        }
    }
}
1. 关键变量transformers使用SPI机制，读取Transformer接口的实现列表，并根据优先级排序
2. 调用过程支持Gradle的增量编译和TransformTask输出的缓存，在transform()方法的实现中，把回调的TransformInvocation对象代理给BoosterTransformInvocation，真正的执行过程发生在BoosterTransformInvocation中

3.3 BoosterTransformInvocation.kt
internal class BoosterTransformInvocation(
        private val delegate: TransformInvocation,
        internal val transform: BoosterTransform
) : TransformInvocation by delegate, TransformContext, ArtifactManager {

    internal fun doFullTransform() = doTransform(this::transformFully)

    internal fun doIncrementalTransform() = doTransform(this::transformIncrementally)
   
}
代理默认的TransformInvocation对象，并进行扩展，另外还实现了booster-transform-spi模块的相关接口:
● TransformContext: 封装了Booster在Transform过程中的上下文数据:类路径、文件目录、应用信息、调试信息等
● TransformListener: 定义transform的前后监听回调方法onPreTransform()和 onPostTransform()
● ArtifactManager: 对Artifact的管理，具体为构建过程的各种文件，get()方法使用VariantScope的各种方法一一对应实现。
对于关键doFullTransform()和 doIncrementalTransform()方法，主要了解两点：
1. 如何实现增量编译的兼容
1. 具体逻辑流程
增量编译：
onPreTransform() -> doIncrementalTransform() -> onPostTransform() 
全量编译：
onPreTransform() -> doFullTransform() -> onPreTransform()
构建过程中Transform的回调，通过inputs:List<TransformInput>属性获取资源。
public interface TransformInput {

    /**
     * Returns a collection of {@link JarInput}.
     */
    @NonNull
    Collection<JarInput> getJarInputs();

    /**
     * Returns a collection of {@link DirectoryInput}.
     */
    @NonNull
    Collection<DirectoryInput> getDirectoryInputs();
}
JarInput和DirectoryInput都是QualifiedContent的子接口，提供getStatus()方法获取资源的状态：
public enum Status {
    /**
     * The file was not changed since the last build.
     */
    NOTCHANGED,
    /**
     * The file was added since the last build.
     */
    ADDED,
    /**
     * The file was modified since the last build.
     */
    CHANGED,
    /**
     * The file was removed since the last build.
     */
    REMOVED;
}
增量编译是发生在两次编译之间的过程，这四种状态包含了每个文件、文件夹的变化状态。
无论是JarInput还是DirectoryInput，都是这样兼容增量构建。
    private fun doIncrementalTransform(xxx: Xxx) {
        when (xxx.status) {
            REMOVED -> // delete
            CHANGED, ADDED -> // compile
            NOTCHANGED -> // nothing to do
        }
    }
下面来看看input的内容是如何使用的。
    private fun QualifiedContent.transform(output: File) {
        outputs += output
        this.file.transform(output) { bytecode ->
            bytecode.transform()
        }
    }

    private fun ByteArray.transform(): ByteArray {
        return transform.transformers.fold(this) { bytes, transformer ->
            transformer.transform(this@BoosterTransformInvocation, bytes)
        }
    }
将每个文件通过File的拓展函数转为bytecode后，由#BoosterTransform.kt中的变量transformers传递到下层SPI具体实现的Transformer中
4. booster-transform-spi
4.1 Transformer
interface Transformer : TransformListener {

    /**
     * Returns the transformed bytecode
     *
     * @param context
     *         The transforming context
     * @param bytecode
     *         The bytecode to be transformed
     * @return the transformed bytecode
     */
    fun transform(context: TransformContext, bytecode: ByteArray): ByteArray

}

@AutoService(Transformer::class)
class AsmTransformer : Transformer {}

@AutoService(Transformer::class)
class JavassistTransformer : Transformer {}
使用SPI机制，将transform()方法的执行发生在AsmTransformer类和JavassistTransformer类中

4.2 AsmTransformer
@AutoService(Transformer::class)
class AsmTransformer : Transformer {

    internal val transformers: Iterable<ClassTransformer>

    override fun transform(context: TransformContext, bytecode: ByteArray): ByteArray {
        return ClassWriter(ClassWriter.COMPUTE_MAXS).also { writer ->
            this.transformers.fold(ClassNode().also { klass ->
                ClassReader(bytecode).accept(klass, 0)
            }) { a, transformer ->
                this.threadMxBean.sumCpuTime(transformer) {
                     transformer.transform(context, a)
                }
            }.accept(writer)
        }.toByteArray()
    }
}
在这一层的职责，是把ByteArray -> ClassNode，暴露给下一层进行方法修改操作，然后转换回ByteArray返回上层。
使用auto-service库的@AutoService注解，声明AsmTransform按照SPI的规范生成相关的META-INF文件。
在build.gradle中声明依赖:
kapt "com.google.auto.service:auto-service:1.0-rc4"
compile 'com.google.auto.service:auto-service:1.0-rc4'
保证了前面的BoosterTransformInvocation能通过ServiceLoader加载数据。

ClassTransform是其他booster-transform-xxx模块，需要进行字节码插桩时候，需要具体实现的接口。
interface ClassTransformer : TransformListener {

    /**
     * Transform the specified class node
     *
     * @param context The transform context
     * @param klass The class node to be transformed
     * @return The transformed class node
     */
    fun transform(context: TransformContext, klass: ClassNode) = klass

}

@AutoService(ClassTransformer::class)
实现者注意要使用@AutoService注解，不然AsmTransform是找不到对应类的。

4.3 JavassistTransformer 
与AsmTransformer逻辑类似，只是为了提供了基于 *Javassist* 的字节码操作相关的实用类和扩展属性及方法

5. booster-task-spi
VariantProcessor是其他booster-task-xxx模块，需要执行自定义task时候，需要具体实现的接口。
interface VariantProcessor {

    fun process(variant: BaseVariant)

}

@AutoService(VariantProcessor::class)
通过BaseVariant，添加自定义task，并作为编译task的依赖，保证自动执行

6. booster-task-compression
这里以项目中使用到的包体优化为例，分析booster task是如何进行图片压缩的
6.1 booster-pngquant-provider
基于压缩效果和使用便利性的考虑，booster使用了开源软件pngquant来进行图片压缩。
但是pngquant是GPL License，而booster是Apache License，所以就独立出了booster-pngquant-provider来作为pngquant命令的提供者，采用GPL Licence 独立开源，目前内置了 linux/macos/windows 三端的 pngquant 可执行文件

p.s. 
Apache 可用于商用软件，闭源
GPL具有传染性，相关的都要开源
faker.js是MIT协议，属于“白嫖”协议


6.2 booster-command
而为了解决Apache和GPL开源协议的冲突问题，booster提供了 booster-command 用于发现和调用外部命令的 SPI，允许开发者来实现自定义的外部命令提供者。
其原理是将API与实现进行分离，API采用Apache License，实现采用GPL License
#booster-command
interface CommandProvider {

    /**
     * returns a mapping of command name and url
     */
    fun get(): Collection<Command>

    /**
     * returns a command with the specific name
     *
     * @param name The command name
     */
    operator fun get(name: String): Command? = get().find { it.name == name }

}
#booster-pngquant-provider
@AutoService(CommandProvider::class)
class PngquantProvider : CommandProvider {

    override fun get(): Collection<Command> = listOf(
            Command(PNGQUANT, javaClass.classLoader.getResource(PREBULT_PNGQUANT_EXECUTABLE)!!)
    )

}


6.3 booster-task-compression-pngquant
@AutoService(VariantProcessor::class)
@Priority(1)
class PngquantCompressionVariantProcessor : VariantProcessor {

    override fun process(variant: BaseVariant) {
        val project = variant.project
        val results = CompressionResults()
        val ignores = project.findProperty(PROPERTY_IGNORES)?.toString()?.trim()?.split(',')?.map {
            Wildcard(it)
        }?.toSet() ?: emptySet()

        Pngquant.get(variant)?.newCompressionTaskCreator()?.createCompressionTask(variant, results, "resources", {
            variant.mergedRes.search(if (project.isAapt2Enabled) ::isFlatPngExceptRaw else ::isPngExceptRaw)
        }, ignores, variant.mergeResourcesTaskProvider)?.configure { task ->
            variant.project.tasks.withType(CompressImages::class.java).filter {
                it.name != task.name && it.variant.name == variant.name
            }.takeIf {
                it.isNotEmpty()
            }?.let {
                task.dependsOn(*it.toTypedArray())
            }
            task.doLast {
                results.generateReport(variant, Build.ARTIFACT)
            }
        }
    }
}
通过autoservice注解，将图片压缩的执行放在PngquantCompressionVariantProcessor中，通过调用Pngquant命令行真正实现图片压缩

6.4 实现思路

根据 Android 的构建流程，我们选择在 mergeRes 和 processRes 任务之间插入图片压缩任务，如下图所示：
  
自 AGP 3.0.0 开始，AAPT2 默认会被开启，由于 AAPT2 采用了 AAPT2 Container 格式在新窗口打开 (文件后缀为 .flat) 所以，需要在压缩之前需要明确，项目中是否开启 android.enableAapt2 的选项，因此 Booster 根据 AAPT2 选项分不同的 task 来处理：
 

6.5 PngquantCompressionVariantProcessor.kt
@AutoService(VariantProcessor::class)
@Priority(1)
class PngquantCompressionVariantProcessor : VariantProcessor {

    override fun process(variant: BaseVariant) {
        val project = variant.project
        val results = CompressionResults()
        val ignores = project.findProperty(PROPERTY_IGNORES)?.toString()?.trim()?.split(',')?.map {
            Wildcard(it)
        }?.toSet() ?: emptySet()

        Pngquant.get(variant)?.newCompressionTaskCreator()?.createCompressionTask(variant, results, "resources", {
            variant.mergedRes.search(if (project.isAapt2Enabled) ::isFlatPngExceptRaw else ::isPngExceptRaw)
        }, ignores, variant.mergeResourcesTaskProvider)?.configure { task ->
            variant.project.tasks.withType(CompressImages::class.java).filter {
                it.name != task.name && it.variant.name == variant.name
            }.takeIf {
                it.isNotEmpty()
            }?.let {
                task.dependsOn(*it.toTypedArray())
            }
            task.doLast {
                results.generateReport(variant, Build.ARTIFACT)
            }
        }
    }
}

四、总结
以上的分析都只是booster框架的大致脉络，大部分的插件实现没有涉及到，感兴趣的话还是需要大家下面再去了解。

现在再看这张图，是不是就觉得十分清晰？

