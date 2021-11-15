---
layout: post
title:  "Spring Cloud Circuit Breaker"
date:   2021-10-07 16:00:00 +0900
categories: CloudNative
---
created : 2021-10-07, updated : 2021-10-07

# Introduction
서비스간 데이터를 요청할때 한쪽 서비스에 문제가 발생하였을 경우가 있다. 이럴 때를 대비하여 정보는 Default로 전달하여 전체 서비스가 오류가 나는 것을 방지 할 수 있다. 이때를 위해 Circuit Breaker를 이용하여 Default정보를 보낼 수 있다.

> Note. 본 Tutorial은 [“Client Side Load Balancer - Ribbon (Spring Cloud)”](https://ahnchan.github.io/posts/CloudNative-Client-Side-Load-Balancer-Ribbon/)를 기반으로 Circuit Breaker 부분을 추가로 구현하였다. 

# Requirements
[“Service Registration and Discovery (Spring Boot)”](https://ahnchan.github.io/posts/CloudNative-Service-Registration-and-Discovery/)

[“Client Side Load Balancer - Ribbon (Spring Cloud)”](https://ahnchan.github.io/posts/CloudNative-Client-Side-Load-Balancer-Ribbon/)


> Note. 이 튜토리얼의 소스는 [이곳](https://github.com/ahnchan/tutorial-spring-cloud-circuit-breaker)에서 확인할 수 있다. initial은 이전 튜토리얼에서의 구성한 소스이고 complate는 본 튜토리얼에 추가한 부분이 포함되어 있다. 

# 구성
Review 서비스가 장애가 발생하는 것을 가정할 것이다. 구성도에서 보듯이 서비스가 임의로 종료하였을때 Details 서비스에서 Reviews 서비스를 호출하면 장애가 발생할 것이다. 이때 Spring Cloud Circuit Breaker를 사용하도록 할 것이다.

![Diagram](/posts/assets/cloudnative/images/circuitbreaker-diagram.png){: width="500"} 

> Note. Production 환경에서는 상황에따라 다양하게 오류를 대처가 필요하다. 이 튜토리얼에서는 오류가나는 API에 대해서 Default 정보를 보내서 회피하는 것을 학습할 것이다. 

# 장애 상황을 만들어보기
먼저 장애상황을 만들어보고 이를 해결해보기로 하자. 전체 시스템을 먼저 모두 시작하고 장애 상태를 만들어 보겠다. . 

> Note. 시작시 오류가 나면 이전 튜토리얼 ([링크](https://ahnchan.github.io/posts/CloudNative-Client-Side-Load-Balancer-Ribbon/])을 참조하기 바란다.

## Discovery 시작하기
```
$ cd discovery
$ mvn spring-boot:run
...
```

## Details 시작하기
```
$ cd details
$ maven spring-boot:run
…
```

## Reviews 시작하기
```
$ cd reviews
$ mvn spring-boot:run
…
```

브라우저에서 [http://localhost:8671](http://localhost:8671) 을 호출하면 Eureka Server에 DETAILS, REVIEWS가 접속되어 있는 것을 확인할 수 있다.

Reviews 서비스가 오류가 나는 상황을 만들어 보자. Reviews 서비스를 Ctrl+c로 종료를 한다. Reviews 서비스가 종료가되면 /products/1/detailsV1 이나 products/1/detailsV2 를 curl을 이용하여 호출해보자. 아래와 같이 서버 오류가 나올 것이다.

```
$ curl localhost:8080/products/1/detailsV1 | jq
{
  "timestamp": "2021-10-07T06:34:28.367+00:00",
  "status": 500,
  "error": "Internal Server Error",
  "path": "/products/1/detailsV1"
}
```

API의 결과 값이 정의된 스펙이 아닌 Internal Server 오류로 넘어온다. 이제 Reviews 서비스가 장애로 해당 API가 오류가 발생되는 경우를 Circuit Breaker를 이용하여 회피해보자. 


# OpenFeign 환경에서 구현하기
이 기능을 사용하기 위해서는 먼저 설정에서 Circuit Breaker를 true로 변경해줘야 한다. properties에 아래의 내용을 추가해주자.

파일: application.properties
```
feign.circuitbreaker.enabled=true
```

Details에 정의된 /products/{id}/detailsV2 부분이다. 이 API에서는 Interface를 구현했었다. 이 Interface의 정의 부분에 Fallback시의 ReviewClientFallback.class를 선언추가 한다.

파일 : DetailsApplication.java
```
...
@FeignClient(name="REVIEWS", fallback=ReviewClientFallback.class)
interface ReviewClient {
	@RequestMapping(method = RequestMethod.GET, value="/products/{id_product}/reviews")
	List<Review> getReviews(@PathVariable Integer id_product);
}
```

ReviewClientFallback.class를 구현해보자. 장애 대처상황에 따라 다르겠지만 이번에는 장애와 데이터가 없는 것을 구분하기 위해서 빈 List가 아닌 -1인 Review 정보를 만들어 봤다. 

파일 : DetailsApplications.java
```
...
@Component
class ReviewClientFallback implements ReviewClient {
	@Override
	public List<Review> getReviews(Integer id_product) {
		List results = new ArrayList<Review>();
		Review fallbackReview = new Review(-1, -1, "-", "-");
		results.add(fallbackReview);

		return results;
	}
}
```

이제 Details 서비스를 구동을 하고 API 호출해보자. 현재 상태는 Reviews 서비스가 종료되어 있는 상황이다. 위에서 장애상황과 어떻게 다른지 확인 할 수 있다. 

```
$ mvn spring-boot:run
…
```
```
$ curl localhost:8080/products/1/detailsV2 | jq
{
  "product": {
    "id": 1,
    "title": "The Keeper of Happy Endings",
    "author": "Barbara Davis",
    "isbn": "1542021472"
  },
  "reviews": [
    {
      "id": -1,
      "id_product": -1,
      "reviewer": "-",
      "text": "-"
    }
  ]
}
```

500 Internal Server 오류가 아니라 구현한정보에 따라 Review id가 -1인 정보로 전달했다. 


# RestTemplate 호출시 구현하기
/product/{id}/detailV1 API는 RestTemplate을 이용한 것이다. 그럼 이번에는 RestTemplate을 사용하는 부분에서 Circuit Breaker를 구현해보자. 

먼저 라이브러리를 추가해야한다. 

파일: pom.xml
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

이제 Circuit Breaker를 정의하고, Reviews API를 호출하는 부분에 추가를 해보자. 여기서는 기존 API (http://revies/products…) 를 호출하였을때 오류가 발생하면 getDefaultRevies()에서 정보를 가져오게 하였다. 

파일 : DetailsApplication.java
```
...
@Autowired
private CircuitBreakerFactory circuitBreakerFactory;

@GetMapping("/products/{id}/detailsV1")
public ProductDetail getProductDetails(@PathVariable Integer id) {

	CircuitBreaker circuitBreaker = circuitBreakerFactory.create("circuitbreaker");

	Optional<Product> product = repository.findById(id);
	ProductDetail details = new ProductDetail();
	details.setProduct(repository.findById(id).get());
	details.setReviews(circuitBreaker.run(()->
			restTemplate.getForObject("http://reviews/products/"+id+"/reviews", List.class),
			throwable -> getDefaultReviews());

	return details;
}

protected List<Review> getDefaultReviews() {
	List results = new ArrayList<Review>();
	Review fallbackReview = new Review(-1, -1, "-", "-");
	results.add(fallbackReview);

	return results;
}
```

구현을 했으니 이제 Details를 구동하고 API를 호출해보자.

```
$mvn spring-boot:run
…
```

```
$ curl localhost:8080/products/1/detailsV1 | jq
{
  "product": {
    "id": 1,
    "title": "The Keeper of Happy Endings",
    "author": "Barbara Davis",
    "isbn": "1542021472"
  },
  "reviews": [
    {
      "id": -1,
      "id_product": -1,
      "reviewer": "-",
      "text": "-"
    }
  ]
}
```

오류가 아닌 getDefaultReviews()에서 정의한 Review id가 -1 인 정보를 보내줬다.


# Conclusions
API의 장애를 회피하는 방법 중에 서비스단에서 처리할 수 있는 방법을 구현해봤다. 장애를 대처하기 위해서는 많은 방법이 있을 것이다. Circuit Breaker를 이용하여 간단하게 Default 정보를 보내는 것을 확인하였다. CIrcuit Breaker은 Retry, Forward 등 다양하게 처리할 수 있다. 다른 기능은 시간이되면 다른 튜토리얼에서 이야기를 해보겠다.


# References
[Spring Cloud OpenFeign - 1.5. Feign Spring Cloud CircuitBreaker Support](https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/#spring-cloud-feign-circuitbreaker)

[Quick Guide to Spring Cloud Circuit Breaker](https://www.baeldung.com/spring-cloud-circuit-breaker)



