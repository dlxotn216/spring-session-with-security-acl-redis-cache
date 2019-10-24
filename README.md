# Spring- ession with security ACL Redis Cache
Spring Session과 Spring Security ACL (Redis Cache) 설정을 진행할 때 발생하는 이슈 등 정리

## Cloud 환경 이관

Legacy 프로젝트를 Cloud에 이관하기 위해 약간의 구조 변경이 필요했다.  
이중화를 위해 JBoss Cluster를 사용하고 있었으나 Spring Session으로의 전환이 필요했고  
Spring Security ACL이 사용하는 권한 캐시의 동기화를 위해서 JBoss의 Infinispan cache container 대신 Redis로 전환이 필요했다.  

Spring Security ACL의 캐시 구현체 전환은 이전에 진행한 내용이 있어 쉽게 적용이 가능했다. 
<a href="https://github.com/dlxotn216/spring-security-acl-with-redis-cache">Spring Security ACL with Redis Cache</a>

위의 내용 진행 후 아래와 같이 추가적인 작업을 통해 Spring Sesion 설정을 진행했다.  
(Spring session의 Session Repository도 Redis로 진행한다)  

pom.xml
```xml
 <dependency>
      <groupId>org.springframework.session</groupId>
      <artifactId>spring-session-data-redis</artifactId>
      <version>1.3.5.RELEASE</version>
  </dependency>
```

Application Context
```xml
<bean class="org.springframework.session.data.redis.config.annotation.web.http.RedisHttpSessionConfiguration"/>
```

web.xml
```xml
<filter>
    <filter-name>springSecurityFilterChain</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>

<filter-mapping>
    <filter-name>springSecurityFilterChain</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

그 후 JBoss를 가동시키고 테스트 중에 문제가 발생했다.  
권한 캐시의 특성 상 권한의 내용이 바뀌면 캐시를 모두 비우도록 설정되어있는데  
캐시를 모두 비우는 경우 세션이 모두 날아가 모든 사용자가 로그아웃 되는 현상이 발생한 것이다.  

따라서 Spring Session이 사용할 Redis와 Spring Security ACL이 사용할 Redis를 분리하는 작업이 필요했다.  


## Docker를 이용한 두 개의 Redis 설정
먼저 아래와 같이 두 개의 Redis Container를 띄운다

- docker run -d -p 6379:6379 redis
- docker run -d -p 7777:7777 redis --port 7777

여기서 -p {in:out)은 host의 in port를 container의 out port로 연결하는 옵션이다.  
-d는 Daemon으로 띄우는 설정이다.  

두번째 컨테이너는 별개의 서비스로 동작해야 하므로 독립된 포트인 7777포트로 매핑하였고 redis 인스턴스 자체도 컨테이너 안에서 7777로 동작하게 했다.  

## 각 모듈 별 Redis 설정
아래와 같이 Spring session이 사용할 것, Security ACL이 사용할 것 각각 Lettuce connection factory를 분리했다.  
```xml
<!-- Spring session 전용 Redis config -->
<bean id="lettuceConnectionFactory" class="org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory"
          p:hostName="localhost" p:port="6379" p:database="0" p:password=""/>

<bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
    <property name="connectionFactory" ref="lettuceConnectionFactory" />
</bean>
<bean class="org.springframework.session.data.redis.config.annotation.web.http.RedisHttpSessionConfiguration"/>

<!-- Spring Security ACL 전용 Redis config -->
<bean id="cacheLettuceConnectionFactory" class="org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory"
      p:hostName="localhost" p:port="7777" p:database="0" p:password=""/>

<bean id="cacheRedisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
    <property name="connectionFactory" ref="cacheLettuceConnectionFactory" />
</bean>
```

그 후 아래와 같이 기존에 Spring Security ACL이 사용하는 cacheManager가 cacheRedisTemplate을 사용하도록 변경했다.  
```xml
<cache:annotation-driven/>

<jee:jndi-lookup id="nativeCacheManager"
                 expected-type="org.infinispan.manager.EmbeddedCacheManager"
                 jndi-name="java:#{stageProperties['jndi.nativeCacheManager']}"/>

<bean id="cacheManager" class="kr.co.crscube.ctms.security.acl.AsyncFlushableCacheManager">
    <constructor-arg ref="cacheRedisTemplate" />
</bean>
```

JBoss를 띄워보면 아래와 같은 메시지를 띄우고 뻗어버린다.  
14:50:49,127 JBWEB000287: Exception sending context initialized event to listener instance of class org.springframework.web.context.ContextLoaderListener  
14:50:49,164 JBWEB001103: Error detected during context  start, will stop it  
14:50:49,182 Closing Spring root WebApplicationContext  
14:50:49,186 MSC000001: Failed to start service jboss.web.deployment.default-host./  
...  
[2019-10-24 02:50:49,290] Artifact ctms:war exploded: Error during artifact deployment. See server log for details.  

정확한 판단을 위해 server.log를 확인하였고 아래와 같은 로그를 확인할 수 있었다.  

Caused by: org.springframework.beans.factory.NoUniqueBeanDefinitionException: No qualifying bean of type [org.springframework.data.redis.connection.RedisConnectionFactory]  
is defined: expected single matching bean but found 2: lettuceConnectionFactory,cacheLettuceConnectionFactory  
at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:970) [spring-beans-4.0.4.RELEASE.jar:4.0.4.RELEASE]  

RedisConnectionFactory 타입의 Bean이 두개가 있어 qualifying이 필요한 상황이었다.  

LettuceConnectionFactory는 어쩔 수 업이 두개가 필요하므로 qualifying을 위해서는 Bean name을 알맞게 설정해주어야 할 것인데  
현재 필요한 것은 어떤 Bean이 RedisConnectionFactory에 대한 의존성 주입을 필요로 하는 것이다. 그래야 Bean name을 설정해주어 qualifying 할수 있을테니...  

로그를 조금 더 찾아보니 아래와 같은 것을 찾을 수 있었다.  
14:50:49,117 ERROR [org.springframework.web.context.ContextLoader] (ServerService Thread Pool -- 5) Context initialization failed:  
org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'redisMessageListenerContainer'  
defined in class path resource [org/springframework/session/data/redis/config/annotation/web/http/RedisHttpSessionConfiguration.class]:  
Unsatisfied dependency expressed through constructor argument with index 0 of type   
[org.springframework.data.redis.connection.RedisConnectionFactory]  

조금 전 Spring Session의 설정을 진행하면서 등록 한 RedisHttpSessionConfiguration Bean 안에서 생성되는 redisMessageListenerContainer Bean의 생성자 0번째 타입으로  
RedisConnectionFactory를 주입 받도록 되어있으며, 현재 설정에선 RedisConnectionFactory가 두 개 발견되어 어떤 것을 주입해주어야 하는지  
Spring Contect Loader가 제대로 알 수 없는 상황이 원인임을 파악할 수 있엇다.  

해당 Bean의 설정 코드를 보면 아래와 같다.   
```java
@Bean
public RedisMessageListenerContainer redisMessageListenerContainer(
    RedisConnectionFactory connectionFactory,
    RedisOperationsSessionRepository messageListener) {

  RedisMessageListenerContainer container = new RedisMessageListenerContainer();
  container.setConnectionFactory(connectionFactory);
  if (this.redisTaskExecutor != null) {
    container.setTaskExecutor(this.redisTaskExecutor);
  }
  if (this.redisSubscriptionExecutor != null) {
    container.setSubscriptionExecutor(this.redisSubscriptionExecutor);
  }
  container.addMessageListener(messageListener,
      Arrays.asList(new PatternTopic("__keyevent@*:del"),
          new PatternTopic("__keyevent@*:expired")));
  container.addMessageListener(messageListener, Arrays.asList(new PatternTopic(
      messageListener.getSessionCreatedChannelPrefix() + "*")));
  return container;
}
```

0번째 타입으로 RedisConnectionFactory를 주입 받으며 스프링의 의존성 주입 원칙에 따라 동일 타입이 있는 경우   
Qualifying 할 수 있는 Bean의 이름은 파라미터 이름, 변수명이다.  
따라서 Spring session이 사용할 LettuceConnectionFactory Bean의 이름은 connectionFactory로 지정해주면 될 것이다.  

최종적으로 아래와 같이 설정파일을 변경한다.
```xml
<context:annotation-config />

<!-- RedisHttpSessionConfiguration에서 connectionFactory 이름으로 주입 받음 -->
<!-- Spring session 전용 Redis config -->
<bean id="connectionFactory" class="org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory"
      p:hostName="localhost" p:port="6379" p:database="0" p:password=""/>

<bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
    <property name="connectionFactory" ref="connectionFactory" />
</bean>
<bean class="org.springframework.session.data.redis.config.annotation.web.http.RedisHttpSessionConfiguration"/>

<!-- Spring Security ACL 전용 Redis config -->
<bean id="cacheLettuceConnectionFactory" class="org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory"
      p:hostName="localhost" p:port="7777" p:database="0" p:password=""/>

<bean id="cacheRedisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
    <property name="connectionFactory" ref="cacheLettuceConnectionFactory" />
</bean>
```

아래와 같이 정상적으로 서비스가 시작 된 것을 볼 수 있다.  
[2019-10-24 03:02:43,051] Artifact ctms:war exploded: Deploy took 9,364 milliseconds



