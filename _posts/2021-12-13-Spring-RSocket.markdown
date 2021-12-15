---
layout: post
title:  "Spring RSocket & Retrosocket"
date:   2021-12-13 16:00:00 +0900
categories: Network
---
created : 2021-12-13, updated : 2021-12-13

# Introduction
RSocket은 REST API, gRPC 이후에 나온 Protocol로 기존의 단점들을 보강하기 위해 Netflix에서 개발을 시작하였다. 계속 개발이 이루어지고 있는 상태고 Spring에서도 관련 개념이 정리되고 있다. 기존의 통신과 차이점은 OSI Layer 5,6 이라고 한다. 앞으로 주목해야할 Protocol이라고 생각된다.

Spring RSocket은 GA가 되었고 관련해서 편하게 쓰기 위한 Retroscoket은 아직 GA가 아니다. 본 튜토리얼에서는 Spring RSocket의 동작방법과 Retrosocket의 맛배기로 보면 될 것 같다. 


# 라이브러리
Spring Retrosocket 을 사용하기 위해 SNAPSHOT으로 프로젝트를 생성하였다. 

- RSocket: RSocket의 기능을 사용
- Lombok: Entity 구현


# hello 메시지 구현하기
hello와 hello.{name}에 대한 서비스를 만들어 보자. 메세지를 호출하고 간단하게 답을 받는 방식이다. (Request and response)

## Service 구현하기 
먼저 설정에 port를 추가해보자. 8181로 rsocket 을 사용하도록 설정을 했다.

파일: application.properties
```
spring.rsocket.server.port=8181
```

간단한 hello 메시지에 대해 “Hello World!”를 출력하게 해보자. @Controller를 선언한 class를 만들고 간단하게 “Hello World!”를 전달하도록 구현을 해보았다.
hello.{name} 메시지는 인자{name} 을 받아서 Hello + name을 출력하도록 하였다. 

파일: ServiceApplication.java
```
@Controller
class GreetingController {

	@MessageMapping("hello")
	Mono<String> hello() {
		return Mono.just("Hello World!");
	}

	@MessageMapping("hello.{name}")
	Mono<String> helloName(@DestinationVariable String name) {
		return Mono.just("Hello "+ name + "!");
	}
}
```

## CLI로 테스트하기
[rsc cli](https://github.com/making/rsc) 를 이용하면 cli로 간단하게 테스트를 할 수 있다. (설치는 링크의 문서를 참조하여 설치하기 바란다.)

```
$ rsc tcp://localhost:8181 --route hello

Hello World!

$ rsc tcp://localhost:8181 --route hello.Gildong

Hello Gildong!
```

## Client를 만들어보자.
service가 잘동작하는지를 보았으니 client 를 만들어 보자. client 프로젝트에 서비스와 통신하는 부분을 설정하고 간단하게 통신하는 부분을 만들어보자.

service와 연결을 해보자. main안에 bean으로 만들었다. 특정한 이벤트를 발생하지 않고 바로 실행되게 하였다.
서버의 port 8181을 선언하여 RSocketRequester를 만들었다.

파일: ClientApplication.java
```
@Bean
RSocketRequester rSocketRequester(RSocketRequester.Builder builder) {
	return builder.tcp("localhost", 8181);
}
```

service에 RSocketRequester를 이용하여 메시지를 호출을 한다.  hello는 인자 없이 호출하였고 결과값은 화면을 출력(System.out::println)하도록 하였다. 
```
@Bean
ApplicationRunner basic(RSocketRequester rSocketRequester) {
	return event -> {
		Mono<String> reply = rSocketRequester.route("hello").retrieveMono(String.class);
		reply.subscribe(System.out::println);
	};
}
```

hello.name으로도 호출을 하도록 하였다. 
```
@Bean
ApplicationRunner basicName(RSocketRequester rSocketRequester) {
	return event -> {
		Mono<String> reply = rSocketRequester.route("hello.Gildong").retrieveMono(String.class);
		reply.subscribe(System.out::println);
	};
}

```

실행을 하면 아래와 같이 결과가 표시되는 것을 볼수가 있다. 
```
Hello World!
Hello Gildong!
```

> Note: ClientApplcation을 계속 구동중인 상태로 만들기 위해 main에 System.read.in()을 추가하였다. Bean 으로만 구현을 하면 프로그램이 바로 종료되기 때문에 이렇게 추가가 된 부분이다. 실제 프로젝트에서는 Application이 구동중인 상태 일테니 이 부분은 생략해도 될 것이다.


# Spring Retrosocket 이용하기
위에서는 간단하게 RSocket만으로 Client를 만들어 보았다. 이번에는 아직은 GA가 아니지만 좀더 간단하게 명시적으로 사용할 수 있다. 

## 라이브러리 추가하기
먼저 라이브러리에 spring-retrosocket을 추가해보자. 
파일: pom.xml
```
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-rsocket</artifactId>
	</dependency>
```

> Note: SNAPSHOT Repository가 선언되어 있어야 한다. 맨처음에 프로젝트를 SNAPSHOT을 이용하여 만든 이유이다.

## Client 추가하기
Service에는 추가할 부분은 없고 Client에 @EnableRSocketClients으로 RSocketClient를 활성화 한다. 

파일: ClientApplication.java
```
@EnableRSocketClients
@SpringBootApplication
public class ClientApplication {
…
```

이제 RSocketClient interface를 선언해줘야한다. 메시지에 대해 선언하는 부분이다. 두개의 method를 선언하였고 하나는 Argument를 받도록 정의되었다. 

```
@RSocketClient
interface GreetingClient {
	@MessageMapping("hello")
	Mono<String> greet();

	@MessageMapping("hello.{name}")
	Mono<String> greetName(@DestinationVariable String name);
}
```

새로운 bean으로 두개의 메시지를 RSocketClient를 이용하여 구현을 해보자. 
기존의 route, return되는 부분의 정의를 하지 않아도 되서 상당히 직관적이 된것을 볼수 있다. 또한 재사용이 가능하여 생산성이 높아질 것으로 생각된다.

```
@Bean
ApplicationRunner retroClient(GreetingClient gc) {
	return event -> {
			Mono<String> reply = gc.greet();
			reply.subscribe(System.out::println);
	};
}

@Bean
ApplicationRunner retroClientName(GreetingClient gc) {
	return event -> {
		Mono<String> reply = gc.greetName("Spring Fans");
		reply.subscribe(System.out::println);
	};
}
```

## 테스트 하기  
client를 실행을 하면 아래와 같이 결과를 확인할 수 있다. 처음 2개의 메세지는 RSocket으로 만 만든것이고 뒤에 2개는 Retrosocket을 이용하여 만든 것이다. 

```
Hello World!
Hello Gildong!
Hello World!
Hello Spring Fans!
```

> Note: GIT에 공유된 소스에는 추가로 GreetingRequest, GreetingResponse를 이용하여 Retrosocket을 이용한 데이터 통신 부분도 추가하였다. 사용하는 방식은 동일한 소스에서 참조하기 바란다. 


# Conclusions
RSocket은 gRPC의 대안이 될 것 같다. 계속 기능이 개발중이고 빠르게 발전하고 있는 분야이다. Network이 빨라지고 데이터 양이 많아지면서 좀더 Low latency, High Performance의 요구사항이 늘고 있다. RSocket이 그 대안이 될 것 같다고 생각이 된다.


# References
[Spring Tips - Spring RSocket](https://www.youtube.com/watch?v=d4HAqS_VfkQ)

[Spring Tips - Spring Retrosocket, an easy-to-use, proxy-powered client for RSocket](https://www.youtube.com/watch?v=vHOztWJNnBs)

[RSocket](https://rsocket.io/)

[RSocket CLI](​​https://github.com/making/rsc)

[Spring Retrorsockt](https://spring-projects-experimental.github.io/spring-retrosocket/)

