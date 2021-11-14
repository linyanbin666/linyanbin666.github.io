---
title: "Spring Data JPA多数据源配置"
date: 2021-10-28T13:39:51+08:00
tags: ["JPA", "多数据源"]
categories: ["Spring"]
draft: false
---

> 每日一言：知识是勤奋的影子，汗珠是勤奋的镜子  

### 配置多数据源
#### maven依赖

```java
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.1.0.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
<dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
</dependencies>
```
#### application.yml文件

```java
spring:
  datasource:
    primary:
      name: primaryDataSource
      driver-class-name: org.h2.Driver
      url: jdbc:h2:mem:test1;MODE=MySQL
      username: root
      password: root
      jpa:
        show-sql: true
        generate-ddl: true
        properties:
          hibernate.dialect: org.hibernate.dialect.H2Dialect
        hibernate:
          ddl-auto: update
    secondary:
      name: secondaryDataSource
      driver-class-name: org.h2.Driver
      url: jdbc:h2:mem:test2;MODE=MySQL
      username: root
      password: root
      jpa:
        show-sql: true
        generate-ddl: true
        properties:
          hibernate.dialect: org.hibernate.dialect.H2Dialect
        hibernate:
          ddl-auto: update
```
#### Java Config  
第一个数据源配置类
```java
@Configuration
@EnableJpaRepositories(
    entityManagerFactoryRef = "primaryEntityManager",
    transactionManagerRef = "primaryTransactionManager",
    basePackageClasses = {com.horizon.article.demo.jpa.primary.repo.Package.class})
public class PrimaryDsConfiguration {

  @Bean("primaryDataSourceProperties")
  @ConfigurationProperties(prefix = "spring.datasource.primary")
  public DataSourceProperties primaryDataSourceProperties() {
    return new DataSourceProperties();
  }

  @Bean("primaryDataSource")
  public DataSource primaryDataSource() {
    return primaryDataSourceProperties().initializeDataSourceBuilder().build();
  }

  @Bean("primaryJpaProperties")
  @ConfigurationProperties(prefix = "spring.datasource.primary.jpa")
  public JpaProperties primaryJpaProperties() {
    return new JpaProperties();
  }

  @Bean("primaryHibernateProperties")
  @ConfigurationProperties(prefix = "spring.datasource.primary.jpa.hibernate")
  public HibernateProperties primaryHibernateProperties() {
    return new HibernateProperties();
  }

  @Bean("primaryEntityManager")
  public LocalContainerEntityManagerFactoryBean primaryEntityManager() {
    final LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
    em.setPackagesToScan(com.horizon.article.demo.jpa.primary.bean.Package.class.getPackage().getName()
    );
    em.setDataSource(primaryDataSource());
    em.setJpaVendorAdapter(primaryVendorAdapter());
    Map<String, Object> properties = primaryHibernateProperties()
        .determineHibernateProperties(
            primaryJpaProperties().getProperties(), new HibernateSettings()
        );
    em.setJpaPropertyMap(properties);
    em.setPersistenceUnitName("primary");
    return em;
  }

  @Bean("primaryVendorAdapter")
  public HibernateJpaVendorAdapter primaryVendorAdapter() {
    final HibernateJpaVendorAdapter adapter = new HibernateJpaVendorAdapter();
    adapter.setGenerateDdl(false);
    return adapter;
  }

  @Bean("primaryTransactionManager")
  public PlatformTransactionManager primaryTransactionManager() {
    final JpaTransactionManager transactionManager = new JpaTransactionManager();
    transactionManager.setEntityManagerFactory(primaryEntityManager().getObject());
    return transactionManager;
  }

  @Bean
  @Qualifier("primaryJdbcTemplate")
  public JdbcTemplate primaryJdbcTemplate() {
    return new JdbcTemplate(primaryDataSource());
  }
}
```
以同样的方式配置第二个数据源
```java
@Configuration
@EnableJpaRepositories(
    entityManagerFactoryRef = "secondaryEntityManager",
    transactionManagerRef = "secondaryTransactionManager",
    basePackageClasses = {com.horizon.article.demo.jpa.secondary.repo.Package.class})
public class SecondaryDsConfiguration {

  @Bean("secondaryDataSourceProperties")
  @ConfigurationProperties(prefix = "spring.datasource.secondary")
  public DataSourceProperties secondaryDataSourceProperties() {
    return new DataSourceProperties();
  }

  @Bean("secondaryDataSource")
  public DataSource secondaryDataSource() {
    return secondaryDataSourceProperties().initializeDataSourceBuilder().build();
  }

  @Bean("secondaryJpaProperties")
  @ConfigurationProperties(prefix = "spring.datasource.secondary.jpa")
  public JpaProperties secondaryJpaProperties() {
    return new JpaProperties();
  }

  @Bean("secondaryHibernateProperties")
  @ConfigurationProperties(prefix = "spring.datasource.secondary.jpa.hibernate")
  public HibernateProperties secondaryHibernateProperties() {
    return new HibernateProperties();
  }

  @Bean("secondaryEntityManager")
  public LocalContainerEntityManagerFactoryBean secondaryEntityManager() {
    final LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
    em.setPackagesToScan(com.horizon.article.demo.jpa.secondary.bean.Package.class.getPackage().getName()
    );
    em.setDataSource(secondaryDataSource());
    em.setJpaVendorAdapter(secondaryVendorAdapter());
    Map<String, Object> properties = secondaryHibernateProperties()
        .determineHibernateProperties(
            secondaryJpaProperties().getProperties(), new HibernateSettings()
        );
    em.setJpaPropertyMap(properties);
    em.setPersistenceUnitName("secondary");
    return em;
  }

  @Bean("secondaryVendorAdapter")
  public HibernateJpaVendorAdapter secondaryVendorAdapter() {
    final HibernateJpaVendorAdapter adapter = new HibernateJpaVendorAdapter();
    adapter.setGenerateDdl(false);
    return adapter;
  }

  @Bean("secondaryTransactionManager")
  public PlatformTransactionManager secondaryTransactionManager() {
    final JpaTransactionManager transactionManager = new JpaTransactionManager();
    transactionManager.setEntityManagerFactory(secondaryEntityManager().getObject());
    return transactionManager;
  }

  @Bean
  @Qualifier("secondaryJdbcTemplate")
  public JdbcTemplate secondaryJdbcTemplate() {
    return new JdbcTemplate(secondaryDataSource());
  }
}
```
排除SpringBoot本身的数据源自动配置及JPA自动配置
```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    DataSourceTransactionManagerAutoConfiguration.class,
    JdbcTemplateAutoConfiguration.class,
    HibernateJpaAutoConfiguration.class
})
public class Application {

  public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
  }

}
```
[完整示例代码地址](https://github.com/linyanbin666/article-demo-code/tree/master/src/main/java/com/horizon/article/demo/jpa)
### 配置动态切换数据源
#### 使用枚举（或字符串）定义不同数据源的唯一标识
```java
public enum DatabaseEnvironment {
    DEVELOPMENT, TESTING, PRODUCTION
}
```
#### 使用ThreadLocal存放当前线程持有的环境
```java
public class DatabaseContextHolder {

    private static final ThreadLocal<DatabaseEnvironment> CONTEXT =
        new ThreadLocal<>();

    public static void set(DatabaseEnvironment databaseEnvironment) {
        CONTEXT.set(databaseEnvironment);
    }

    public static DatabaseEnvironment getEnvironment() {
        return CONTEXT.get();
    }

    public static void clear() {
        CONTEXT.remove();
    }

}
```
#### 继承AbstractRoutingDataSource类，创建自定义路由数据源
```java
public class DataSourceRouter extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return DatabaseContextHolder.getEnvironment();
    }
}
```
#### Java Config配置路由数据源
```java
@Configuration
@EnableJpaRepositories(
    basePackageClasses = CustomerRepository.class,
    entityManagerFactoryRef = "customerEntityManager",
    transactionManagerRef = "customerTransactionManager")
public class DataSourceConfiguration {

    @Autowired(required = false)
    private PersistenceUnitManager persistenceUnitManager;

    @Bean
    @ConfigurationProperties("app.customer.jpa")
    public JpaProperties customerJpaProperties() {
        return new JpaProperties();
    }

    @Bean
    @ConfigurationProperties("app.customer.jpa.hibernate")
    public HibernateProperties customerHibernateProperties() {
        return new HibernateProperties();
    }

    @Bean
    @ConfigurationProperties("app.customer.development.datasource")
    public DataSourceProperties developmentDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean
    public DataSource developmentDataSource() {
        return developmentDataSourceProperties().initializeDataSourceBuilder().build();
    }

    @Bean
    @ConfigurationProperties("app.customer.testing.datasource")
    public DataSourceProperties testingDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean
    public DataSource testingDataSource() {
        return testingDataSourceProperties().initializeDataSourceBuilder().build();
    }

    @Bean
    @ConfigurationProperties("app.customer.production.datasource")
    public DataSourceProperties productionDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean
    public DataSource productionDataSource() {
        return testingDataSourceProperties().initializeDataSourceBuilder().build();
    }

    /**
     * Adds all available datasources to datasource map.
     *
     * @return datasource of current context
     */
    @Bean
    @Primary
    public DataSource customerDataSource() {
        DataSourceRouter router = new DataSourceRouter();

        final HashMap<Object, Object> map = new HashMap<>(3);
        map.put(DatabaseEnvironment.DEVELOPMENT, developmentDataSource());
        map.put(DatabaseEnvironment.TESTING, testingDataSource());
        map.put(DatabaseEnvironment.PRODUCTION, productionDataSource());
        router.setTargetDataSources(map);
        return router;
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean customerEntityManager(
        @Qualifier("customerJpaProperties") final JpaProperties customerJpaProperties) {
        EntityManagerFactoryBuilder builder =
            createEntityManagerFactoryBuilder(customerJpaProperties);

        return builder.dataSource(customerDataSource()).packages(Customer.class)
            .properties(customerHibernateProperties().determineHibernateProperties(
                customerJpaProperties.getProperties(), new HibernateSettings()
            )).persistenceUnit("customerEntityManager").build();
    }

    @Bean
    @Primary
    public JpaTransactionManager customerTransactionManager(
        @Qualifier("customerEntityManager") final EntityManagerFactory factory) {
        return new JpaTransactionManager(factory);
    }

    private EntityManagerFactoryBuilder createEntityManagerFactoryBuilder(
        final JpaProperties customerJpaProperties) {
        JpaVendorAdapter jpaVendorAdapter =
            createJpaVendorAdapter(customerJpaProperties);
        return new EntityManagerFactoryBuilder(jpaVendorAdapter,
            customerJpaProperties.getProperties(), persistenceUnitManager);
    }

    private JpaVendorAdapter createJpaVendorAdapter(
        JpaProperties jpaProperties) {
        AbstractJpaVendorAdapter adapter = new HibernateJpaVendorAdapter();
        adapter.setShowSql(jpaProperties.isShowSql());
        adapter.setGenerateDdl(jpaProperties.isGenerateDdl());
        return adapter;
    }
}
```
#### 定义AOP拦截切换不同数据源
```java
@Aspect
@Component
public class DataSourceDetermineAspect {

  @Before("@annotation(applyDataSource)")
  public void before(ApplyDataSource applyDataSource) {
    DatabaseContextHolder.set(applyDataSource.value());
  }

  @After("@annotation(applyDataSource)")
  public void after(ApplyDataSource applyDataSource) {
    DatabaseContextHolder.clear();
  }

}

```