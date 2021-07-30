---
layout: post
title:  "Node.js 에서 QUIC 설정/컴파일하기"
date:   2021-07-29 16:00:00 +0900
categories: Network
---
created : 2021-07-29, updated : 2021-07-29

# Introduction
HTTP/3에 대해 동작을 확인하기 위해 Node.js에서 실행을 해본다. 정식 Release에는 포함되어 있지 않고 Compiie Config를 수정해야 한다. 
본 문서에서는 MacOS에서 환경을 구성하여 만들어보도록 하겠다.


# Node QUIC 버전 컴파일하기 (15.0.0-pre)

Node 에서 Experimental 기능인 QUIC를 enable 하고 컴파일을 해로 해야한다. 
먼저 Git에서 QUIC가 적용된 소스를 받아온다. 

```
% git clone https://github.com/nodejs/quic.git
% cd quic
```

QUIC 는 experimental 로 configure로 enable을 시켜준다. 

```
% ./configure --experimental-quic --coverage
Node.js configure: Found Python 3.8.3...
INFO: configure completed successfully
```


Node를 컴파일 한다. 이 작업은 시간이 좀 걸린다. 

```
% make -j4 coverage
/Applications/Xcode.app/Contents/Developer/usr/bin/make -C out BUILDTYPE=Release V=0
  LD_LIBRARY_PATH=/Users/jdins/Workspaces/Workspaces/quic/quic/out/Release/lib.host:/Users/jdins/Workspaces/Workspaces/quic/quic/out/Release/lib.target:$LD_LIBRARY_PATH; export LD_LIBRARY_PATH; cd ../.; mkdir -p /Users/jdins/Workspaces/Workspaces/quic/quic/out/Release/obj/gen; dtrace -h -xnolibs -s src/node_provider.d -o "/Users/jdins/Workspaces/Workspaces/quic/quic/out/Release/obj/gen/node_provider.h"
  ...
```

오류없이 실행이 되었으면 node 실행 파일을 만들었을 것이다. 아래의 명령문으로 잘만들어졌는지 확인을 해보자. 

```
% ./out/Release/node --version
v15.0.0-pre
```

QUIC 기능은 v15부터 Experimental 로 제공하고 있다. 해당 소스는 15.0을 기준으로 QUIC를 컴파일할수 있게 만들어져 잇따. 


# QUIC 테스트 해보기

먼저 작업할 디렉토리를 만들어보자.

```
% cd ..
% mkdir server
% cd server
```

간단하게 quic 기능이 enable 되서 컴파일이 잘되었는지 확인을 해보자. quic.js를 만들어보자.

```
% nano quic.js
```
```
const quic = require('net');
console.log(quic);
```


컴파일한 node로 quic.js 를 실행해보자.

```
% ../quic/out/Release/node quic.js
{
  _createServerHandle: [Function: createServerHandle],
  _normalizeArgs: [Function: normalizeArgs],
  _setSimultaneousAccepts: [Function: _setSimultaneousAccepts],
  connect: [Function: connect],
  createConnection: [Function: connect],
  createServer: [Function: createServer],
  isIP: [Function: isIP],
  isIPv4: [Function: isIPv4],
  isIPv6: [Function: isIPv6],
  Server: [Function: Server],
  Socket: [Function: Socket],
  Stream: [Function: Socket],
  createQuicSocket: [Function: createQuicSocket]
}
```

맨아래에 createQuickSocket이 있는 것을 볼수 있다. 이제 QUIC 를 사용할수 있게 node가 잘 컴파일되었다. 

미리 Node가 설치되어 있었으면 아래의 명령으로 QUIC가 설치된 버전과 안된 버전의 차이를 확인할 수 있다. 

```
% node quic.js
{
  _createServerHandle: [Function: createServerHandle],
  _normalizeArgs: [Function: normalizeArgs],
  _setSimultaneousAccepts: [Function: _setSimultaneousAccepts],
  BlockList: [Getter],
  SocketAddress: [Getter],
  connect: [Function: connect],
  createConnection: [Function: connect],
  createServer: [Function: createServer],
  isIP: [Function: isIP],
  isIPv4: [Function: isIPv4],
  isIPv6: [Function: isIPv6],
  Server: [Function: Server],
  Socket: [Function: Socket],
  Stream: [Function: Socket]
}
```


# QUIC Echo 서버/클라이언트 만들기
[링크](https://www.nearform.com/blog/a-quic-update-for-node-js/) 의 예제를 수정하여 만들었다. 
간단히 클라이언트에서 텍스트를 입력하면 서버를 통해서 다시 클라이언트로 전송하는 구조이다. 

## SSL 만들기
QUIC는 TLS 기반이기에 통신을 위한 SSL 만들어 보자.

ssl   키를 넣을 디렉토리를 만든다.

```
% mkdir ssl
% cd ssl
```

Key를 생성한다.

```
% ​​openssl genrsa 2048 > server.key
% openssl req -new -key server.key -subj "/C=KR" > server.csr
% openssl x509 -req -days 1000 -signkey server.key < server.csr > server.crt
```


## Echo Server
서버(server.js)를 만들어 보자.

```
% nano server.js
```
```
const { createQuicSocket } = require('net');
const { readFileSync } = require('fs');

const key  = readFileSync('./ssl/server.key');
const cert = readFileSync('./ssl/server.crt');
const ca   = readFileSync('./ssl/server.csr');

const requestCert = true;

const alpn = 'echo';
const server = createQuicSocket({
    // Bind to local UDP port 5678
    endpoint: { port: 5678 },
    // Create the default configuration for new
    // QuicServerSession instances
    server: {
        key,
        cert,
        ca,
        requestCert,
        alpn
    }
});

server.listen();

server.on('ready', () => {
    console.log('QUIC server is listening ');
});

server.on('session', (session) => {
    session.on('stream', (stream) => {
        // Echo server!
        stream.pipe(stream); 
    });

    const stream = session.openStream();
    stream.end('hello from the server');
});
```


## Echo Client
클라이언트(client.js)를 만들어보자. 

```
% nano client.js
```
```
const { createQuicSocket } = require('net');
const { readFileSync }  = require('fs');

const key  = readFileSync('./ssl/server.key');
const cert = readFileSync('./ssl/server.crt');
const ca   = readFileSync('./ssl/server.csr');

const requestCert = true;

const alpn = 'echo';
const servername = 'localhost';
const socket = createQuicSocket({
    endpoint: { port: 8765 },
    client: {
        key,
        cert,
        ca,
        requestCert,
        alpn,
        servername
    }
});


const req = socket.connect({
    address: 'localhost',
    port: 5678,
});


req.on('secure', () => {
    const stream = req.openStream();

    // input by key
    process.stdin.pipe(stream);

    stream.on('data', (chunk) => { 
        console.info('echo : ', chunk.toString());
    })
    stream.on('end', () => { 
        console.info('end');
    })
    stream.on('close', () => {
        // Graceful shutdown
        socket.close();
    })

    stream.on('error', (err) => {
        console.error(error);
    })

})
```


## 서버/클라이언트를 구동하기

터미널을 2개 열어서 서버와 클라이언트를 구동한다. 

```
% ../quic/out/Release/node server.js
(node:86492) ExperimentalWarning: The QUIC protocol is experimental and not yet supported for production use
(Use `node --trace-warnings ...` to show where the warning was created)
QUIC server is listening 
```

클라이언트를 실행하고 Keyboard 에 텍스트를 입력한다. 

```
% ../quic/out/Release/node client.js --trace-warning
(node:86559) ExperimentalWarning: The QUIC protocol is experimental and not yet supported for production use
(Use `node --trace-warnings ...` to show where the warning was created)
Hello World
echo :  Hello World

TEST
echo :  TEST
```


# Conclusions
간단하게 Node.js 에서 QUIC를 활성해서 컴퍼일하고 간단한 예제를 실행해봤다. IETF에서 표준이되었지만, 개발은 좀더 해야 실제 운영환경에서 사용이 가능할 것 같다. 


# References
[GUIHub QUIC](https://github.com/nodejs/quic)

[For new contributors: here's how to build Node.js with QUIC enabled](https://github.com/nodejs/quic/issues/183)

[A QUIC Update for Node.js](https://www.nearform.com/blog/a-quic-update-for-node-js/)

[Try QUIC in Node.js on Docker](https://dev.to/nwtgck/try-quic-in-node-js-on-docker-l8c)


