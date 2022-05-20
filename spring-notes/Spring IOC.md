# Spring IOC

> Spring：5.3.15
>
> SpringBoot：2.6.3
>
> [Spring Framework Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/)
>
> [Spring Boot Reference Documentation](https://docs.spring.io/spring-boot/docs/2.6.3/reference/html/)

## IOC 概念

**IOC**：Inversion of Control，控制反转，是面向对象编程中的一种设计原则，可以用来**减低代码之间的耦合度**。最常见的方式叫做依赖注入（DI）。对象在被创建的时候，由IOC容器将对象所依赖的对象的引用传递给它，即将依赖注入到对象中。是一种思想。

**DI**：Dependency Injection，依赖注入，是IOC的一种实现。

> [浅谈IOC-说清楚IOC是什么](https://www.cnblogs.com/DebugLZQ/archive/2013/06/05/3107957.html)

IOC本身是一个容器，可以认为是一个很强大的HashMap，除了能存对象，还能对对象进行各种装配修饰。

