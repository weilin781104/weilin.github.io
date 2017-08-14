# Spring Framework
## 属性来源
在Srping Framework中，属性由抽象Enviroment统一管理，Enviroment对象跟profile绑定，管理了所有的属性，由一组PropertySource提供。默认的PropertySource是StandEnviroment（或者StandServletEnviroment等），它由两部分组成（JVM属性（System.getProperties()）和系统属性（System.getenv()））。用户也可以自定义属性，通过@PropertySource引入到Enviroment中。 多个PropertySource中相同属性根据优先级定。
## 属性获取
   ```
   @Autowired
   Environment env;
   
   env.getProperty("")
   ```
# Spring Boot
##  也是用Enviroment来管理所有属性的。PropertySource更多，有优先级
1. Devtools global settings properties on your home directory (~/.spring-bootdevtools.properties when devtools is active).
2. @TestPropertySource annotations on your tests.
3. @SpringBootTest#properties annotation attribute on your tests.
4. Command line arguments.
5. Properties from SPRING_APPLICATION_JSON (inline JSON embedded in an environment variable or system property)
6. ServletConfig init parameters.
7. ServletContext init parameters.
8. JNDI attributes from java:comp/env.
9. Java System properties (System.getProperties()).
10. OS environment variables.
11. A RandomValuePropertySource that only has properties in random.*.
12. Profile-specific application properties outside of your packaged jar (application-{profile}.properties and YAML variants)
13. Profile-specific application properties packaged inside your jar (application-{profile}.properties and YAML variants)
14. Application properties outside of your packaged jar (application.properties and YAMLvariants).
15 .Application properties packaged inside your jar (application.properties and YAML variants).
16. @PropertySource annotations on your @Configuration classes.
17. Default properties (specified using SpringApplication.setDefaultProperties).

## 属性获取
Property values can be injected directly into your beans using the @Value annotation, accessed via Spring’s Environment abstraction or bound to structured objects via @ConfigurationProperties.

- 取某个属性值
```
@Component
public class MyBean {
  @Value("${name}")
  private String name;
  // ...
  }
```

- 把特定前缀的属性值自动映射成对象的字段
```
@ConfigurationProperties(prefix = "bar")
@Bean
public BarComponent barComponent() {
...
}
```
