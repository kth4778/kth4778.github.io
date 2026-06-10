---
title: "싱글톤 패턴 — 왜 인스턴스는 딱 하나여야 할까"
date: 2026-05-31 00:00:00 +0900
categories: [책 Log, 면접을 위한 CS 전공지식 노트, 싱글톤 패턴]
tags: [design-pattern, singleton, java, spring, cs]
image:
  path: /assets/img/posts/singleton-pattern/singleton-structure.png
  alt: "싱글톤 패턴 구조"
---

CS 전공지식 노트를 읽기 시작했다. 1장 첫 번째 주제가 싱글톤 패턴이었는데, 사실 이름은 많이 들어봤지만 제대로 설명해보라고 하면 자신 없었다.

---

# 하나만 있어야 하는 것들

게임에서 소리를 담당하는 객체가 두 개라면 어떻게 될까. 볼륨을 낮춰도 한쪽은 여전히 시끄럽다. DB 커넥션 풀이 두 개라면 각자 따로 연결을 맺으니 비용이 두 배다.

"앱 전체에서 딱 하나만 있어야 하는 것"에 쓰는 게 싱글톤 패턴이다.

구조는 단순하다. 세 가지만 기억하면 된다. 생성자를 `private`으로 막아서 외부에서 `new`를 못 하게 하고, 클래스 내부에 자기 자신을 `static` 변수로 들고 있고, `getInstance()` 메서드 하나로만 접근하게 한다.

```java
public class Singleton {
    private static Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

`Singleton.getInstance()`를 열 번 호출해도 항상 같은 객체가 반환된다. 생각보다 원리 자체는 단순했다.

![싱글톤 패턴 클래스 구조](/assets/img/posts/singleton-pattern/singleton-structure.png)

---

# 멀티스레드에서 터지는 이유

근데 위 코드는 멀티스레드 환경에서 문제가 생긴다.

두 스레드가 동시에 `instance == null`을 체크하면, 둘 다 null이라 판단하고 각자 `new Singleton()`을 실행한다. 인스턴스가 두 개 만들어지는 거다.

그래서 나온 게 Double-Checked Locking이다.

```java
public class Singleton {
    private static volatile Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

`synchronized`로 잠금을 걸고, 잠금 안에서 한 번 더 체크한다. `volatile`은 CPU 캐시가 아닌 메인 메모리에서 직접 읽도록 강제해서 스레드 간 값 불일치를 막는다. 이걸 빠뜨리면 여전히 문제가 생길 수 있다는 게 처음엔 이해가 잘 안 됐는데, CPU가 변수값을 스레드별 캐시에 따로 저장한다는 걸 알고 나서야 납득이 됐다.

![Double-Checked Locking 흐름](/assets/img/posts/singleton-pattern/double-checked-locking.png)

---

# 제일 깔끔한 방법은 Enum

```java
public enum Singleton {
    INSTANCE;

    public void doSomething() {
        System.out.println("싱글톤 동작");
    }
}
```

Enum은 JVM이 인스턴스를 딱 하나만 보장해준다. 스레드 안전하고, 직렬화 문제도 없고, 코드도 짧다. 실무에서 자주 쓰이는 이유가 있었다.

---

# 스프링에서는 그냥 빈이 싱글톤이다

```java
@Service
public class UserService {
    // 스프링 컨테이너가 이걸 딱 하나만 만들어서 관리해줌
}
```

`@Service`, `@Repository`, `@Component` 붙이면 스프링이 알아서 싱글톤으로 관리한다. 직접 짤 일은 거의 없고, "스프링 빈이 왜 기본적으로 하나만 만들어지냐" 물으면 싱글톤 패턴 얘기를 꺼내면 된다.

---

# 단점도 알아야 한다

싱글톤이 편리하지만 욕도 많이 먹는다.

전역 상태를 만들기 때문에 테스트가 어렵다. 테스트A에서 값을 바꾸면 테스트B에도 영향을 준다. 어떤 클래스가 싱글톤을 코드 깊은 곳에서 꺼내 쓰면 겉에서는 그 의존 관계가 보이지 않는다. "싱글톤 안티패턴"이라고 부르는 사람이 있을 정도다. 실무에서 DI 프레임워크를 선호하는 이유가 여기 있다.

---

다음엔 팩토리 패턴과 전략 패턴을 정리할 예정이다. 이 두 패턴은 같이 쓰면 꽤 강력한 조합이다.
