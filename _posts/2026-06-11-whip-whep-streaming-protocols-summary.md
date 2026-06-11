---
title: "RTMP의 시대가 끝나가고 있다 — WHIP·WHEP과 스트리밍 프로토콜 선택 기준"
date: 2026-06-11 22:00:00 +0900
categories: [백엔드, 스트리밍]
tags: [whip, whep, webrtc, rtmp, hls, llhls, dash, srt, streaming, obs, cloudflare, sfu, sdp, ietf]
---

스트리밍 프로토콜 시리즈를 마무리하면서 마지막 단계가 WHIP·WHEP이었다. HLS부터 시작해서 LLHLS, DASH, RTMP, SRT, WebRTC까지 쭉 달려왔다.

솔직히 처음엔 "또 약어네" 하고 가볍게 봤다. 근데 공부하면서 "아, 이게 RTMP를 대체할 미래 표준이구나"라는 게 느껴졌다. 특히 OBS 30.0에 기본 탑재됐다는 걸 알고 나서부터는 그냥 흥미로운 기술이 아니라 당장 현장에서 쓰이는 기술로 보이기 시작했다.

---

# RTMP가 여전히 표준인 이유, 그리고 한계

WebRTC를 배우면서 한 가지 이상한 점이 있었다. WebRTC는 0.5초 이하 지연으로 영상을 실시간 전송할 수 있는데, 왜 방송인들은 아직도 RTMP로 서버에 영상을 보낼까?

이유는 단순했다. **WebRTC 표준엔 "어떻게 연결을 시작하나"가 없었다.**

WebRTC 스펙을 보면 영상·음성을 암호화해서 실시간 전송하는 방법은 아주 잘 정의돼 있다. 근데 연결을 맺기 위한 SDP 교환을 어떤 방식으로 해야 하는지는 안 나와 있다. 그러다 보니 Twitch는 자기들만의 방식으로, YouTube는 또 다른 방식으로 구현했다. OBS가 모든 플랫폼을 지원하려면 플랫폼마다 다른 플러그인이 필요했다.

RTMP가 2002년에 만들어진 Flash 시대 유산임에도 아직까지 인제스트(방송인 → 서버) 표준으로 남아있는 이유가 여기 있었다. 불편하고 최신 코덱도 못 쓰지만, 그냥 표준이었기 때문이다.

![RTMP는 표준이었고 WebRTC는 연결 방법이 없었다](/assets/img/posts/whip-whep-streaming-protocols-summary/rtmp-webrtc-problem.png)

---

# WHIP — HTTP POST 한 방으로 WebRTC 연결

WHIP은 2023년 IETF에서 표준화됐다. **WebRTC HTTP Ingestion Protocol**, 방송인이 서버에 WebRTC로 영상 보내기를 시작할 때 쓰는 규약이다.

핵심은 단순하다. HTTP POST 요청 하나로 WebRTC 연결에 필요한 SDP 교환을 끝낸다.

```
OBS  ──  POST /whip/live  →  서버
         Content-Type: application/sdp
         Body: SDP Offer ("나 H.264로 보낼게, 받아줄 수 있어?")

서버  ──  201 Created  →  OBS
         Location: /whip/live/abc123   (나중에 방송 끊을 때 쓸 URL)
         Body: SDP Answer ("응, 받을게. 내 주소는 여기야")
```

이 HTTP 교환이 끝나면 그 다음부터는 WebRTC가 알아서 한다. ICE로 경로 찾고, DTLS로 암호화 키 교환하고, SRTP로 실제 영상을 흘린다.

HTTP 통신은 그냥 악수 한 번이었던 거다. 악수 끝나면 진짜 대화(영상 스트림)는 WebRTC가 한다.

![WHIP 연결 4단계 흐름](/assets/img/posts/whip-whep-streaming-protocols-summary/whip-connection-flow.png)

RTMP랑 비교하면 얻는 게 꽤 많다.

| | RTMP | WHIP |
|--|------|------|
| 전송 기반 | TCP | UDP (WebRTC) |
| 지연 | 1~3초 | 0.3~0.5초 |
| 암호화 | 없음 (기본) | DTLS+SRTP |
| 최신 코덱 | H.264만 | H.265, AV1 가능 |
| 방화벽 통과 | RTMP 전용 포트 필요 | HTTP(S) 포트 사용 |

가장 인상 깊었던 건 암호화다. RTMP는 기본적으로 평문 전송인데, WHIP은 WebRTC 기반이니까 DTLS+SRTP 암호화가 기본으로 붙어온다.

![RTMP vs WHIP 항목별 비교](/assets/img/posts/whip-whep-streaming-protocols-summary/rtmp-vs-whip-compare.png)

---

# WHEP — 시청자 측 표준

WHEP은 **WebRTC HTTP Egress Protocol**. 서버에서 시청자 브라우저로 영상이 나가는 구간을 표준화한 것이다.

구조가 WHIP이랑 거의 똑같다. HTTP POST 한 번으로 SDP 교환하고, WebRTC로 영상 받는다. 딱 한 가지만 다르다.

```
WHIP SDP:  a=sendonly   ← 나는 보내기만 할게
WHEP SDP:  a=recvonly   ← 나는 받기만 할게
```

전체 흐름을 하나로 그리면 이렇게 된다.

```
[OBS]
  │ WHIP: HTTP POST (SDP Offer, sendonly)
  ↓ 서버: HTTP 201 (SDP Answer)
[SFU 서버]
  │ ICE → DTLS → SRTP (실제 영상 수신)
  │ 수천 명에게 스트림 복사
  ↓
[시청자 브라우저]
  │ WHEP: HTTP POST (SDP Offer, recvonly)
  ↓ 서버: HTTP 201 (SDP Answer)
  → ICE → DTLS → SRTP (영상 수신, 0.3~0.5초 지연)
```

![OBS → WHIP → SFU → WHEP → 시청자 전체 파이프라인](/assets/img/posts/whip-whep-streaming-protocols-summary/whip-whep-full-pipeline.png)

WHIP 이전엔 이 연결 방식을 플랫폼마다 따로 구현했다. 이제는 HTTP POST 스펙이 통일됐으니까 OBS 같은 도구가 플랫폼 종류에 상관없이 하나의 코드로 대응할 수 있다. OBS 30.0이 WHIP을 기본 탑재한 게 그 결과다.

---

# SDP가 뭔지 처음으로 제대로 이해했다

![SDP는 WebRTC 연결 전 주고받는 자기소개서](/assets/img/posts/whip-whep-streaming-protocols-summary/sdp-explained.png)

WHIP·WHEP을 공부하면서 사실 제일 큰 수확은 SDP를 제대로 이해한 거였다.

SDP(Session Description Protocol)는 WebRTC 연결 전에 주고받는 자기소개서다. "나는 H.264 쓸 수 있어", "나는 보내기만 할 거야(sendonly)", "내 IP 후보들은 이것들이야" 같은 정보가 담긴다.

```
m=video 9 UDP/TLS/RTP/SAVPF 96
a=rtpmap:96 H264/90000          ← H.264 코덱으로 보낼게
a=sendonly                      ← 보내기만 할 거야 (WHIP)
a=ice-ufrag:abc1
a=ice-pwd:secretpassword1234
```

WHIP은 이 SDP를 HTTP POST Body로 그냥 보내는 거였다. 특별한 프로토콜이 추가된 게 아니라, 기존 WebRTC의 SDP 교환을 HTTP로 하도록 방법을 통일한 것이다.

---

# 실제로 어디서 쓰이나

**Cloudflare Stream**이 WHIP·WHEP을 완전 지원한다. OBS에서 스트림 서버 주소에 Cloudflare WHIP URL 입력하면 끝이다. 플러그인 없이 OBS 기본 설정으로 WebRTC 기반 0.5초 지연 방송이 가능하다.

사용처를 보면 공통점이 있다. 지연이 게임의 룰을 바꾸는 서비스들이다.

- **스포츠 베팅**: 5초 지연이면 이미 골 들어간 뒤에 베팅 가능. 부정행위 차단을 위해 0.5초 이하 필수.
- **실시간 경매**: "낙찰됩니다!" 방송이 3초 늦으면 경매가 엉망이 된다.
- **인터랙티브 방송**: 시청자가 방송인 게임을 조종하는 방식. HLS 30초 지연이면 조종 불가다.

일반 라이브 방송(Twitch, 아프리카TV)은 아직 RTMP+HLS 조합을 쓴다. 지연이 5~30초여도 괜찮고, CDN 덕분에 수백만 명을 동시에 소화할 수 있기 때문이다. WebRTC 기반 WHIP·WHEP은 SFU 서버 비용이 높아서 수십만 명 이상 동시 시청에는 적합하지 않다.

![서비스 유형별 지연 허용 기준 비교](/assets/img/posts/whip-whep-streaming-protocols-summary/use-cases-latency.png)

---

# 스트리밍 프로토콜 선택 기준 — 시리즈를 마치며

이 시리즈를 시작할 때 목표는 "스트리밍 아키텍처를 설계할 수 있을 정도로 이해하기"였다. 마지막 단계에서 정리해보니 선택 기준이 꽤 명확해졌다.

**지연 요구사항이 첫 번째 질문이다.**

```
5초 이상 OK, 수백만 명 → RTMP(Ingest) + HLS/DASH(배포) + CDN
1~3초, 수백만 명      → RTMP(Ingest) + LLHLS(배포) + CDN
0.5초 이하, 수만 명   → WHIP(Ingest) + SFU + WHEP(배포)
양방향 실시간          → WebRTC P2P (1:1) 또는 SFU (다자)
```

![스트리밍 프로토콜 진화 타임라인](/assets/img/posts/whip-whep-streaming-protocols-summary/protocol-selection-guide.png)

두 번째는 **Ingest(방송인 → 서버) 구간 선택**이다.

- **RTMP**: 지금 당장 빠르게. OBS 기본값. 20년 표준.
- **SRT**: 해외 송출, 불안정 네트워크. ARQ 재전송 + AES 암호화.
- **WHIP**: 미래 지향. H.265/AV1 지원. 방화벽 통과 쉬움. OBS 30.0+ 기본 탑재.

공부하면서 느낀 건, 프로토콜이 많아 보여도 결국 다 이전 프로토콜의 한계를 해결하려고 만들어진 거라는 점이다. RTMP의 Flash 종속 → HLS, HLS의 지연 → LLHLS, HLS의 코덱 제한 → DASH, TCP의 불안정 → SRT, 표준 없는 WebRTC 연결 → WHIP·WHEP. 역사적 흐름으로 보니까 외울 필요가 없었다.

12단계 쭉 달려왔는데, 솔직히 처음엔 "이걸 다 알아야 하나?" 싶었다. 근데 전부 연결되는 구조라는 걸 알고 나서부터는 생각보다 빠르게 정리됐다. SW마에스트로 프로젝트에서 실시간 스트리밍 기능을 붙이게 된다면, 이 지도가 꽤 쓸모 있을 것 같다.
