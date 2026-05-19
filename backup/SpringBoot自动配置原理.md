1. 在SpringBoot项目中的引导类上有一个注解@SpringBootApplication，这个注解是对三个注解进行了封装
	- `@SpringBootConfiguration`
	- `@EnableAutoConfiguration`
	- `@ComponentScan`
2. 其中 `@EnableAutoConfiguration` 是实现自动化配置的核心注解，该注解通过 `@Import` 注解导入对应的配置选择器。内部就是读取了该项目和该项目引用的Jar包的classpath路径下的 `META-INF/spring.factories` 文件中的所配置的类的全类名。在这些配置类中所定义的Bean会根据条件注解所指定的条件来决定是否需要将其导入到Spring容器中。
3. 条件判读会有像 `@ConditionOnClass` 这样的注解，判断是否有对应的class文件，如果有则加载该类，把这个配置类的所有的Bean放入Spring容器中使用。