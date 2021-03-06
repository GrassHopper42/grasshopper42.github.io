---
title: "[오삽내]private LocalDateTime 에러 해결..."
description: "LocalDateTime을 private 변수로 설정했을 때 생긴 오류 해결"
date: 2022-07-07
update: 2022-07-07
tags:
  - Java
  - Spring
  - Mongodb
  - TroubleShooting
---

## 오삽내

> 오늘의 삽질 내용

서비스 백엔드를 `NestJS`에서 `Spring`으로 넘어가면서 `Java`까지 통으로 새로배우면서 뚝딱거리려니 힘들다. 그나마 `TypeScript`로 타입지정하고 `NestJS`에서 데코레이션 달아보고 `JavaScript`와 유사한 부분도 있어서 한결 수월하다는데 위안을 삼고 있다.
오늘은 몇시간동안 낑낑거렸는데 허무하게 끝난 Trouble Shooting이 구글링할때 나오지 않은게 아쉬워 짧게나마 삽질 내용을 남겨보려한다.  
`Spring Data Mongodb`로 DB연결하고 간단한 CRUD코드 몇개 만들어서 테스트를 돌려보는데 전혀 예상치도 못결 부분에서 에러가 터졌다.
`createdAt`에 `Date`타입을 적용하기 위해 선언한 `LocalDateTime`타입에서 오류가 나서 클래스가 생성되지 못하고 꺼져버리는 것이었다.
몇시간동안 구글링을 했는데 답이나오지 않아 스택오버플로우에 질문을 올리고 혼자 깨달음을 얻어 허무하게 끝나버렸다.

## 문제

아래는 `LocalDateTime`타입이 사용된 `User`모델의 일부이다.

```java
// User.java
import java.time.LocalDateTime;

@Data
@NoArgsConstructor
@Document("User")
public class User {

  @Id
  private ObjectId id;

  @CreatedDate
  private LocalDateTime createdAt;

  @LastModifiedDate
  private LocalDateTime updatedAt;

  private String username;

  // ...
}
```

이 상태로 테스트를 돌렸더니 아래의 에러를 볼 수 있었다.

```sh
Caused by: java.lang.reflect.InaccessibleObjectException: Unable to make private java.time.LocalDateTime(java.time.LocalDate,java.time.LocalTime) accessible: module java.base does not "opens java.time" to unnamed module
```

## 해결

### 1. VmArgs 설정

이 에러 문구를 가지고 이렇게 저렇게 조합해보면서 구글링을 하다가 한 [스택오버플로우 답변](https://stackoverflow.com/questions/70412805/what-does-this-error-mean-java-lang-reflect-inaccessibleobjectexception-unable)을 발견했다.

`start-argument`로 `--add-opens java.base/java.time=ALL-UNNAMED`를 입력하라는 글이었는데, 이게 도통 뭔소리고 VSC에서 어떻게 입력해줘야 할지도 모르겠어서 넘겨버렸다.
한참 낑낑대다가 스택오버플로우 질문을 남겼더니 위 답변의 링크를 주면서 이걸로 해결이 안되냐는 코멘트가 달렸다. 이게 진짜 되는건가 하고 VSC로 vm 설정하는 법을 찾아서 입력해봤다.

VSC에서 위와 같은 args를 입력하기 위해서는 상단 **Run**탭에서 **Open Configuration | Add Configuration**을 누르거나 프로젝트 루트에서 `.vscode` 폴더에 있는 `launch.json`에 `vmArgs`을 키로 추가한 뒤 값에는 `-add-opens java.base/...`을 입력해주면 된다.

```json
{
  // Use IntelliSense to learn about possible attributes.
  // Hover to view descriptions of existing attributes.
  // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
  "version": "0.2.0",
  "configurations": [
    {
      "type": "java",
      "name": "Launch Current File",
      "request": "launch",
      "mainClass": "${file}",
      "vmArgs": "--add-opens java.base/java.time=ALL-UNNAMED"
    },
    {
      "type": "java",
      "name": "Launch ApiApplication",
      "request": "launch",
      "mainClass": "art.unhide.api.ApiApplication",
      "projectName": "api",
      "vmArgs": "--add-opens java.base/java.time=ALL-UNNAMED"
    }
  ]
}
```

> IDE 짱짱맨... vim은 그냥 VSC 플러그인으로만 쓰는걸로...

하지만 여전히 테스트에서는 같은 에러가 발생한다. 테스트에서 VmArgs를 입력하려면 `.vscode` 폴더의 `setting.json`파일에서 아래의 코드를 추가해준다.

```json
"java.test.config": {
  "vmArgs": ["--add-opens", "java.base/java.time=ALL-UNNAMED"]
}
```

### 2. package 변경

위의 방법으로 작동시키면 정상적으로 작동하는데 로그에서 아래와 같은 경고문구를 볼 수 있었다.
처음엔 테스트에서 별도로 VmArgs를 입력하는 방법을 몰라 이 방법을 썼었다.

```sh
WARN 17980 --- [  restartedMain] o.s.data.convert.CustomConversions       : Registering converter from class java.time.LocalDateTime to class org.joda.time.LocalDateTime as reading converter although it doesn't convert from a store-supported type; You might want to check your annotation setup at the converter implementation
```

시스템에서 `java.time.LocalDateTime`을 `org.joda.time.LocalDateTime`으로 convert 시켜같는 것 같다. 그래서 convert할 필요 없게 package를 바꿔줬다. 아래는 바뀐 코드이다.

```java
// User.java
import org.joda.time.LocalDateTime; // 바뀐부분

@Data
@NoArgsConstructor
@Document("User")
public class User {

  @Id
  private ObjectId id;

  @CreatedDate
  private LocalDateTime createdAt;

  @LastModifiedDate
  private LocalDateTime updatedAt;

  private String username;

  // ...
}
```

이 방법을 쓰면 별다른 설정 없이도 문제없이 동작한다.

> 그렇지만 로그는 여전히 `java.time`을 `org.joda.time`으로 convert한다... 찾아보니 사실 관련 없었던것... 그냥 뒷걸음질 치다 쥐 잡은 것 같다.

## 이유

는 잘 모르겠다... 구글링하다보니 `java.time` 뿐만 아니라 `java.util`같은 다른 자바 내장 패키지들도 이따금 private으로 선언하면 비슷한 에러를 띄우는것 같았다.
유사한 문제에 대해서는 로그를 참고해 위의 VmArgs를 조금 바꿔서 추가해주면 해결될 듯 하다.  
재밌는건 모델만 만들어 뒀을때는 문제가 발생하지 않는데 Repository와 Service, Controller를 만들어 주입시키면 문제가 생긴다.  
`java.time`의 `ZonedDateTime`을 사용하는 경우도 있는것 같았는데 이건 자체적으로 convert가 되지 않는 것 같다.
[이 글](https://www.baeldung.com/spring-data-mongodb-zoneddatetime)에서 `ZonedDateTime`의 Custom Converter를 만들어주는 방법을 상세하게 알려주고 있다.
LocalDateTime과 ZonedDateTime의 차이를 비교해보고 더 유용하겠다 싶으면 적용해봐야겠다.

## 참고자료

- [StackOverflow](https://stackoverflow.com/questions/70412805/what-does-this-error-mean-java-lang-reflect-inaccessibleobjectexception-unable)
- [ZonedDateTime Converter](https://www.baeldung.com/spring-data-mongodb-zoneddatetime)
- [NaverD2](https://d2.naver.com/helloworld/645609)
- [Spring-boot Github issue](https://github.com/spring-projects/spring-boot/issues/18817)
