---
title: "팩토리 패턴 + 전략 패턴 — 만드는 책임과 동작하는 책임을 나누다"
date: 2026-06-01 00:00:00 +0900
categories: [책 Log, 면접을 위한 CS 전공지식 노트, 팩토리·전략 패턴]
tags: [design-pattern, factory-pattern, strategy-pattern, java, spring, cs]
image:
  path: /assets/img/posts/factory-strategy-pattern/factory-pattern.png
  alt: "팩토리 패턴 구조"
---

싱글톤 패턴에 이어 팩토리 패턴과 전략 패턴을 공부했다. 이 두 패턴은
따로 배웠는데 쓰다 보면 같이 쓰는 경우가 많다는 게 흥미로웠다.

---

# 팩토리 패턴 — 객체 생성을 공장에 맡기다

결제 시스템을 만든다고 해보자. 카카오페이, 네이버페이, 신용카드 세 가지 방법이 있다.

```java
if (type.equals("kakao")) {
    payment = new KakaoPayment();
} else if (type.equals("naver")) {
    payment = new NaverPayment();
} else if (type.equals("card")) {
    payment = new CardPayment();
}
```

나중에 토스페이가 추가되면? 이 `if-else`가 코드 여러 곳에 퍼져있으면
전부 찾아서 수정해야 한다. 팩토리 패턴은 이 문제를 한 곳에 모아서 해결한다.

```java
interface Payment {
    void pay(int amount);
}

class KakaoPayment implements Payment {
    public void pay(int amount) {
        System.out.println("카카오페이로 " + amount + "원 결제");
    }
}

class NaverPayment implements Payment {
    public void pay(int amount) {
        System.out.println("네이버페이로 " + amount + "원 결제");
    }
}
```

```java
// 객체 생성 코드를 여기 한 곳에만
class PaymentFactory {
    public static Payment create(String type) {
        switch (type) {
            case "kakao": return new KakaoPayment();
            case "naver": return new NaverPayment();
            case "card":  return new CardPayment();
            default: throw new IllegalArgumentException("모르는 결제 방법: " + type);
        }
    }
}
```

```java
Payment payment = PaymentFactory.create("kakao");
payment.pay(10000);
```

새 결제 수단이 생겨도 `PaymentFactory`만 수정하면 된다. 사용하는 쪽은 건드릴 필요가 없다.

팩토리 패턴의 핵심은 "어떻게 만드는지"는 공장이 알고, 사용하는 쪽은
"뭘 원하는지"만 말하면 된다는 거다.

---

# 추상 팩토리 — 공장도 여러 종류일 때

팩토리 메서드가 단일 객체를 만드는 공장이라면, 추상 팩토리는 관련된
객체들의 묶음을 만드는 공장들을 추상화한 거다.

UI 테마를 예로 들면 이렇다.

```java
interface UIFactory {
    Button createButton();
    Background createBackground();
}

class DarkUIFactory implements UIFactory {
    public Button createButton() { return new DarkButton(); }
    public Background createBackground() { return new DarkBackground(); }
}

class LightUIFactory implements UIFactory {
    public Button createButton() { return new LightButton(); }
    public Background createBackground() { return new LightBackground(); }
}
```

```java
UIFactory factory = new DarkUIFactory();
Button btn = factory.createButton();       // 다크 버튼
Background bg = factory.createBackground(); // 다크 배경
```

공장만 바꾸면 관련된 모든 객체가 같은 테마로 바뀐다. 팩토리 메서드는
단일 제품, 추상 팩토리는 제품군이라고 외워두면 면접에서 차이를 물었을 때 쓸 수 있다.

---

# 전략 패턴 — 동작을 바꾸고 싶을 때

네비게이션 앱을 만든다고 하자. 자동차, 도보, 대중교통 세 가지 경로가 있다.

```java
if (mode.equals("car")) {
    // 자동차 경로 로직 100줄
} else if (mode.equals("walk")) {
    // 도보 경로 로직 100줄
} else if (mode.equals("transit")) {
    // 대중교통 로직 100줄
}
```

자전거 경로가 추가되면 이 클래스가 점점 뚱뚱해진다. 전략 패턴은 각
동작을 별도 클래스로 분리해서 실행 중에도 교체할 수 있게 한다.

```java
interface RouteStrategy {
    void findRoute(String start, String end);
}

class CarStrategy implements RouteStrategy {
    public void findRoute(String start, String end) {
        System.out.println(start + " → " + end + ": 고속도로 이용");
    }
}

class WalkStrategy implements RouteStrategy {
    public void findRoute(String start, String end) {
        System.out.println(start + " → " + end + ": 골목길 포함");
    }
}
```

```java
class Navigator {
    private RouteStrategy strategy;

    public void setStrategy(RouteStrategy strategy) {
        this.strategy = strategy;
    }

    public void navigate(String start, String end) {
        strategy.findRoute(start, end);
    }
}
```

```java
Navigator nav = new Navigator();

nav.setStrategy(new CarStrategy());
nav.navigate("서울", "부산");  // 고속도로 이용

nav.setStrategy(new WalkStrategy());
nav.navigate("강남역", "역삼역");  // 골목길 포함
```

`Navigator`는 경로 찾는 방법을 직접 알지 않아도 된다. 전략 객체에게
위임하는 것뿐이다. 새 이동 수단이 생겨도 새 전략 클래스를 추가하기만
하면 되고 `Navigator`는 손댈 필요가 없다.

![전략 패턴 구조](/assets/img/posts/factory-strategy-pattern/strategy-pattern.png)

---

# 두 패턴을 같이 쓰면

실무에서 이 두 패턴은 자주 함께 쓰인다. 팩토리가 전략 객체를 만들어주는 방식이다.

```java
class RouteStrategyFactory {
    public static RouteStrategy create(String mode) {
        switch (mode) {
            case "car":     return new CarStrategy();
            case "walk":    return new WalkStrategy();
            case "transit": return new TransitStrategy();
            default: throw new IllegalArgumentException("모르는 이동 방법");
        }
    }
}

Navigator nav = new Navigator();
nav.setStrategy(RouteStrategyFactory.create("car"));
nav.navigate("서울", "부산");
```

팩토리가 "어떤 전략 객체를 만들지" 결정하고, 전략이 "어떻게 동작할지"
결정한다. 역할이 깔끔하게 나뉜다.

---

# 스프링에서 전략 패턴

스프링 시큐리티의 인증 방식이 대표적인 예다.

```java
@Bean
public AuthenticationManager authManager() {
    return new ProviderManager(
        List.of(new JwtAuthenticationProvider(),
                new OAuth2AuthenticationProvider())
    );
}
```

JWT 인증에서 OAuth2 인증으로 바꾸거나 두 가지를 같이 쓰고 싶을 때,
핵심 코드는 건드리지 않고 전략만 교체하거나 추가하면 된다.

---

다음엔 옵저버 패턴과 프록시 패턴을 정리할 예정이다.
이 두 패턴은 이벤트 처리와 접근 제어 쪽에서 많이 만나게 된다.
