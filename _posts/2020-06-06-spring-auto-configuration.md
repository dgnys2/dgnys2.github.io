---
layout: post
title: Spring AutoConfiguration(md)
date: 2020-06-06 00:00:00 +0300
description: # Add post description (optional)
img: software.jpg # Add image post (optional)
tags: [SpringBoot,자동완성] # add tag
---


Spring AutoConfiguration
=============


Spring Boot Framework 를 이용해 개발을 하다보면,
Auto Configuration 라는 말을 많이 접하게 된다.
일반 Spring 과 Spring Boot 와의 큰 차이점 중 하나 이기도 한데,
간단히 말하면, Spring Boot App 을 구동 시, 개발자가 Bean 으로 직접 등록하지 않더라도 자동으로 연관 필요한 Bean 들을 등록시켜 주는 기능이다.

Spring Boot 구동 시, 어떤 식으로 동작하는지 한번 알아보자.



### 1. @EnableAutoConfiguration
----

AutoConfiguration 기능을 사용하려면, @EnableAutoConfiguration 이라는 Annotation 을 붙이면 된다. 하지만 별도로 위 Annotation 을 명시한적이 없더라도, 자동완성(AutoConfiguration) 기능을 사용하고 있었을 텐데, 이는 아래와 같이 @SpringBootConfiguration Annotation 안에 이미 포함되어 있기 때문이다. 


```
@SpringBootApplication
public class DemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}

}
```

위 처럼, Spring boot 앱의 main 함수에 @SpringBootApplciation 이 붙혀져있고,
이 @SpringBootApplication 어노테이션을 들어가보면, 아래와 같이 
@EnableAutoConfiguration 이 포함되어있음을 알 수 있다.

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    ...
}
```

  

## 2. 어떤 Bean 들을 등록할지 어떻게 아는가?
----


Spring Boot App 구동 시, Bean 들을 스캔하는데 아래와  같은 순서로 Bean 을 Scan 하여 등록한다.

1) @ComponentScan 을 통한 Beans 스캔 & 등록
2) AutoConfiguration 기능을 통한 Bean 등록

먼저, 1차적으로 1)번의 @ComponentScan 을 통해 스프링 앱 내 모든 Bean 들을 스캔하고 등록한다. @ComponentScan의 대상이 되는 것들은 @Component 어노테이션이 붙어있거나,
혹은 해당 Annotation 을 포함하는 Annotation이 붙어있는 모든 클래스들이다.

예시)
@Component, @Configuration @Repository @Service @ Controller @RestController


이 때, @ComponentScan 을 포함한 하위 패키지에 대해서만 진행하고, 상위 패키지에 대해서는
진행하지 않으니 주의하자.


그리고 나서, 2)번의 AutoConfiguration 기능을 통해 필요한 Bean 들을 등록한다.
이 때, dependencies 에 포함된 모든 .jar 파일들의 META-INF 디렉토리 안 spring.factories 라는
파일들에서 Bean 스캔을 할 대상을 가져와, 스캔을 하여 등록을 한다.

자세히 살펴보면, spring.factories 파일 내에 특정 key값(org.springframework.boot.autoconfigure.EnableAutoConfiguration) 에 등록된 Configuration Classes 들을 가져와 스캔을 하고, 조건에 맞는 Bean 들을 등록시킨다. 

간단히 샘플을 하나 살펴보면, spring-boot-autoconfigure-버전.RELEASE.jar (예, spring-boot-autoconfigure-2.2.5.RELEASE.jar) 안에 META-INF 라는 경로가 있고,
해당 경로안에 spring.factories 파일이 있다.

이 파일을 살펴보면 아래와 같이 org.springframework.boot.autoconfigure.EnableAutoConfiguration 에 매칭되는 configuration class 들이 선언되어있고,
해당 configuration class 들을 스캔하면서 조건에 맞는 Bean 들을 등록시킨다.

이처럼 모든 .jar 에서 spring.factories 를 찾아 그 안에 있는 이 값(org.springframework.boot.autoconfigure.EnableAutoConfiguration ) 에 해당하는 모든 클래스들에 대해 진행하고,
컨디션에 맞으면 Bean 으로 등록시킨다.

```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.cloud.CloudServiceConnectorsAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,
...
```

   
      
	     

## 3. 어떤 Bean 을 등록시킬지 어떻게 결정하는가?
---
spring.facotires 내 org.springframework.boot.autoconfigure.EnableAutoConfiguration 에 매칭되는 configuration class 들을 대상으로 Bean 을 등록시킬때, 조건에 따라 Bean 들을 등록시킨다고 하였다. 그럼 그 조건들은 과연 무엇일까?

Configration class 를 살펴보면, 아래와 같은 
@Conditional...으로 시작하는 Annotation 들을 볼수있다.

예)
@ConditionalOnMissingBean
@ConditionalOnClass..

예시로, WebMvcAutoconfiguration 을 살펴보면 아래와 같이 @ConditionalOnMissingBean(HiddenHttpMethodFilter.class) 처럼 달려있는 Annotation 을 볼 수 있는데, 이게 바로 조건을 나타내는 Annotation 이다.

```
...
	@Bean
	@ConditionalOnMissingBean(HiddenHttpMethodFilter.class)
	@ConditionalOnProperty(prefix = "spring.mvc.hiddenmethod.filter", name = "enabled", matchIfMissing = false)
	public OrderedHiddenHttpMethodFilter hiddenHttpMethodFilter() {
		return new OrderedHiddenHttpMethodFilter();
	}

	@Bean
	@ConditionalOnMissingBean(FormContentFilter.class)
	@ConditionalOnProperty(prefix = "spring.mvc.formcontent.filter", name = "enabled", matchIfMissing = true)
	public OrderedFormContentFilter formContentFilter() {
		return new OrderedFormContentFilter();
	}

	static String[] getResourceLocations(String[] staticLocations) {
		String[] locations = new String[staticLocations.length + SERVLET_LOCATIONS.length];
		System.arraycopy(staticLocations, 0, locations, 0, staticLocations.length);
		System.arraycopy(SERVLET_LOCATIONS, 0, locations, staticLocations.length, SERVLET_LOCATIONS.length);
		return locations;
	}
    ...
```    

 
@ConditionalOnMissingBean(HiddenHttpMethodFilter.class) 의 의미는
    HiddenHttpMethodFilter 타입의 빈이 등록되어 있지 않으면, 다음 Bean 을 등록시켜라는 의미로, 앞 서, Auto Configuration 의 앞 단계에서, Component Scan 에 의해 등록된 Bean 들의 목록을 훑으면서 해당 조건들에 부합하는 Bean 들을 자동으로 모두 등록시킨다.

