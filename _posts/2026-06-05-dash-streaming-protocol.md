---
title: "DASH — 유튜브가 LLHLS 대신 선택한 이유"
date: 2026-06-05 10:00:00 +0900
categories: [백엔드, 스트리밍]
tags: [dash, hls, llhls, streaming, mpd, abr, codec, youtube, vp9]
image:
  path: /assets/img/posts/dash-streaming-protocol/thumbnail.webp
  alt: HLS의 단일 코덱과 DASH의 여러 코덱 조합을 화이트보드에 비교한 그림
---

지난번에 HLS랑 LLHLS를 정리했는데, 쓰면서 계속 걸리는 게 있었다.

치지직은 LLHLS를 쓴다고 배웠는데, 그럼 유튜브는? 유튜브도 같은 방식으로 영상 쪼개서 보내는 거 맞는데, 확인해보니 `.mpd` 라는 파일이 나왔다. HLS랑 다른 방식이었다. 이게 DASH다.

---

# DASH가 왜 생겼나

HLS는 애플이 2009년에 만들었다. 처음엔 라이선스 조건이 있었고, 구글이나 넷플릭스 입장에선 애플한테 의존하는 게 껄끄러웠을 거다.

그래서 2012년에 구글, 넷플릭스, 마이크로소프트 등이 모여서 **ISO 국제 표준**으로 DASH를 만들었다. 공식 명칭은 `MPEG-DASH`고, 로열티 없이 누구나 쓸 수 있다.

출발점부터 목적이 달랐다. HLS는 아이폰에서 잘 돌아가게, DASH는 특정 회사에 종속되지 않게.

실제로 유튜브는 `.mpd` 파일을 그대로 내려주는 건 아니다. 유튜브 플레이어 내부에서 자체 JavaScript로 DASH 방식을 구현한다. 크롬 개발자 도구 Network 탭을 열어보면 영상 요청(`mime=video/mp4`)이랑 오디오 요청(`mime=audio/mp4`)이 완전히 분리돼서 따로 날아오는 걸 확인할 수 있다. "표준 MPEG-DASH 그대로"라기보다는 "DASH 방식을 유튜브가 직접 구현"한 것에 가깝다.

---

# MPD 파일 — DASH의 목차

HLS는 `.m3u8` 파일로 "이 영상엔 이런 화질들 있어요"를 알려준다. DASH는 `.mpd` 파일을 쓴다.

```xml
<MPD>
  <Period>
    <AdaptationSet mimeType="video/mp4">
      <Representation bandwidth="4000000" width="1920" height="1080">
        <SegmentTemplate media="1080p_$Number$.mp4" duration="4"/>
      </Representation>
      <Representation bandwidth="2000000" width="1280" height="720">
        <SegmentTemplate media="720p_$Number$.mp4" duration="4"/>
      </Representation>
    </AdaptationSet>

    <AdaptationSet mimeType="audio/mp4">
      <Representation bandwidth="128000">
        <SegmentTemplate media="audio_$Number$.mp4" duration="4"/>
      </Representation>
    </AdaptationSet>
  </Period>
</MPD>
```

HLS `.m3u8`이 메모장 같은 텍스트라면, MPD는 XML이라 처음엔 읽기 복잡하다. 근데 자세히 보면 구조가 명확하다. `AdaptationSet`이 화질 묶음, `Representation`이 화질 하나하나다.

그리고 여기서 눈에 띄는 게 있다. 영상 `AdaptationSet`이랑 오디오 `AdaptationSet`이 완전히 분리돼 있다는 거다.

![DASH 구조 — MPD에서 Segment까지](/assets/img/posts/dash-streaming-protocol/dash-structure.png)

---

# 영상·오디오가 따로 있다는 게 왜 중요한가

HLS는 보통 영상이랑 오디오가 같은 파일에 묶인다. `720p_ko.ts` 하나에 720p 영상이랑 한국어 오디오가 같이 들어있다.

DASH는 완전히 다르다. 영상 파일은 하나고, 오디오 파일이 언어별로 따로 있다.

```
HLS 방식 (언어마다 영상+오디오 세트)
  720p_ko.ts  = 영상 + 한국어
  720p_en.ts  = 영상 + 영어    ← 영상 데이터 중복
  720p_ja.ts  = 영상 + 일본어  ← 또 중복

DASH 방식 (영상 1개, 오디오만 언어별)
  720p.mp4       = 영상만
  audio_ko.mp4   = 한국어
  audio_en.mp4   = 영어
  audio_ja.mp4   = 일본어
```

유튜브가 80개 언어를 지원한다고 생각해보면, HLS 방식으로는 영상 파일이 80배가 필요하다. DASH는 영상 파일은 그대로 1개고, 오디오만 80개 추가하면 된다.

![HLS vs DASH 파일 구조 비교](/assets/img/posts/dash-streaming-protocol/hls-vs-dash-comparison.png)

처음 이 구조를 봤을 때 "왜 이렇게 복잡하게 만들었지?" 싶었는데, 규모가 커지면 이게 무조건 맞는 선택이더라. 스토리지 비용 차이가 상상 이상으로 크다.

---

# 코덱 — VP9가 DASH를 택한 또 다른 이유

코덱은 영상을 어떻게 압축하느냐를 결정한다. 같은 화질이라도 코덱에 따라 파일 크기가 크게 달라진다.

```
H.264  → 100 기준
VP9    →  70 (30% 작아짐, 구글이 만든 것)
AV1    →  45 (55% 작아짐, VP9 후계자)
```

문제는 아이폰이 VP9을 지원하지 않는다는 거다. HLS를 쓰면 iOS 호환성 때문에 VP9을 못 쓰고 H.264에 발목이 잡힌다.

유튜브의 DASH 방식 구현은 이 제약이 없다. 유튜브가 크롬에서 VP9으로 영상을 서비스하는 이유다. 같은 화질에 파일이 30% 작아지면 CDN 전송 비용도 그만큼 줄어든다. 유튜브 규모에서 이 차이는 수백억 단위다.

---

# 그럼 치지직은 왜 LLHLS를 택했나

결국 목적이 다르다.

치지직은 **라이브 방송 플랫폼**이다. 스트리머가 말하면 시청자 채팅에 1~2초 내로 반응이 와야 자연스러운 대화가 된다. DASH의 2~5초 지연은 이걸 망친다.

거기다 한국 아이폰 사용자가 약 30%다. iOS Safari는 HLS/LLHLS만 네이티브 지원한다. DASH를 쓰면 iOS를 위한 별도 구현이 필요해진다.

반면 유튜브는 지연보다 **규모와 효율**이 핵심이다. 전 세계 수십억 명이 쓰는 플랫폼에서 스토리지 30%를 아끼는 게 1초 지연 줄이는 것보다 훨씬 중요하다.

같은 "영상 쪼개기"인데 무엇을 최우선에 두느냐에 따라 선택이 완전히 달라진 거다.

---

# 정리

|  | LLHLS (치지직) | DASH (유튜브) |
|--|--|--|
| 지연 | 1~2초 | 2~5초 |
| 영상·오디오 | 묶음 | 완전 분리 |
| VP9/AV1 코덱 | ❌ | ✅ |
| iOS 기본 지원 | ✅ | ❌ |
| 주 목적 | 라이브 저지연 | 대규모 효율 |

우리 프로젝트는 치지직 스트림을 받아오는 거라 DASH를 직접 구현할 일은 없다. 근데 면접에서 "스트리밍 시스템 설계해보세요"라는 질문이 나오면 이 차이를 설명할 수 있어야 한다. 라이브 방송이면 LLHLS, VOD 대규모 플랫폼이면 DASH — 목적부터 다르다는 걸 이해하면 흔들리지 않는다.

다음은 RTMP다. Flash 시대의 인제스트 프로토콜인데, 아직도 OBS에서 방송 송출할 때 기본으로 쓰인다. 이유가 있을 것 같다.
