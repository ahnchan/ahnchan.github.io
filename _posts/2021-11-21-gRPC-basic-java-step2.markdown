---
layout: post
title:  "gRPC - Greeting Service 만들기"
date:   2021-11-21 16:00:00 +0900
categories: Network
---
created : 2021-11-21, updated : 2021-11-21

# Introduction
[이전 튜토리얼](https://github.com/ahnchan/tutorial-gRPC)에서는 gRPC의 기본 개념과 Protocol Buffer와 Dummy Server/Client를 간단하게 만들어 보았다. 이제는 한단계 더 들어가서 데이터를 통신하는 방식을 알아보겠다. 


# Pre-Installation
IntelliJ Community
Java SDK
[gRPC - Basic 튜토리얼](https://ahnchan.github.io/posts/gRPC-basic-java-step1/)

> Note. 소스코드는 [https://github.com/ahnchan/tutorial-gRPC](https://github.com/ahnchan/tutorial-gRPC)에서 확인할 수 있다.


# 구성
4가지 데이터 교환 방식을 만들어 볼것이다. [gRPC 의 공식문서](https://grpc.io/docs/what-is-grpc/core-concepts/#rpc-life-cycle)를 보면 RPC Life cycle 부분을 보면 나와있다. 

## Unary 
간단한 구성으로 하나의 Request에 하나의 Response로 통신을 한다. 일반적인 REST 방식과 같은 형식이다. 지금 가장 많이 사용되는 방식이다.  

## Server streaming
Unary와 같이 Client에서 Request를 하면, Server는 Stream으로 데이터를 계속 전송을 한다. 그리고 Server에서 메시지 전송이 완료되면, 완료되었다는 정보를 Client에 Respnse로 보내면 하나의 Cycle이 완료된다. 
Server에서 대량의 데이터를 나누어서 전송할 때 사용할 수 있겠다. 

## Client streaming
Server streaming과는 반대로 Client에서 Streming으로 정보를 계속 전송을 하고 Client에서 완료를 하면, Server에서는 Response 를 한번 보내는 방식이다. 

Client에서 대량의 데이터를 나누어서 전송하는데 사용할 수 있겠다. 또한 IoT처럼 여러 데이터를 계속 보낼때에도 사용할 수 있을 것이다. 

## Bi-Directional streaming
Server/Client 모두 데이터를 Stream으로 보내는 것이다. 한쪽에서 완료를 하면 다른 쪽에서도 완료를 하는 방식이다. 

Server/Client 모두 대량의 데이터를 나누어서 전송을 할때 사용할 수 있겠다. 또한 Chatting 과 같이 서로서로 계속 데이터를 주고 받을 경우 사용할 수 있다. 



# Protocol Buffer
전달할 데이터, Procedure를 Protocol Buffer로 정의해보자. src/main/proto 로 디렉토리를 만들고  패키지를 greet로 만든다. 만들어진 패키지/디렉토리에 greet.proto 파일을 만들자. 

proto version 3의 문법으로 작성하고, 패키지명, option을 선언한다. Java의 패키지명, 여러파일로 만들기 위한 설정을 한다. 

파일: greet.proto
```
syntax = "proto3";

package greet;

option java_package = "com.ahnchan.gprc.greet";
option java_multiple_files = true;
```

메세지를 하나 선언해보자. 여러 정보를 가지고 있는 자료 구조(Structure)라고 생각하면 된다. 
```
message Greeting {
  string first_name = 1;
  string last_name = 2;
}
```
Greeting이라는 메시지에 first_name과 last_name의 정보를 가지는 메시지를 선언하였다. 

이제 Client, Server 간 Call을 할 Service와 그 Service/Procedure 에서 사용할 메시지를 선언하면 된다. 이 부분은 각각의 통신방식을 구현할 때 선언할 것이다.  


# Unary
간단하게 Client에서 Greeting 정보를 Request 로 Server에 전송하면 받은 정보에서 first_name을 추출하여 Response로 반환하게 만들어 보겠다.
 
## Protocol Buffer 정의하기
service를 먼저 선언을 해보자. GreetService에 greet 라는 unary 방식의 rpc를 선언하였다. Request는 GreetRequest라는 메세지를 Response에는 GreetResponse라는 메세지를 선언하였다. 
```
service GreetService {
  // Unary
  rpc greet (GreetRequest) returns (GreetResponse) {};
}
```

이제는 rpc 에서 정의한 메세지들을 선언해보자. Request는 처음 정의한 Greeting 메세지를 하나 가지고 있다. Response는 String인 정보를 result로 넘겨주도록 하였다. 
```
message GreetRequest {
  Greeting greeting = 1;
}

message GreetResponse {
  string result = 1;
}
```

unary를 위한 메시지와 서비스가 구성되었으면 Gradle -> Other -> generateProto 을 이용하여 Generate해보자. 문제 없이 실행이 되면 /build/generated/source/proto/main/grpc 에 Java Class 파일로 generate되어 있는 것을 볼 수 있을 것이다. 


> Note. 이 튜토리얼에서는 설명을 하기 위해 Request / Response 메세지를 각각의 통신방식마다 다 따로 추가를 하였다. 뒤에 다른 방식에서도 Proto 파일을 추가하면 genernateProto를 실행해야 한다. 

## Server 코딩하기
Server 부분은 기존 dummyService를 만들었을 때와 비슷한데, Server를 Build하는 부분에서 Service를 추가(addService)해주면 된다. 

파일:  GreetingServer.java
```
public class GreetingServer {
    public static void main(String[] args) throws IOException, InterruptedException {
        System.out.println("Hello World - gRPC Server");

        Server server = ServerBuilder.forPort(50051)
                .addService(new GreetServiceImpl())
                .build()
                .start();

        Runtime.getRuntime().addShutdownHook(new Thread( () -> {
            System.out.println("Received shutdown request");
            server.shutdown();
            System.out.println("Successfully stopped the server");
        }));

        server.awaitTermination();
    }
}
```

.addService(new GreetServiceImpl()) 을 추가하면된다.  실제로 서버가 proto에 선언한 Service에 대해서 구현하는 부분이 GreetServiceImpl에 하면 된다. 

GreetServiceImpl은 generateProto 로 만들어진 GreetServiceGrpc.GreetServiceImplBase 를 상속을 받아 실제 서비스 부분을 구현하면 된다. 

greet 를 보면 void로 return을 하고 GreetRequest, StreamObsever의 두개의 Arguments를 받는다. Client에서 온 GreetRequest를 받아서 Observer로 반환하는 것이다. 

파일: GreetServiceImpl.java
```
public class GreetServiceImpl extends GreetServiceGrpc.GreetServiceImplBase {

    @Override
    public void greet(GreetRequest request, StreamObserver<GreetResponse> responseObserver) {
        Greeting greeting = request.getGreeting();
        String firstName = greeting.getFirstName();

        String result = "Hello " + firstName;
        GreetResponse response = GreetResponse.newBuilder()
                .setResult(result)
                .build();

        responseObserver.onNext(response);

        responseObserver.onCompleted();
    }
}
```

greet 는 GreetRequest에서 firstName 앞에 “Hello “ 를 붙여서 result로 만들고 Client에 Response 하도록 되어 있다. Observer로 Response를 해야하기에 GreetResponse를 Build하고 결과(result)로 설정해서 reponseObserver에 onNext에 넣고, onComplete를 하면 결과 OK로 전송이 된다. 

## Client 코딩하기
먼저 Client의 전체 구조를 만들어보겠다. Unary, Server streaming, Client streaming, Bi-directional 등 여러개를 구현해볼거라 조금은? 읽기 쉽게 작성을 해보자.

Client Channel을 생성하고 Security는 제외하게 usePlaintext()를 설정하여 빌드를 한다. 
그리고 각각 방식을 구형하기 위해 unary() 함수를 만들어 준다. 

파일: GreetingClient.java
```
public class GreetingClient {

    public static void main(String[] args) {
        GreetingClient client = new GreetingClient();
        client.run();
    }

    private void run() {
        System.out.println("Greeting gRPC client");

        ManagedChannel channel = ManagedChannelBuilder.forAddress("localhost", 50051)
                .usePlaintext()
                .build();

        unary(channel); // Unar
//        serverStreaming(channel); // Server streaming
//        clientStreaming(channel); // Client streaming
//        everyoneStreaming(channel);   // Bi-directional

        System.out.println("Shutting down channel");
        channel.shutdown();

    }

}
```
unary함수 이외에도 3개의 함수를 미리 선언하여 주석처리를 하였다. Server streaming, Client streaming, Bi-directional 을 구현하면서 하나씩 만들어 볼것이다. 

Unary 부분을 구현을 해보자. channel를 받아서 먼저 Synchronous/blocking 하게 Stub을 만들어보자. 

```
...
    private void unary(ManagedChannel channel) {
        // GreetService
        GreetServiceGrpc.GreetServiceBlockingStub greetClient = GreetServiceGrpc.newBlockingStub(channel);

        // Unary
        Greeting greeting = Greeting.newBuilder()
                .setFirstName("Gildong")
                .setLastName("Hong")
                .build();

        GreetRequest greetRequest = GreetRequest.newBuilder()
                .setGreeting(greeting)
                .build();

        GreetResponse result = greetClient.greet(greetRequest);
        System.out.println(result.getResult());
    }
…
```
그리고 Request로 전달할 Greeting 객체를 만들고 GreetRequest를 만든 객채를 가지고 build를 하자. 
proto에서 정의한  greet를 이용하여 greetRequest를 요청하고 결과를 result로 받는다. 


> Note. 통신하는 부분은 모두 gRPC에서 담당하기 때문에 Client에서 프로그램할때는 SDK의 Procedure를 사용하는 것과 같은 방식으로 사용하면 된다. 


## 실행해보기 
이제 Server, Client를 실행해보자. IntelliJ 에서 GreetServer에서 Run을 누르고 실행을 해본다. 
```
Hello World - gRPC Server
```
이후에 Client를 실행하여도 서버에서는 따로 출력되는 코드를 짜지 않아서 위와 같은 상태가 유지될 것이다. 

GreetClient를 실행해보자. 시작을 하면 서버에 바로 요청을 하고 Hello Gildong이라는 정보를 받아서 화면에 표시를 한다. 그리고 channel을 shutdown 한 것을 알수 있다. 
```
Greeting gRPC client
Hello Gildong
Shutting down channel
```

# Server streaming
이번에는 Server streaming을 만들어 보자. Client에서 Request 를 보내면, Server에서는 여러개의 데이터를 보내고 완료처리를 하면 완료가 된다. 

## Protocol Buffer 정의하기
Unary와 다른 부분은 return 부분의 정의에 stream 을 붙인 것이다. 다른 방식과 구분을 위해 Request와 Response는 따로 정의하였다. 

```
  // Server streaming
  rpc greetManyTime (GreetManyRequest) returns (stream GreetManyResponse) {}
```

rpc에 정의한 greetManyRequest와 GreetManyResponse도 정의해보자. 
```
message GreetManyRequest {
  Greeting greeting = 1;
}

message GreetManyResponse {
  string result = 1;
}
```

generateProto를 실행하여 Class 들을 generate해 보자.

## Server 코딩하기
Server는 이제 GreetServiceImpl 에 추가하면 된다. generateProto가 성공적으로 되었으면 Override할 함수가 하나 더 만들수 있을 것이다. 

> Note. IntelliJ에서 함수명을 쓰면 Override 할수 있는 메뉴가 만들어지고 기본 틀을 생성해 준다. 


greetManyTime 함수를 Override하고 로직을 구현해보자. 10번 데이터를 onNext에 넣어서 Client에 전송을 한다. 각각의 메시지를 구분하기 위해 no에 숫자를 증가시켜 넣어서 보낸다. 

```
@Override
    public void greetManyTime(GreetManyRequest request, StreamObserver<GreetManyResponse> responseObserver) {
        String firstName = request.getGreeting().getFirstName();

        try {
            for(int i=0; i<10; i++) {
                GreetManyResponse response = GreetManyResponse.newBuilder()
                        .setResult("Hello " + firstName + ", no: " + i)
                        .build();
                responseObserver.onNext(response);

                Thread.sleep(1000);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            responseObserver.onCompleted();
        }
    }
```

Unary에서 선언한 부분과 Argument 형식이 똑같은 것을 볼수 있다. 이 부분이 그래서 좀 헷갈린다. proto에 정의된 부분을 잘 보고 구현을 해야한다. 
Client에 정보를 전송하는 부분은 onNext 로 전송을 하고 onComplete로 완료하는 것은 똑같다. 

## Client 코딩하기
serverStreaming(channel) 함수를 만들어보자. procedure에 serverStreming을 선언하고 구현을 해보자. 

request를 작성하여 보내는 부분까지는 똑같다. 그런데 greetClient에 greetManyRequest를 보내고 forEachRemaing으로 Response 를 계속 받는다. Server에서 onComplete하면 forEashRemaing은 종료하게 된다. 

```
    private void serverStreaming(ManagedChannel channel) {
        // GreetService
        GreetServiceGrpc.GreetServiceBlockingStub greetClient = GreetServiceGrpc.newBlockingStub(channel);

        // Server streaming
        GreetManyRequest greetManyRequest = GreetManyRequest.newBuilder()
                .setGreeting(Greeting.newBuilder()
                        .setFirstName("Gildong")
                        .setLastName("Hong")
                        .build())
                .build();

        greetClient.greetManyTime(greetManyRequest)
                .forEachRemaining(greetManyResponse -> {
                    System.out.println(greetManyResponse.getResult());
                });
    }
```

## 실행해보기
Server는 기존에 실행이 되고 있었으면 Stop후에 다시 Run을 해야한다. Unary와 동일하게 메세지가 출력되면서 준비가 완료된다. 
```
Hello World - gRPC Server
```

Client를 실행해보자. 서버에서 메세지를 1초간격으로 보내게 처리해놔서 Client에는 메세지가 1초마다 한줄씩 나타날 것이다. 
``` 
Greeting gRPC client
Hello Gildong, no: 0
Hello Gildong, no: 1
Hello Gildong, no: 2
Hello Gildong, no: 3
Hello Gildong, no: 4
Hello Gildong, no: 5
Hello Gildong, no: 6
Hello Gildong, no: 7
Hello Gildong, no: 8
Hello Gildong, no: 9
Shutting down channel
```

# Client streaming
Client streaming은 Server streaming과 반대이다. Client에서 계속 데이터를 보내고 Server에서는 계속 데이터를 받는다. 그러다 Client에서 완료인 onComplete를 하면, Server는 완료 정보를 받고 지금까지온 데이터를 Response 하는 방식으로 구현하였다.

## Protocol Buffer 정의하기
rpc를 정의해보면 이번에는 Request에만 stream이 선언되어 있다.
```
  // Server Streaming
  rpc longGreet (stream LongGreetRequest) returns (LongGreetResponse) {};
```

메시지를 정의해보자.
```
message LongGreetRequest {
  Greeting greeting = 1;
}

message LongGreetResponse {
  string result = 1;
}
```

완료되었으면 generateProto를 실행한다. 

## Server 코딩하기
longGreet를 Override하면 이번에는 Return 값이 Request에 부분으로 StreamObserver 이다. 그리고 Argument는 Observer response 하나로 되어 있다. Request가 Return에 있어서 조금 헷갈린다. 
longGreet는 onNext, onError 와 onComplete를 구현하게 되어 있다. onNext는 Client에서 stream으로 보내는 데이터를 받는 부분이고, onError는 오류가 발생하였을 때의 처리부분, onComplete는 Client에서 데이터 전송을 완료하고 onComplete를 하였을때 처리하는 부분이다. 
여러개의 데이터를 onNext로 받고 완료가 되면 onComplete에서 Client에 한개의 Response를 보내는 것이다. 

```
    @Override
    public StreamObserver<LongGreetRequest> longGreet(StreamObserver<LongGreetResponse> responseObserver) {

        StreamObserver<LongGreetRequest>  longGreetRequest = new StreamObserver<LongGreetRequest>() {

            String result = "";

            @Override
            public void onNext(LongGreetRequest value) {
                result += " Hello " + value.getGreeting().getFirstName();
            }

            @Override
            public void onError(Throwable t) {

            }

            @Override
            public void onCompleted() {
                responseObserver.onNext(LongGreetResponse.newBuilder()
                        .setResult(result)
                        .build());
                responseObserver.onCompleted();
            }
        };

        return longGreetRequest;
    }
```

기존에 Unary나 Server streaming과는 다르게 구현하는 방법이라 헷갈릴 수 있다. IntelliJ에서는 Procedure를 Override하면 구현해야할 내부 Procedure를 선언해주니 onNext, onError와 onComplete만 잘 구분하여 구현하면 된다. 

## Client 코딩하기
Client 도 Server 처럼 헷갈린다. 순서대로만 잘 생각하면 구현하는데 문제는 없을 것이다. 

먼저 channel로 Stub을 생성한다. 그리고 requestObserver를 longGreet 로 정의를 하는데 Observer response를 구현하여 생성한다. Observer response는 Server에서 OnComplete를 모든 처리가 완료되었을 때를 처리하는 부분이다. 이 곳에서는 Client에서 메세지를 보낸것을 다 조합하여 서버가 Response로 보내게 되어 있다. 이때 보낸 result를 onNext에서 화면에 출력하게  되어 있다. 그리고 모든 것이 완료되면 onCompete가 된다. 

> Note. CountDownLatch는 Async통신을 하기 때문에 프로세스를 잠시 latch 상태로 두었다가 모든 것이 완료되면 “completed”를 출력하고 프로세스를 더 진행하게 하였다. 이 기능을 사용하지 않으면 최종 onComplete를 실행하지 않고 Client가 종료가 된다. 

여기까지 구현한 부분은 Server로 부터의 Response를 처리하는 부분이다. 그 이후에는 Client가 데이터를 보내는 부분이다. 
10번을 반복하여 requestObserver를 통해서 onNext에 LongGreetRequest 를 빌드하여 보낸다. 각각의 메세지를 구분하기 위해 no를 포함하게 만들었다. 


```
    private void clientStreaming(ManagedChannel channel) {
        GreetServiceGrpc.GreetServiceStub asyncClient = GreetServiceGrpc.newStub(channel);

        CountDownLatch latch = new CountDownLatch(1);

        StreamObserver<LongGreetRequest> requestObserver = asyncClient.longGreet(new StreamObserver<LongGreetResponse>() {
            @Override
            public void onNext(LongGreetResponse value) {
                // send from server onCompleted()
                System.out.println("Received message from server.");
                System.out.println(value.getResult());
            }

            @Override
            public void onError(Throwable t) {

            }

            @Override
            public void onCompleted() {
                // send to server sending stream is done.
                System.out.println("completed.");
                latch.countDown();
            }
        });

         for(int i=1; i<= 10; i++) {
             requestObserver.onNext(LongGreetRequest.newBuilder()
                     .setGreeting(Greeting.newBuilder()
                             .setFirstName("Gildong #" + i)
                             .build())
                     .build());
             try {
                 Thread.sleep(1000);
             } catch (InterruptedException e) {
                 e.printStackTrace();
             }

         }

         requestObserver.onCompleted();

        try {
            latch.await(3, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
```

## 실행해보기
Server를 실행해보면 따로 출력하는 부분이 없어서 똑같이 나타난다. 
```
Hello World - gRPC Server
```

Client는 실행하면 시작메세지가 나오고 약 10초정도 후에 아래와 같은 결과를 표시한다. 
```
Greeting gRPC client
Received message from server.
 Hello Gildong #1 Hello Gildong #2 Hello Gildong #3 Hello Gildong #4 Hello Gildong #5 Hello Gildong #6 Hello Gildong #7 Hello Gildong #8 Hello Gildong #9 Hello Gildong #10
completed.
Shutting down channel
```


# Bi-Directional streaming
이번에는 Client, Server 모두 stream으로 전송하도록 하겠다. 양쪽 모두에 stream을 받는 부분을 구현하면 된다.
 
## Protocol Buffer 정의하기
rpc를 선언할때 request, response에 모두 stream을 선언한다. 
```
  // Bi Directional streaming
  rpc greetEveryone (stream GreetEveryRequest) returns (stream GreetEveryResponse) {};
```

메시지도 선언을 한다. 
```
message GreetEveryRequest {
  Greeting greet = 1;
}

message GreetEveryResponse {
  string result = 1;
}
```

## Server 코딩하기
Server/Client의 stream을 구현해봤기에 아래 greetEveryone의 조는 익숙할 것이다. Server에서는 간단하게 Client가 접속하여 정보를 전달하면 Hello 를 붙여서 다시 데이터를 전달하도록 하였다. 클라이언트가 onComplete처리를 하면 Server도 종료 처리를 하게 되어 있다. 

```
    @Override
    public StreamObserver<GreetEveryRequest> greetEveryone(StreamObserver<GreetEveryResponse> responseObserver) {

        StreamObserver<GreetEveryRequest> greetEveryRequest = new StreamObserver<GreetEveryRequest>() {
            @Override
            public void onNext(GreetEveryRequest value) {
                System.out.println("Received from client: " + value.getGreet().getFirstName());
                String result = "Hello " +  value.getGreet().getFirstName();

                responseObserver.onNext(GreetEveryResponse.newBuilder()
                        .setResult(result)
                        .build());
            }

            @Override
            public void onError(Throwable t) {

            }

            @Override
            public void onCompleted() {
                responseObserver.onCompleted();
            }
        };

        return greetEveryRequest;
    }
```

## Client 코딩하기
Client는 먼저 Server에서 오는 정보를 처리하는 부분을 선언하였다. everyoneRequest를 정의하면서 greetEveyone을 구현하였다. 이 부분에서는 Server부터 오는 stream 정보를 화면에 출력하였다. 

Client는 Array로 되어 있는 이름의 정보들을 하나씩 보내게 되어있다. Client, Server가 메시지를 교환되는 것을 보기 위해 300ms를 설정하였다. 이 설정을 바꿔보면서 테스트해면 바로 알 수 있을 것이다. 

```
    private void everyoneStreaming(ManagedChannel channel) {
        GreetServiceGrpc.GreetServiceStub asyncClient = GreetServiceGrpc.newStub(channel);

        CountDownLatch latch = new CountDownLatch(1);

        StreamObserver<GreetEveryRequest> everyoneRequest = asyncClient.greetEveryone(new StreamObserver<GreetEveryResponse>() {
            @Override
            public void onNext(GreetEveryResponse value) {
                System.out.println("Received from server: "+ value.getResult());
            }

            @Override
            public void onError(Throwable t) {
                latch.countDown();
            }

            @Override
            public void onCompleted() {
                latch.countDown();
            }
        });


        Arrays.asList("Gildong 1", "Gildong 2", "Gildong 3", "Gildong 4", "Gildong 5").forEach(
                name -> {
                    System.out.println("Send to server: "+ name);
                    everyoneRequest.onNext(GreetEveryRequest.newBuilder()
                            .setGreet(Greeting.newBuilder()
                                    .setFirstName(name)
                                    .build())
                            .build());
                    try {
                        Thread.sleep(300);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
        );

        everyoneRequest.onCompleted();

        try {
            latch.await(3, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

## 실행해보기
Server를 실행하고 Client도 같이 실행을 해보겠다. Server는 Client의 접속을 기다렸다가 Client가 보내는 정보를 화면에 표시한다. Server는 메세지가 출력되는 것을 볼수 있다. 
```
Hello World - gRPC Server
Received from client: Gildong 1
Received from client: Gildong 2
Received from client: Gildong 3
Received from client: Gildong 4
Received from client: Gildong 5
```

Client는 Server에 보낸 정보와 받은 정보를 바로바로 출력하게 표시하였다. 메세지를 보내는 순서가 300ms의 간격이 있어서 아래와 같이 Server에 보낸 정보와 받은 정보를 확인할 수 있다. 

```
Greeting gRPC client
Send to server: Gildong 1
Send to server: Gildong 2
Send to server: Gildong 3
Received from server: Hello Gildong 1
Received from server: Hello Gildong 2
Received from server: Hello Gildong 3
Send to server: Gildong 4
Received from server: Hello Gildong 4
Send to server: Gildong 5
Received from server: Hello Gildong 5
Shutting down channel
```


# Conclusions
gRPC의 여러방식의 통신을 확인해봤다. 단순히 Request/Response(Unary)가 아닌 Stream데이터 통신도 처리할 수 있다. gRPC를 좀더 공부해보면 Security부분도 강화할 수 있고 HTTP2를 이용하여 좀더 빠르게 통신을 할 수도 있다. 또한 Unary 통신만 보더라도 JSON으로 통신하는 것보다 그 크기가 많이 줄어들어 데이터 사용량도 줄일 수 있다. (이 부분은 기회가 되면 테스트 프로그램을 만들어 보겠다.)


# References
[gRPC](https://docs.dapr.io/getting-started/quickstarts/)

[gRPC Concepts](https://grpc.io/docs/what-is-grpc/core-concepts/)

[gRPC [Java] Master Class: Build Modern API and Microservices ](https://learning.oreilly.com/videos/grpc-java-master/9781838558048/)

