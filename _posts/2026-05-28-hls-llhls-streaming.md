---
title: "HLS부터 LLHLS까지 — 스트리밍 지연을 줄이는 방법 (이론)"
date: 2026-05-28 00:00:00 +0900
categories: [백엔드, 스트리밍]
tags: [streaming, hls, llhls]
image:
  path: /assets/img/posts/hls-llhls-streaming/thumbnail.webp
  alt: 6초 세그먼트 3개로 18초가 되는 HLS 지연과 LLHLS의 500ms를 비교한 노트 필기
---

영상 재생 정도면 그냥 라이브러리
하나 붙이면 되겠지 싶었는데, HLS가 뭔지부터 찾아보다가 생각보다 깊은 세계였다.

---

# TCP와 UDP — 스트리밍 전에 알아야 할 것

HLS를 이해하려면 TCP와 UDP 차이부터 짚어야 한다.

**TCP**는 등기 택배다. 보내면 수신 확인을 받고, 분실되면 재전송하고, 순서도 맞춰준다.
신뢰성이 높은 대신 확인하는 과정 때문에 느리다.

**UDP**는 전단지 배포다. 그냥 뿌린다. 누가 받았는지 확인 안 한다. 빠른 대신 몇 개
날아가도 모르고 넘어간다.

![TCP vs UDP 비교](/assets/img/posts/hls-llhls-streaming/tcp-udp-comparison.png)

스트리밍 프로토콜이 어떤 걸 쓰는지 보면 성격이 보인다.

| 프로토콜 | 기반 | 지연 |
|----------|------|------|
| HLS / LLHLS | HTTP → TCP | 2~30초 |
| WebRTC | UDP | < 500ms |
| RTSP | UDP | 낮음 |

HLS는 TCP 기반이다. 신뢰성이 높은 대신 패킷이 손실되면 재전송이 일어나는데,
스트리밍에서 이게 생기면 해당 패킷 기다리는 동안 뒤 패킷들도 전부 대기한다.
버퍼링이 생기는 근본 이유가 여기 있다.

WebRTC가 실시간 통화에 쓰이는 건 UDP라서 패킷 날아가면 그냥 다음으로 넘어가기
때문이다. 순간 화질이 깨져도 지연이 없다.

---

# HLS 기본 구조

HLS (HTTP Live Streaming) — Apple이 2009년에 만든 프로토콜이다.

핵심 아이디어는 단순하다. 영상을 통째로 보내지 말고, 작게 쪼개서 순서대로 보내자.

![HLS 전체 아키텍처](/assets/img/posts/hls-llhls-streaming/hls-architecture.png)

```
원본 영상.mp4
  ↓ (FFmpeg이 쪼갬)
seg_0.ts   (0~6초)
seg_1.ts   (6~12초)
seg_2.ts   (12~18초)
index.m3u8 ← 목차 파일
```

클라이언트는 `index.m3u8` 먼저 받고 거기에 적힌 순서대로 `.ts` 파일을 받아서 재생한다.

`index.m3u8`은 실제로 이렇게 생겼다.

```m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:6
#EXT-X-MEDIA-SEQUENCE:0

#EXTINF:6.000,
seg_0.ts
#EXTINF:6.000,
seg_1.ts
#EXTINF:6.000,
seg_2.ts

#EXT-X-ENDLIST
```

`#EXT-X-ENDLIST`가 있으면 VOD, 없으면 라이브다.

## ABR — 화질 자동 전환

유튜브에서 네트워크 느려지면 자동으로 화질이 떨어지는 게 **ABR (Adaptive Bitrate
Streaming)** 이다. HLS는 마스터 플레이리스트 하나가 여러 화질의 플레이리스트를
가리키는 방식으로 구현한다.

```m3u8
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
360p/index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=2000000,RESOLUTION=1280x720
720p/index.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
1080p/index.m3u8
```

플레이어가 네트워크 상태 보고 알아서 골라서 요청한다. HTTP 기반이라 CDN을 그대로
쓸 수 있는 게 HLS의 큰 장점이다.

---

# 라이브 스트리밍 — 슬라이딩 윈도우

라이브는 VOD랑 다르게 `.m3u8`이 계속 갱신된다. 서버가 새 세그먼트를 만들면
플레이리스트에 추가하고, 오래된 건 빠진다.

![슬라이딩 윈도우 동작 방식](/assets/img/posts/hls-llhls-streaming/sliding-window.png)

```
t=0s   [seg0][seg1][seg2]
t=6s         [seg1][seg2][seg3]
t=12s              [seg2][seg3][seg4]
```

항상 최근 N개만 유지하는 이 방식이 **슬라이딩 윈도우**다. 오래된 세그먼트는 지워도
돼서 CDN 부하도 줄어든다.

클라이언트는 `#EXT-X-TARGETDURATION` 간격마다 플레이리스트를 새로 요청해서 새 세그먼트가
생겼는지 확인한다. 이 폴링 방식이 지연의 원인 중 하나다.

유튜브 라이브에서 뒤로 감기가 되는 DVR 기능은 세그먼트를 지우지 않고 계속 쌓아두는
방식이다. `#EXT-X-PLAYLIST-TYPE:EVENT` 태그 하나로 동작이 달라진다.

---

# HLS 지연이 왜 15~30초인가

파고들기 전까지는 그냥 "느린 거겠지" 했는데, 원인이 명확했다.

```
[원인 1] 세그먼트 단위가 큼
  → 6초짜리가 완성돼야 전달 가능 → 최소 6초 손해

[원인 2] 폴링 방식
  → "새 거 있어?" 물어봐도 없으면 기다렸다가 다시 물어봄
  → 평균 폴링 간격의 절반만큼 추가 지연

[원인 3] 버퍼링 여유분
  → 끊김 없이 보려면 1~2개 세그먼트 미리 받아둠
  → 추가 6~12초
```

![HLS 지연 원인 분석](/assets/img/posts/hls-llhls-streaming/hls-latency.png)

더하면 쉽게 16~23초가 된다. "세그먼트를 짧게 하면 되지 않냐"는 답이 아닌 게,
파일 수가 폭증해서 CDN 요청 수가 감당 안 된다.

---

# LLHLS 등장

WWDC 2019에서 Apple이 LLHLS (Low Latency HLS) 를 발표했다.

목표는 두 가지였다. 기존 CDN 인프라 그대로 쓰고, 지연을 2~5초로 줄이자.
"CDN 그대로"가 핵심이다. WebRTC처럼 새 인프라를 깔 필요 없이 HTTP 서버와 CDN
위에서 동작해야 했다.

설계가 영리한 건 하위 호환성이다. LLHLS 플레이리스트를 구버전 클라이언트가 받으면
`#EXT-X-PART` 같은 모르는 태그는 그냥 무시한다. 기존 `.ts` 세그먼트는 그대로 있으니까
구버전은 일반 HLS처럼 동작하고, 최신 클라이언트만 빠른 재생을 한다.

---

# LLHLS 핵심 기술 세 가지

## ① Partial Segments

6초짜리 세그먼트를 0.5초 조각으로 쪼개서 완성되는 족족 바로 전달한다.

![Partial Segments — HLS vs LLHLS 비교](/assets/img/posts/hls-llhls-streaming/partial-segments.png)

```
일반 HLS:
├────────────────────────┤  6초 기다려야 받음

LLHLS:
├──┤├──┤├──┤├──┤├──┤├──┤  0.5초마다 받을 수 있음
p0  p1  p2  p3  p4  p5
```

플레이리스트에서 이렇게 보인다.

```m3u8
#EXTINF:6.000,
seg_0.ts

#EXT-X-PART:DURATION=0.5,URI="p_1_0.mp4",INDEPENDENT=YES
#EXT-X-PART:DURATION=0.5,URI="p_1_1.mp4"
#EXT-X-PART:DURATION=0.5,URI="p_1_2.mp4"
```

`INDEPENDENT=YES`는 이 조각만 있어도 독립적으로 디코딩 가능하다는 뜻이다. 키프레임이
포함된 조각이다. 기존 `.ts` 대신 fMP4를 쓰는 이유도 여기 있다. fMP4는 원래 조각
단위(fragment)로 설계돼 있어서 Partial Segment에 딱 맞는다.

## ② Blocking Playlist Reload

폴링 방식을 바꾼다. 클라이언트가 이렇게 요청한다.

```
GET /index.m3u8?_HLS_msn=1&_HLS_part=4
```

"seg_1의 4번째 Partial이 나올 때까지 기다려." 서버는 이 요청을 블로킹해뒀다가
p_1_4가 완성되는 순간 응답한다. 클라이언트 입장에선 "요청하자마자 바로 왔네?"처럼
느껴진다.

![일반 폴링 vs Blocking Playlist Reload](/assets/img/posts/hls-llhls-streaming/blocking-reload.png)

```
일반 폴링: 요청 → 없음 → 대기 → 요청 → 없음 → 대기 → 요청 → 있음
블로킹:    요청 ─────────────────────────────────────────▶ 있음
```

HTTP/2가 여기서 중요하다. 블로킹 요청을 여러 개 동시에 보내야 할 때 HTTP/1.1은
순차 처리지만, HTTP/2는 하나의 연결로 동시에 처리(멀티플렉싱)할 수 있다.

## ③ Preload Hints

응답할 때 다음에 올 파일이 뭔지 미리 알려준다.

```m3u8
#EXT-X-PART:DURATION=0.5,URI="p_1_4.mp4"
#EXT-X-PRELOAD-HINT:TYPE=PART,URI="p_1_5.mp4"  ← 다음에 이게 나올 거야
```

클라이언트는 p_1_4 재생하면서 p_1_5를 미리 블로킹 요청해둔다. p_1_5가 완성되면
바로 응답이 온다. 파이프라인이 생기는 거다.

```
재생:  p0 → p1 → p2 → p3 → p4 → p5
다운:   p1   p2   p3   p4   p5   p6
        └ 항상 한 발 앞서 미리 받아둠
```

---

# CDN 설정 — 놓치면 다 무너지는 부분

LLHLS 붙여놓고 지연이 줄지 않는다면 이게 원인일 가능성이 높다.

블로킹 요청은 서버가 응답을 붙잡고 있다가 파일 완성되면 보내주는 방식인데, CDN이
중간에서 "아까 없다고 했잖아" 하고 이전 응답을 캐싱해서 돌려주면 클라이언트는
영원히 파일을 못 받는다.

![CDN 캐싱 문제와 바이패스 설정](/assets/img/posts/hls-llhls-streaming/cdn-cache-bypass.png)

`_HLS_msn`, `_HLS_part` 파라미터가 있는 요청은 캐시 바이패스하도록 CDN 설정이 필수다.

```nginx
if ($arg__HLS_msn != "") {
    set $no_cache 1;
}
if ($arg__HLS_part != "") {
    set $no_cache 1;
}
proxy_cache_bypass $no_cache;
```

반면 `.mp4` 세그먼트 파일들은 한 번 생성되면 바뀌지 않으니 캐싱해도 된다.

---

# 느낀점

솔직히 처음엔 스트리밍이 이렇게 복잡한 줄 몰랐다. "영상 파일 서빙하면 되는 거 아냐?"
수준이었는데, HLS 하나만 제대로 파봐도 TCP/UDP, CDN, 세그먼트 설계, HTTP/2까지
전부 연결돼 있었다.

특히 LLHLS가 기존 인프라를 뜯지 않고 지연을 줄인 방식이 인상적이었다. 세그먼트를
더 작게 쪼개는 것만으로 해결하려 하면 CDN 부하 문제가 생기고, 그걸 Blocking
Reload로 보완하고, 거기서 생기는 대기 시간을 Preload Hints로 없애는 구조가 꽤
정교했다.

CDN 캐싱 문제는 모르고 지나쳤으면 운영 나가서 삽질했을 것 같다. 이론만 알고 넘어갔으면
몰랐을 포인트라, 다음에 실제 서버 구성해보면서 직접 확인해볼 예정이다.
