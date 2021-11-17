---
layout: post
title:  "WebRTC Video Chat 만들기"
date:   2021-11-15 16:00:00 +0900
categories: Network
---
created : 2021-11-15, updated : 2021-11-15

# Introduction
WebRTC는 Device간의 Video, Audio와 Data까지 Peer to Peer로 통신을 할 수 있게 한다. 통신환경이 좋아져서 적은 사용자의 커넥션에서는 WebRTC가 중간 Streaming 서버 없이도 Device의 자원, 인터넷속도로도 Video/Audio 의 통신이 가능하다. 심지어는 모바일 에서도 가능하다.
Android, iOS등 모바일 뿐 아니라 웹 브라우져에서도 가능하여 손쉽게 서비스를 개발할 수 있다.

> Note. 본 튜토리얼은 OReiliy 의 Video 강의인 "[Practical WebRTC: A Complete WebRTC Bootcamp for Beginners](https://learning.oreilly.com/videos/practical-webrtc-a/9781801810012/)"을 듣고 코드로 구현한 것이다. 좀더 자세한 내용을 원하면 해당 강의를 듣기를 권장한다. 

좀더 이론적인 부분과 컨셉은 [WebRTC 공식 문서](https://webrtc.org/getting-started/overview?hl=en)를 확인하면 좋겠다.


# Pre-Installation
Node.js

> Note. 튜토리얼의 소스는 [https://github.com/ahnchan/tutorial-WebRTC](https://github.com/ahnchan/tutorial-WebRTC)에 있다.


# Concepts
WebRTC는 기본적으로 Peer To Peer 로 영상, 오디오를 서로 스트리밍을 하는 방식이다. 이를 위해서는 중간에 데이터를 전달하기 위한 Signaling Server와 NAT 환경에서도 사용이 가능하게 해주는 STUN Server, TURN Server가 필요하다. 이런 환경 구성은 NAT 환경에 있는 Client들은 IP로 직접 접속을 할 수가 없기 때문에 이런 서버들이 필요한 것이다. 

## Signaling Server
접속한 Client들이 정보(Signal)을 주고 받기 위해 사용한다. Client가 STUN Server에서 얻어온 정보인 ICE Candidate 정보와 offer, answer를 서로 교환하기 위해 사용한다. 
본 튜토리얼에서는 Node.js에서 socket.io를 이용하여 Signaling Server를 구현해볼 것이다. 

## STUN Server
Client와 Client간 접속을 할 수 있게 하기 위해 서로의 접속할 수 있는 정보(ICE Candidate)를 제공한다. 이 렇게 하는 이유는 NAT 안에 있는 Client에 직접 접속을 할 수 없기 때문이다. NAT 방식에 따라서는 데이터를 relay  해야하는 경우가 있는데 이럴 경우에는 TURN Server를 사용해야한다. 

## TURN Server
Client끼리 통신을 못할 경우에 사용하는 서버로 스트리밍 데이터를 전달해주는 역할을 한다. 이번 튜토리얼에서는 STUN만 사용하도록 하였다. 


# Signal에 따른 처리 Flow
간단하게 이번에 구성된 부분의 Signal들의 흐름을 정리해보았다. User A, User B가 ICE Candidate, offer, answer 정보를 Signaling Server 통해서 전달 받도록 되어 있다. 소스를 구성하면 좀 더 이해하기 편할 것이다. 

![Signal Flow](/posts/assets/network/WebRtc/images/webrtc-basic-signal-flow.png){: width="500"} 


> Note. candidate는 맨마지막에 전달하는 것은 아니다 offer, answer를 처리하는 중간에 서로 교환이 된다. Flow diagram에서는 표현하기 힘들어 맨 아래에 따로 넣었다. 

> Note. 이 부분은 시그널 명칭은 본인의 마음대로 설정할 수 있고 프로세스도 변경할 수 있다. 이 튜토리얼은 WebRTC를 연결하기 위해 상호 어떤 데이터가 언제 필요한지를 이해하는데 참조하면 좋겠다. 


# Video Chat Project를 만들기
Signaling Server이자 HTML, Javascript 를 Serving할 Project를 만들어 보자.

```
$ mkdir videochat
$ cd videchat
$ npm init
…
```

Node.js 서버에서는 express와 socket.io를 사용할 것이다. 프로젝트에 해당 라이브러리를 설치해보자.
```
$ npm install --save express socket.io
…
```

# Server 구성하기
맨처음 Signal에 대한 Flow에서 표시된 부분에 대해 개발을 한다. Signaling Server는 데이터를 Client간에 통신을 하는 목적이기 때문에 복잡하지는 않다. index.js 파일을 만들고 순서대로 코딩을 해보자.

## 필요 라이브러리 선언하기
프로젝트 초기화하고 npm으로 설치한 express, socket.io를 선언해보자. 
```
const express = require("express");
const socket = require("socket.io")
```
express 설정
express를 생성하고, 4000 port로 server를 구동한다. 현재는 4000 Port를 express가 열어 놓은 상태이고 아직 어떠한 일도 하지 않는다. 
```
const app = express();

let server = app.listen(4000, function () {
    console.log("Server is running!");
});
```

## HTML, CSS등 Static한 파일을 Serving 할수 있게 설정하기
public 디렉토리를 static 파일의 위치로 선언을 한다. HTTP로 요청오면 모두 이 디렉토리의 파일을 참조하도록 설정을 한 것이다. 

```
app.use(express.static("public"));
```

## Socket.io 설정하기
socket을 설정하고 접속이 되었을때 접속 socket id를 출력하게 해보자. 브라우저에서 socket.io로 접근을 하면 로그를 찍도록 하였다.
```
let io = socket(server);

io.on("connection", function (socket) {
    console.log("User connected: " + socket.id);

   // TODO - Signal
}
```

이제 TODO 로 커멘트 되어 있는부분에 Signal들을 구현하면 된다. Signal들의 처리는 간단하다. Signaling Server는 대부분이 Signal들을 접속한 사용자에게 전달하는 역할을 한다. 

## join 요청처리하기
User가 Room이름을 넣고 Join을 누르면 join 요청을 한다. 서버는 받아서 해당 Room이름이 있는지에 따라 created, joined로 나누어서 답변을 한다. 
```
    socket.on("join", function (roomName) {
        let rooms = io.sockets.adapter.rooms;
        let room = rooms.get(roomName);

        if (room == undefined) {
            // create a room
            socket.join(roomName);
            socket.emit("created");
        } else if (room.size == 1) {
            // join
            socket.join(roomName);
            socket.emit("joined");
        } else {
            // Full
            socket.emit("full");
        }

        console.log(rooms);
    });
```

created: Room 이 없는 경우 User에게 “created”를 전달한다.
joined: Room이 있는 경우 User에게 “joined” 를 전달한다.
full: Room에 인원이 다 채워졌을 경우 User에게 “full”을 전달한다. 

> Note. 이번 구성은 1:1 접속만 하려고 하여 Room의 인원은 2명을 넘을수 없다.
 
## ready 요청처리하기
두번째 접속하는 User가 joined 명령을 받은 상황에서 준비가 되면 ready 를 보낸다. 그러면 Signaling Server는 Room에 Join된 모든 User에게 ready 신호를 전송한다. 

```
    socket.on("ready", function (roomName) { 
        socket.broadcast.to(roomName).emit("ready");
    });
```

## candidate 요청처리하기
WebRTC 접근을 위해서는 상호 ICE Candidate를 전달해야한다. STUN Server나 TURN Server에서 받은 접속정보를 상호 교환하기 위한 부분이다. 
User에서 ICE candidate 정보를 받으면 서버에 요청하여 Room에 속한 모든 User에게 해당 정보를 보낸다. 
```
    socket.on("candidate", function (candidate, roomName) {
        console.log("candidate -------------");
        console.log(candidate);
        socket.broadcast.to(roomName).emit("candidate", candidate);
    });
```

## offer 요청처리하기
Room을 생성한 User에서 offer를 생성한 것을 Room의 다른 사용자에게 전달한다.
```
    socket.on("offer", function(offer, roomName) {
        console.log("offer -------------");
        console.log(offer);
        socket.broadcast.to(roomName).emit("offer", offer);
    });
```

## answer 요청처리하기
Room의 참가자(joined) User에서 answer를 생성한 것을 Room의 다른 사용자에게 전달한다.
```
    socket.on("answer", function(answer, roomName) {
        console.log("answer -------------");
        console.log(answer);        
        socket.broadcast.to(roomName).emit("answer", answer);
    });
```

> Note. Signaling Server는 User 간에 ICE candidate 정도와 offer, answer을 상황에 맞게 전달하는 역할을 한다. 


# Client 구성하기 
Client는 처음 Room을 만든 생성자와 이후에 접속하는 사용자로 구분하였다. creator라는 변수(Boolean)으로 구분을 하도록 되어 있다. Signal의 구분을 위해서 User A와 User B로 구분해서 작성을 해보겠다. Signal Flow에 명시한 Client의 명칭이랑 동일하다.  

> Note. 같은 Client코드에 있어서 좀 헷갈릴 수 있기에 생성자, 참가자로 구분을 하였다. 이 용어는 공식적인 명칭은 아니다. 

## Client 화면 구성하기
public 디렉토리를 만들고, HTML, CSS를 넣을 것이다. 이 부분은 본인의 디자인에 따라 만들면 좋으니 단순히 Git Repository에서 가져다가 사용을 해보자. 

파일: [index.html](https://github.com/ahnchan/tutorial-WebRTC/blob/main/public/index.html)
파일: [styles.css](https://github.com/ahnchan/tutorial-WebRTC/blob/main/public/styles.css)

[chart_init.js](https://github.com/ahnchan/tutorial-WebRTC/blob/main/public/chat_init.js) 를 가져와서 chart.js를 만들고 시작을 한다. 이 파일은 기본적인 변수만 지정해 놓았다. 완성된 chart.js는 GIT에 있으니 참조하기 바란다.  

## User A. Room 이름을 넣고 Join 요청하기
Room 이름을 입력하고 해당 Room에 Join을 요청한다. 이부분은 User A, User B가 같이 사용한다. 

```
join.addEventListener("click", function () {

    roomName = roomInput.value;
    socket.emit("join", roomName);
});
```

User A인 경우 Room 을 처음 만드는 것이기 때문에 “created” 신호가 돌아온다. 그 신호를 받아서 userVideo를 설정하고 대기를 한다. 
이 상태는 아직 User B가 Join을 하지 않아서 정보의 교환은 없는 상태이다. 
creator 변수를 true로 설정하여 해당 User가 Room의 생성자인 것을 명시한다. 

```
socket.on("created", function (data) {
    console.log("created");
    creator = true;

    navigator.mediaDevices.getUserMedia({
        audio: true,
        video: { width: 1024, height: 640 }
    }).then(function (stream) {
        userStream = stream;
        userVideo.srcObject = stream;
        userVideo.onloadedmetadata = function (e) {
            userVideo.play();
        }
    }).catch(function (err) {
        alert("Couldn't access user media");
    });
});
```

## User B, Room 이름을 넣고 Join 요청하기
User A에서 입력한 Room 이름과 동일하게 입력하고 Join을 하면 Signaling Server에서는 “joined”가 전달된다. User B의 userVideo를 설정하고 접속 준비가 되었다는 “ready” 정보를 Signaling Server에 전달한다. 

```
socket.on("joined", function () {
    console.log("joined");
    creator = false;

    navigator.mediaDevices.getUserMedia({
        audio: true,
        video: { width: 1024, height: 640 }
    }).then(function (stream) {
        userStream = stream;
        userVideo.srcObject = stream;
        userVideo.onloadedmetadata = function (e) {
            userVideo.play();
        }
        socket.emit("ready", roomName);
    }).catch(function (err) {
        alert("Couldn't access user media");
    });
});
```

Signaling Server에서 “full”이 전달되면 Alert을 표시한다. 
```
socket.on("full", function () {
    alert("Room is full");
});
```

## User A. ready를 받고 RTCPeerConnection 생성
Signaling Server는 “ready”를 Room의 모든 User에 전달을 하지만 이 부분은 creator만 받아서 처리하도록 한다. 그러므로 User A만 처리하게 될 것이다. 
STUN  혹은 TURN Server 정보를 가지고 RTCPeerConnection 객체를 생성한다. 

먼저 STUN 혹은 TURN Server의 정보를 만든다.  이 튜토리얼에서는 Public Free STUN 서버를 사용해보겠다. 
```
let iceServers = {
    iceServers: [
        { urls: "stun:stun.services.mozilla.com" },
        { urls: "stun:stun.l.google.com:19302" },
    ]
};
```

이제 “ready”에 대한 처리를 해보자. User B에서 보낸 ready는 User A에서만 처리하도록 하였다. creator인 사용자만 처리하도록 하였다. RTCPeerConnection을 생성하면 ICE Candidate 정보를 받아진다. 이 정보를 다른 User에게 전달을 해야되서 oniIceCnadidateFunction에 구현을 하였다. 
User A에서는 먼저 offer 를 생성해서 User B에게 전달해야한다. Offer와 cadicate가 전달이 완료되면 User A와 User B의 연결이 설정되어 Video, Audio를 서로 Streaming하게 된다. (onTrackFunction)

```
socket.on("ready", function () {
    console.log("ready");
    if (creator) {
        rtcPeerConnection = new RTCPeerConnection(iceServers);
        rtcPeerConnection.onicecandidate = onIceCandidateFunction;
        rtcPeerConnection.ontrack = onTrackFunction;
        rtcPeerConnection.addTrack(userStream.getTracks()[0], userStream);  // audio
        rtcPeerConnection.addTrack(userStream.getTracks()[1], userStream);  // video
        rtcPeerConnection.createOffer()
            .then((offer) => {
                console.log("createOffer");
                rtcPeerConnection.setLocalDescription(offer);
                socket.emit("offer", offer, roomName);
            })
            .catch((error) => {
                console.log(error);
            });
    }
});

function onIceCandidateFunction(event) {
    console.log("onIceCandidate");
    if (event.candidate) {
        socket.emit("candidate", event.candidate, roomName);
    }
}

function onTrackFunction(event) {
    console.log("onTrack");

    navigator.mediaDevices.getUserMedia({
        audio: true,
        video: { width: 512, height: 320 }
    }).then(function (stream) {
        peerVideo.srcObject = stream;
        peerVideo.onloadedmetadata = function (e) {
            peerVideo.play();
        }
    }).catch(function (err) {
        alert("Couldn't access user media");
    });

    peerVideo.srcObject = event.streams[0];
    peerVideo.onloadedmetadata = function(e) {
        peerVideo.play();
    }
}

socket.on("candidate", function (candidate) {
    console.log("candidate");
    rtcPeerConnection.addIceCandidate(new RTCIceCandidate(candidate));
});
```

## User B. offer를 받고 RTCPeerConnection 생성
User A가 보낸 offer를 받으면, RTCPeerConnection 생성한다. User A에서 RTCPeerConnection을 생성할때와 동일하게  STUN Server에서 ICE Candidate를 받아 User A에게 전달을 한다.  
offer 정보를 Remote Description에 설정하고 answer를 생성한다. 생성된 answer는 User A에게 전달되도록 한다. 

```
socket.on("offer", function (offer) {
    console.log("offer");
    if (!creator) {
        rtcPeerConnection = new RTCPeerConnection(iceServers);
        rtcPeerConnection.onicecandidate = onIceCandidateFunction;
        rtcPeerConnection.ontrack = onTrackFunction;
        rtcPeerConnection.addTrack(userStream.getTracks()[0], userStream);  // audio
        rtcPeerConnection.addTrack(userStream.getTracks()[1], userStream);  // video
        rtcPeerConnection.setRemoteDescription(offer);
        rtcPeerConnection.createAnswer()
            .then((answer) => {
                console.log("createAnswer");
                rtcPeerConnection.setLocalDescription(answer);
                socket.emit("answer", answer, roomName);
            })
            .catch((error) => {
                console.log(error);
            });
    }
});
```

## User A. answer를 받고 remoteDescription 설정
User B가 보낸 answer 정보를 User A가 받아서 remoteDescription 을 설정을 한다. 

```
socket.on("answer", function (answer) {
    console.log("answer");
    if (creator) {
        rtcPeerConnection.setRemoteDescription(answer);
    }
});
```

> Note. candidate, offer, answer 정보는 Peer to Peer로 연결하고 Streaming을 위해 필요하다. 이 곳에서는 저장소 없이 상호 정보를 제공하기 위해 조금은 헷갈리는 구조가 되어 있으나. 생성자와 참여자로 구분하여 ICE Candidate, Offer, Answer를 저장소에 저장하여 사용할 수도 있다. 


# Conclusions
WebRTC의 기본 연결 방법을 알아봤다. 실제 운영환경에서 다양한 Network, NAT환경을 고려해야할 것 이다. STUN만으로 만될수도 있어 TURN Server을 구성해야할 수도 있을 것이다. User간의 각종 정보를 교환하는 방식또한 상황에따라 본 튜토리얼에서처럼 Signal(Event) 를 서로 전달하면서 할수도 있고 Storage (DB, Memory 등등)를 이용할 수도 있겠다. 


# References
[Practical WebRTC: A Complete WebRTC Bootcamp for Beginners](https://learning.oreilly.com/videos/practical-webrtc-a/9781801810012/)

[WerRTC Guide](https://webrtc.org/getting-started/overview?hl=en)

[Socket.io](https://socket.io/docs/v4/)

