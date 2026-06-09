---
title: "RTMP는 Flash가 죽어도 살아남았다 — 그리고 SRT가 필요한 순간"
date: 2026-06-09 21:00:00 +0900
categories: [백엔드, 스트리밍]
tags: [rtmp, srt, streaming, hls, tcp, udp, obs, ffmpeg, arq, aes]
---

스트리밍 프로토콜을 공부하다 보니 묘한 게 있었다.

HLS, DASH는 "시청자에게 뿌리는" 프로토콜인데, OBS에서 방송 시작 버튼을 누르면 실제로 무슨 프로토콜로 서버에 영상을 보내는 건지 한 번도 제대로 생각해본 적이 없었다. 그냥 유튜브 스트림 키 넣고 버튼 누르면 되는 거 아닌가 했는데, 그 뒤에 RTMP가 있었다.

![스트리밍 전체 파이프라인](/assets/img/posts/rtmp-srt-streaming/streaming-pipeline.png)

---

# RTMP — Flash의 유산이 왜 아직도 살아있나

RTMP(Real-Time Messaging Protocol)는 Adobe가 2002년경 Flash Player와 Flash Media Server 간 통신을 위해 만든 프로토콜이다.

당시에는 브라우저가 영상을 직접 재생하는 기능이 없었다. 영상을 보려면 Flash Player를 따로 설치해야 했고, Flash끼리 실시간으로 영상을 주고받으려면 전용 통신 규약이 필요했다. HTTP는 클라이언트가 먼저 요청해야 서버가 응답하는 구조라, 서버가 지속적으로 데이터를 밀어줘야 하는 실시간 스트리밍에는 맞지 않았다.

그래서 RTMP는 TCP 기반의 지속 연결(Persistent Connection)을 택했다. 한 번 연결을 맺으면 끊기 전까지 오디오·비디오·메타데이터를 동시에 흘려보낼 수 있다.

```
OBS (인코더) ─── RTMP ──► 서버 ─── HLS/DASH ──► 시청자
```

RTMP는 딱 왼쪽 화살표, 즉 인코더에서 서버로 영상을 밀어넣는 **인제스트(Ingestion) 구간**만 담당한다. 시청자에게 뿌리는 건 HLS나 DASH가 맡는다.

---

## TCP를 선택한 이유와 그 대가

RTMP가 TCP를 고른 건 합리적인 선택이었다. 2000년대 초 인터넷은 지금보다 훨씬 불안정했고, 패킷이 중간에 사라지는 일이 잦았다. UDP를 쓰면 손실 복구 로직을 직접 구현해야 하는데, TCP를 쓰면 OS가 재전송을 알아서 처리해준다.

대신 **청크(Chunk) 방식**으로 지연을 최소화했다. 메시지를 기본 128바이트 단위로 쪼개서 오디오·비디오 청크를 번갈아 전송하는 방식이다. 덕분에 큰 비디오 프레임을 보내는 중에도 오디오가 끊기지 않는다.

```bash
ffmpeg \
  -re -i input.mp4 \
  -c:v libx264 -preset veryfast -b:v 2500k \
  -g 60 \          # GOP 크기: 30fps 기준 2초마다 키프레임
  -c:a aac -b:a 128k \
  -f flv \         # RTMP는 FLV 컨테이너
  rtmp://localhost:1935/live/stream
```

근데 여기서 `-g 60`이 생각보다 중요하다. GOP(Group of Pictures)는 키프레임 하나와 그 뒤를 따르는 프레임들의 묶음인데, 새로 접속한 시청자는 다음 키프레임이 올 때까지 기다려야 한다. GOP를 300으로 설정하면(10초) 새 시청자는 최대 10초를 기다린다. 나는 이걸 처음에 몰라서 왜 새 시청자가 한참 검은 화면을 보는지 한참 헤맸다.

---

## 2020년 Flash가 죽었는데 RTMP는 왜 살아있나

Adobe가 Flash를 공식 종료한 게 2020년 12월이다. 그런데 지금도 유튜브, 트위치, 아프리카TV 모두 RTMP ingest를 지원한다.

이유는 간단하다. OBS가 RTMP로 동작하고, 유튜브·트위치가 RTMP를 받아주는 한 아무도 바꾸려 하지 않는다. 수억 명이 쓰는 생태계를 바꾸려면 인코더 소프트웨어와 플랫폼이 동시에 움직여야 하는데, 현실적으로 쉽지 않다.

"고장나지 않으면 고치지 않는다"는 레거시의 힘이다. 이 부분이 꽤 인상적이었다. 기술적으로 더 나은 대안이 있어도 생태계가 버티면 살아남는다는 게.

---

# SRT — UDP인데 왜 더 안전한가

RTMP(TCP)의 문제는 열악한 네트워크에서 드러난다.

위성 인터넷이나 해외 장거리 구간은 RTT(왕복 지연)가 200~600ms씩 나고 패킷 손실도 잦다. 이런 환경에서 TCP를 쓰면 **HOL Blocking(Head-of-Line Blocking)** 이 발생한다.

TCP는 패킷 순서를 보장하기 위해, 유실된 패킷이 재전송돼 올 때까지 그 뒤에 정상 도착한 패킷들도 전달을 보류한다. 패킷 손실이 잦은 환경에서는 버퍼링이 폭증하고, 최악의 경우 연결이 끊어진다.

2017년 Haivision이 이 문제를 해결하려고 SRT(Secure Reliable Transport)를 오픈소스로 공개했다. 핵심 아이디어는 "UDP로 빠르게 보내되, 유실된 건 조용히 재전송하자."

---

## ARQ — HOL Blocking 없는 재전송

![TCP HOL Blocking vs SRT ARQ 비교](/assets/img/posts/rtmp-srt-streaming/hol-blocking-vs-arq.png)

SRT의 신뢰성은 **ARQ(Automatic Repeat reQuest)** 로 구현된다.

```
[1] 송신 측: 패킷을 UDP로 전송, 버퍼에 잠시 보관
[2] 수신 측: 빠진 패킷 번호 발견 → NAK(Negative Acknowledgement) 전송
[3] 송신 측: 해당 패킷만 재전송
[4] 나머지 패킷은 계속 수신 중 → HOL Blocking 없음
```

TCP는 유실 패킷 때문에 스트림 전체가 멈추지만, SRT는 유실된 것만 조용히 처리한다. 이 차이가 불안정한 네트워크에서 결정적이다.

그리고 **Latency Buffer**가 ARQ가 작동할 시간을 보장한다. 수신 측에서 예를 들어 200ms 버퍼를 설정하면, 받은 영상을 바로 재생하지 않고 200ms 모아뒀다가 재생한다. 그 시간 안에 유실 패킷이 재전송돼 오면 빈칸이 채워진다.

```bash
# SRT 송신 (FFmpeg)
ffmpeg \
  -re -i input.mp4 \
  -c:v libx264 -preset veryfast -b:v 2500k \
  -c:a aac -b:a 128k \
  -f mpegts \     # SRT는 MPEG-TS 컨테이너
  "srt://서버IP:9000?mode=caller&latency=200&pbkeylen=16&passphrase=비밀번호1234"

# SRT 수신 (서버 측)
ffmpeg \
  -i "srt://0.0.0.0:9000?mode=listener" \
  -c copy output.ts
```

Latency 설정값은 RTT의 3~4배가 권장이다.

```
RTT 10ms  → latency 40ms   (같은 데이터센터)
RTT 50ms  → latency 200ms  (국내 서버)
RTT 200ms → latency 800ms  (해외, 위성)
```

나는 처음에 "latency 낮을수록 실시간이겠지" 하고 무작정 낮게 설정했다가, 오히려 더 끊기는 걸 경험했다. Latency가 RTT보다 짧으면 재전송 패킷이 도착하기도 전에 재생 시간이 지나버린다. ARQ가 있어도 소용없는 상황이 된다.

---

## AES 암호화가 기본 내장

RTMP는 암호화가 없다. 스트림 키가 URL에 평문으로 노출되고, 누군가 중간에서 가로채면 내용을 볼 수 있다.

SRT는 AES-128/256 암호화가 기본 내장이다. passphrase 하나만 설정하면 자동으로 Diffie-Hellman 키 교환 후 암호화된다. 별도의 TLS 설정 없이도 군사급 암호화가 적용된다는 게 방송 기여(contribution) 구간에서 큰 장점이다.

한 가지 삽질 포인트: passphrase는 반드시 10자 이상이어야 한다. 9자 이하면 연결 자체가 안 된다. 에러 메시지가 이걸 명확히 알려주지 않아서 연결이 계속 거부될 때 한참 다른 원인을 찾았다.

---

# RTMP vs SRT — 어떤 걸 선택해야 하나

결국 둘의 차이는 **네트워크 환경**이다.

![RTMP vs SRT 비교](/assets/img/posts/rtmp-srt-streaming/rtmp-vs-srt-comparison.png)

| 항목 | RTMP | SRT |
|------|------|-----|
| 전송 기반 | TCP | UDP |
| 패킷 손실 허용 | ~1% | ~25% |
| 지연 | 1~3초 | 0.1~1초 |
| 암호화 | 없음 (RTMPS는 TLS) | AES-128/256 내장 |
| 주 용도 | 인코더→서버 인제스트 | 현장→서버 기여(contribution) |

집에서 유튜브 방송이라면 RTMP로 충분하다. 인터넷이 안정적이면 TCP의 단점이 잘 드러나지 않는다. 하지만 스포츠 현장 중계, 해외 송출, 위성 구간처럼 네트워크가 불안정한 환경이라면 SRT가 확실히 낫다.

의사결정 기준을 하나 잡으면: **패킷 손실률 2% 이하면 RTMP, 초과하면 SRT**.

---

# 마무리

사실 이 둘을 공부하기 전까지는 "방송 서버에 RTMP로 보내면 끝 아닌가" 정도로만 알고 있었다. 근데 RTMP가 TCP 기반이라 열악한 환경에선 HOL Blocking이 발생한다는 걸, 그리고 SRT가 UDP 위에 ARQ와 Latency Buffer를 얹어서 그 문제를 해결했다는 걸 알고 나니 왜 방송 장비 업계에서 SRT로 넘어가고 있는지 납득이 됐다.

SW마에스트로 프로젝트에서 실시간 스트리밍을 다룰 일이 생기면, 네트워크 환경에 따라 RTMP와 SRT 중에서 고르는 판단 기준이 생긴 것 같아서 수확이 있었다.

다음은 WebRTC다. SRT가 서버 기반 스트리밍의 해결책이라면, WebRTC는 브라우저 간 P2P 실시간 통신의 해결책이다. ICE, STUN/TURN부터 시작해야 할 것 같다.
