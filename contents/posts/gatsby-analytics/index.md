---
title: "[Gatsby]블로그에 Google Analytics 연결"
description: "Gatsby블로그에 Google Analytics 연결"
date: 2022-06-09
update: 2022-06-09
tags:
  - Gatsby
  - blog
  - google analytics
---

## 블로그 방문 통계 분석

블로그를 이왕시작한거 방문수도 체크해보기로 했다.
여기저기 기웃거리면서 따라해봤는데 github page 배포 딜레이때문인지 측정이 안돼서 당황했다.
이것저것 만지다보니 되긴 했는데 정확히 어떤부분인지는 모르겠다.
일단 현재 상태 기준으로 작성하기로 했다.(2022.06.09 기준)

## 1. **Google Aanlytics** 설정

먼저 [**Google Analytics** 홈페이지](http://analytics.google.com/)에 들어가 계정을 만들고 속성을 설정해준다.

![계정 생성](1.JPG)

계정 이름을 입력하고 계정 데이터 공유 설정을 해준다.
처음부터 Google제품 및 서비스를 제외한 나머지 3항목이 체크되어있다.
나는 계정 이름은 grasshopper42, 체크박스는 처음 그대로 진행했다.

![속성 설정](2.JPG)
계정을 만들고 다음을 누르면 이제 데이터 측정을 위한 속성설정을 해야한다.
개발블로그 추적을 위한 속성이기 때문에 속성 이름은 dev blog로 했다.
시간대와 통화는 기본으로 미국과 달러로 기본설정되어있기 때문에 각각 대한민국과 원으로 바꿔준다.

![비즈니스 정보](3.JPG)
비즈니스 정보는 그렇게 중요할것같지 않아서 대충 설정했다.

![플랫폼 선택](4.JPG)
이제 플랫폼을 선택해야하는데 블로그를 추적할 계획이니까 **웹**을 선택한다.

![데이터 스트림 설정](5.JPG)
데이터를 스트리밍할 블로그 정보를 입력해준다.
나는 grasshopper42.github.io/ 와 Dev blog라고 입력했다.
아래 향상된 측정 부분에 측정할 데이터를 선택할 수 있는데 기본 그대로 뒀다.

여기까지 설정을 마쳤으면 웹 스트림 세부정보라는 창이 뜨면서 **측정ID**값을 얻을 수 있다.

## 2. gatsby-plugin-gtag

이제 블로그와 **Google Analytics**를 연결해주기 위해 **Gatsby**에 **plugin**을 설치해준다

```bash
npm install gatsby-plugin-gtag
```

플러그인 설치가 완료되었으면 프로젝트 루트 폴더의 `gatsby-config.js`를 열어서 `plugin`을 추가해준다.

```js
// ~/gatsby-config.js

// ...
module.exports = {
  // ...other options
  plugins: [
    {
      resolve: `gatsby-plugin-gtag`,
      options: {
        // Google Analytics에서 얻은 측정ID를 입력
        trackingId: `측정ID`,
        // gtag tracking script를 배포될 html의 head에 넣을지 선택
        head: true,
        // IP 익명화 선택
        anonymize: true,
      },
    },
    // ...other plugins
  ],
}
```

이제 블로그를 다시 배포하고 블로그에 방문한 다음 Google Analytics 대쉬보드를 보면 사용자 1이 증가하는 모습을 볼 수 있다.
`head`를 `true`로 설정했다면 블로그에서 개발자도구를 열어 `<head>`에 다음과 같은 코드가 있는지 보면 제대로 입력이 됐는지 알 수 있다.

```html
<script async src="https://www.googletagmanager.com/gtag/js?id=측정ID"></script>
<script>
  //... trackingId, options
</script>
```

## 참고자료

- <https://www.gatsbyjs.com/plugins/gatsby-plugin-gtag/#gatsby-plugin-gtag>
- <https://inasie.github.io/it%EC%9D%BC%EB%B0%98/1/>
- <https://janeljs.github.io/blog/google-analytics/>
- <https://devfoxstar.github.io/web/gatsby-ga-tracking/>
- <https://blog-lino.dev/trouble-shooting/ga4-issues/>
