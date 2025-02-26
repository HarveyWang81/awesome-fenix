# 没有虚拟机的 Java

尽管 Java 已经看清楚了在微服务时代的前进目标，但是，Java 语言和生态在微服务、微应用环境中的天生的劣势并不会一蹴而就地被解决，通往这个目标的道路注定会充满荆棘；尽管已经有了放弃“一次编写，到处运行”、放弃语言动态性的思想准备，但是，这些特性并不单纯是宣传口号，它们在 Java 语言诞生之初就被植入到基因之中，当 Graal VM 试图打破这些规则的同时，也受到了 Java 语言和在其之上的生态生态的强烈反噬，笔者选择其中最主要的一些困难列举如下：

- 某些 Java 语言的特性，使得 Graal VM 编译本地镜像的过程变得极为艰难。譬如常见的反射，除非使用[安全管理器](/architect-perspective/general-architecture/system-security/authentication.html)去专门进行认证许可，否则反射机制具有在运行期动态调用几乎所有 API 接口的能力，且具体会调用哪些接口，在程序不会真正运行起来的编译期是无法获知的。反射显然是 Java 不能放弃不能妥协的重要特性，为此，只能由程序的开发者明确地告知 Graal VM 有哪些代码可能被反射调用（通过 JSON 配置文件的形式），Graal VM 才能在编译本地程序时将它们囊括进来。

  ```json
  [
      {
          name: "com.github.fenixsoft.SomeClass",
          allDeclaredConstructors: true,
          allPublicMethods: true
      },
      {
          name: "com.github.fenixsoft.AnotherClass",
          fileds: [{name: "foo"}, {name: "bar"}],
          methods: [{
              name: "<init>",
              parameterTypes: ["char[]"]
          }]
      },
      // something else ……
  ]
  ```

  这是一种可操作性极其低下却又无可奈何的解决方案，即使开发者接受不厌其烦地列举出自己代码中所用到的反射 API，但他们又如何能保证程序所引用的其他类库的反射行为都已全部被获知，其中没有任何遗漏？与此类似的还有另外一些语言特性，如动态代理等。另外，一切非代码性质的资源，如最典型的配置文件等，也都必须明确加入配置中才能被 Graal VM 编译打包。这导致了如果没有专门的工具去协助，使用 Graal VM 编译 Java 的遗留系统即使理论可行，实际操作也将是极度的繁琐。

- 大多数运行期对字节码的生成和修改操作，在 Graal VM 看来都是无法接受的，因为 Substrate VM 里面不再包含即时编译器和字节码执行引擎，所以一切可能被运行的字节码，都必须经过 AOT 编译成为原生代码。请不要觉得运行期直接生成字节码会很罕见，误以为导致的影响应该不算很大。事实上，多数实际用于生产的 Java 系统都或直接或间接、或多或少引用了 ASM、CGLIB、Javassist 这类字节码库。举个例子，CGLIB 是通过运行时产生字节码（生成代理类的子类）来做动态代理的，长期以来这都是 Java 世界里进行类增强的主流形式，因为面向接口的增强可以使用 JDK 自带的动态代理，但对类的增强则并没有多少选择的余地。CGLIB 也是 Spring 用来做类增强的选择，但 Graal VM 明确表示是不可能支持 CGLIB 的，因此，这点就必须由用户（面向接口编程）、框架（Spring 这些 DI 框架放弃 CGLIB 增强）和 Graal VM（起码得支持 JDK 的动态代理，留条活路可走）来共同解决。自 Spring Framework 5.2 起，@Configuration 注解中加入了一个新的 proxyBeanMethods 参数，设置为 false 则可避免 Spring 对与非接口类型的 Bean 进行代理。同样地，对应在 Spring Boot 2.2 中，@SpringBootApplication 注解也增加了 proxyBeanMethods 参数，通常采用 Graal VM 去构建的 Spring Boot 本地应用都需要设置该参数。

- 一切 HotSpot 虚拟机本身的内部接口，譬如 JVMTI、JVMCI 等，在都将不复存在了——在本地镜像中，连 HotSpot 本身都被消灭了，这些接口自然成了无根之木。这对使用者一侧的最大影响是再也无法进行 Java 语言层次的远程调试了，最多只能进行汇编层次的调试。在生产系统中一般也没有人这样做，开发环境就没必要采用 Graal VM 编译，这点的实际影响并不算大。

- Graal VM 放弃了一部分可以妥协的语言和平台层面的特性，譬如 Finalizer、安全管理器、InvokeDynamic 指令和 MethodHandles，等等，在 Graal VM 中都被声明为不支持的，这些妥协的内容大多倒并非全然无法解决，主要是基于工作量性价比的原因。能够被放弃的语言特性，说明确实是影响范围非常小的，所以这个对使用者来说一般是可以接受的。

- ……

以上，是 Graal VM 在 Java 语言中面临的部分困难，在整个 Java 的生态系统中，数量庞大的第三方库才是真正最棘手的难题。可以预料，这些第三方库一旦脱离了 Java 虚拟机，在原生环境中肯定会暴露出无数千奇百怪的异常行为。Graal VM 团队对此的态度非常务实，并没有直接硬啃。要建设可持续、可维护的 Graal VM，就不能为了兼容现有 JVM 生态，做出过多的会影响性能、优化空间和未来拓展的妥协牺牲，为此，应该也只能反过来由 Java 生态去适应 Graal VM，这是 Graal VM 团队明确传递出对第三方库的态度：

:::quote 3rd party libraries

Graal VM native support needs to be sustainable and maintainable, that's why we do not want to maintain fragile pathches for the whole JVM ecosystem.

The ecosystem of libraries needs to support it natively.

::: right

—— Sébastien Deleuze，[DEVOXX 2019](https://www.youtube.com/watch?v=3eoAxphAUIg)

:::

为了推进 Java 生态向 Graal VM 兼容，Graal VM 主动拉拢了 Java 生态中最庞大的一个派系：Spring。从 2018 年起，来自 Oracle 的 Graal VM 团队与来自 Pivotal 的 Spring 团队已经紧密合作了很长的一段时间，共同创建了[Spring Graal Native](https://github.com/spring-projects-experimental/spring-graal-native)项目来解决 Spring 全家桶在 Graal VM 上的运行适配问题，在不久的将来（预计应该是 2020 年 10 月左右），下一个大的 Spring 版本（Spring Framework 5.3、Spring Boot 2.3）的其中一项主要改进就是能够开箱即用地支持 Graal VM，这样，用于微服务环境的 Spring Cloud 便会获得不受 Java 虚拟机束缚的更广阔舞台空间。
