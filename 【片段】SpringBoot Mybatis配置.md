---
title: '[片段] SpringBoot Mybatis配置'
date: '2019-03-13 10:00:00'
tags:
    - spring
    - java
    - mybatis
    - 代码
categories:
    - Java框架
    - Mybatis
---

纯记录，供自己参考🤣。


```java
private final MybatisProperties properties;

private final Interceptor[] interceptors;

private final ResourceLoader resourceLoader;

private final DatabaseIdProvider databaseIdProvider;

private final List<ConfigurationCustomizer> configurationCustomizers;

public DataSourceConfig(MybatisProperties properties,
                        ObjectProvider<Interceptor[]> interceptorsProvider,
                        ResourceLoader resourceLoader,
                        ObjectProvider<DatabaseIdProvider> databaseIdProvider,
                        ObjectProvider<List<ConfigurationCustomizer>> configurationCustomizersProvider) {
    this.properties = properties;
    this.interceptors = interceptorsProvider.getIfAvailable();
    this.resourceLoader = resourceLoader;
    this.databaseIdProvider = databaseIdProvider.getIfAvailable();
    this.configurationCustomizers = configurationCustomizersProvider.getIfAvailable();
}


/**
 * 普通数据源
 * 主数据源，必须配置，spring启动时会执行初始化数据操作（无论是否真的需要），选择查找DataSource class类型的数据源
 *
 * @return {@link DataSource}
 */
@Primary
@Bean(name = BEANNAME_DATASOURCE_COMMON)
@ConfigurationProperties(prefix = "com.lianjia.confucius.bridge.boot.datasource.common")
public DataSource createDataSourceCommon() {
    return DataSourceBuilder.create().build();
}

/**
 * 只读数据源
 *
 * @return {@link DataSource}
 */
@Bean(name = BEANNAME_DATASOURCE_READONLY)
@ConfigurationProperties(prefix = "com.lianjia.confucius.bridge.boot.datasource.readonly")
public DataSource createDataSourceReadonly() {
    return DataSourceBuilder.create().build();
}

private SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
    SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
    factory.setDataSource(dataSource);
    factory.setVfs(SpringBootVFS.class);
    if (StringUtils.hasText(this.properties.getConfigLocation())) {
        factory.setConfigLocation(this.resourceLoader.getResource(this.properties.getConfigLocation()));
    }
    org.apache.ibatis.session.Configuration configuration = this.properties.getConfiguration();
    if (configuration == null && !StringUtils.hasText(this.properties.getConfigLocation())) {
        configuration = new org.apache.ibatis.session.Configuration();
    }
    if (configuration != null && !CollectionUtils.isEmpty(this.configurationCustomizers)) {
        for (ConfigurationCustomizer customizer : this.configurationCustomizers) {
            customizer.customize(configuration);
        }
    }
    factory.setConfiguration(configuration);
    if (this.properties.getConfigurationProperties() != null) {
        factory.setConfigurationProperties(this.properties.getConfigurationProperties());
    }
    if (!ObjectUtils.isEmpty(this.interceptors)) {
        factory.setPlugins(this.interceptors);
    }
    if (this.databaseIdProvider != null) {
        factory.setDatabaseIdProvider(this.databaseIdProvider);
    }
    if (StringUtils.hasLength(this.properties.getTypeAliasesPackage())) {
        factory.setTypeAliasesPackage(this.properties.getTypeAliasesPackage());
    }
    if (StringUtils.hasLength(this.properties.getTypeHandlersPackage())) {
        factory.setTypeHandlersPackage(this.properties.getTypeHandlersPackage());
    }
    if (!ObjectUtils.isEmpty(this.properties.resolveMapperLocations())) {
        factory.setMapperLocations(this.properties.resolveMapperLocations());
    }

    return factory.getObject();
}

public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
    ExecutorType executorType = this.properties.getExecutorType();
    if (executorType != null) {
        return new SqlSessionTemplate(sqlSessionFactory, executorType);
    } else {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}

@Bean
@Primary
public SqlSessionFactory primarySqlSessionFactory() throws Exception {
    return this.sqlSessionFactory(this.createDataSourceCommon());
}

@Bean
public SqlSessionFactory secondarySqlSessionFactory() throws Exception {
    return this.sqlSessionFactory(this.createDataSourceReadonly());
}

/**
 * 实例普通的 sqlSession
 *
 * @return SqlSession
 * @throws Exception when any exception occured
 */
@Bean(name = BEANNAME_SQLSESSION_COMMON)
public SqlSession initSqlSessionCommon() throws Exception {
    return this.sqlSessionTemplate(this.primarySqlSessionFactory());
}

/**
 * 实例只读的 sqlSession
 *
 * @return SqlSession
 * @throws Exception when any exception occured
 */
@Bean(name = BEANNAME_SQLSESSION_READONLY)
public SqlSession initSqlSessionReadonly() throws Exception {
    return this.sqlSessionTemplate(this.secondarySqlSessionFactory());
}


@MapperScan(annotationClass = PrimaryMapper.class,
        sqlSessionTemplateRef = BEANNAME_SQLSESSION_COMMON,
        basePackageClasses = ITalentApplicationSpringBootStart.class)
static class PrimaryMapperConfiguration {
}

@MapperScan(annotationClass = SecondaryMapper.class,
        sqlSessionTemplateRef = BEANNAME_SQLSESSION_READONLY,
        basePackageClasses = ITalentApplicationSpringBootStart.class)
static class SecondaryMapperConfiguration {
}
```