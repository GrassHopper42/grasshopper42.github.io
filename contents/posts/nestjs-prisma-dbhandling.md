---
title: "[NestJS]NestJS에서 Prisma 사용하기"
description: "NestJS에서 기본적인 Prisma사용과 test를 위한 PrismaClient 추가 DB연결"
date: 2022-06-08
update: 2022-06-08
tags:
  - NestJS
  - Prisma
  - MongoDB
  - Jest
---

## NestJS + Prisma

처음 백엔드를 구축할때 **Apollo + GraphQL**을 사용하기로 하면서 쓰기 시작한 **Prisma**를 **NestJS + RestAPI**로 migration하면서 **NestJS**와 많이 쓰는 **TypeORM**보다 익숙하고 편리해서 가져왔는데 막상 쓰려니 부족한 자료에 지금이라도 **TypeORM**으로 넘어가야하나 하는 고민을 수없이 했다.
가장 큰 문제는 **OOP**환경에 대한 **Prisma**의 문서부족이었다.
기본적인 세팅에 대한 부분은 **Prisma**와 **NestJS**공식문서에 잘 나와있었지만 조금만 깊어지면 문서가 영문으로조차 없어서 며칠동안 삽질을 했다.
대체적으로 **OOP**와 **테스트**에 대한 내 이해부족때문이었다.
NestJS시작부터 글을 적을 생각이었지만 기억이 날아갈까봐 우선 Prisma적용에 대한 글부터 남긴다.

> 여러모로 부족해 설명에 오류나 누락이 있을 수 있습니다. 혹시나 이 글을 보신다면 많은 피드백 부탁드립니다.

### 1. NestJS에 PrismaClient 생성

1. 우선 **NestJS**와 **Prisma** 기본세팅이 돼있다는걸 가정하고 바로 **PrismaClient**생성으로 넘어간다. `nest-cli`를 이용해 `prisma`모듈과 서비스를 생성해준다.

   ```bash
   nest g mo prisma && nest g s prisma
   ```

2. `prisma.service.ts`에서 아래의 코드를 입력한다.

   ```typescript
   // ~/src/prisma/prisma.service.ts
   import { INestApplication, Injectable, OnModuleInit } from "@nestjs/common"
   import { PrismaClient } from "../../prisma/generated" //보통은 @prisma/client에서 불러오지만 yarn berry를 써서 따로 생성된 디렉토리에서 불러왔다.

   @Injectable()
   export class PrismaService extends PrismaClient implements OnModuleInit {
     async onModuleInit() {
       await this.$connect()
     }

     async enableShutdownHook(app: INestApplication) {
       this.$on("beforeExit", async () => {
         await app.close()
       })
     }
   }
   ```

   여기서부터 당황스러웠다. 지금까지 프리즈마 클라이언트를 생성하면 `const prisma = new PrismaClient()`같은 방법을 썼는데 서비스를 `PrismaClient`에서 `extends`하고 이걸 `onModuleInit`에 `implement`를 한다._[extends? implement?](https://www.howdy-mj.me/typescript/extends-and-implements/)_ [공식문서](https://docs.nestjs.com/recipes/prisma)에 이유가 상세히 잘 나와있지만 간단하게 설명하자면 **Prisma**를 **DB**에 빠르게 연결하기 위해 `onModuleInit`을 사용한다. 선택사항이지만 사용하지 않을경우 **Prisma**는 첫 호출이 있기 전까지 **DB**에 연결하지 않는다. `enableShutdownHooks`는 [공식문서](https://docs.nestjs.com/recipes/prisma#issues-with-enableshutdownhooks)와 [이 포스트](https://progressivecoder.com/build-a-nestjs-prisma-rest-api/)를 보면 **Prisma**와 **NestJS**가 종료메서드에 상호간섭하기 때문에 서비스에서 함수를 생성하고 `main.ts`에서 호출해서 종료를 강제해주는 것 같다.(이부분은 완벽하게 이해를 못했다.)

   <details>
   <summary><code>main.ts</code></summary>

   ```typescript
   // ~/src/main.ts

   import { NestFactory } from "@nestjs/core"
   import { AppModule } from "./app.module"
   import { PrismaService } from "./prisma/prisma.service"

   async function bootstrap() {
     const app = await NestFactory.create(AppModule)
     const prisma: PrismaService = app.get(PrismaService)
     prisma.enableShutdownHooks(app)
     await app.listen(3000)
   }
   bootstrap()
   ```

   </details>

3. `prisma.module.ts`에서 `Global`로 `export`해서 어느 모듈에서나 `provider`추가 없이 `injection`이 가능하도록 해준다.

   ```typescript
   // ~/src/prisma/primsa.module.ts

   import { Global, Module } from "@nestjs/common"
   import { PrismaService } from "./prisma.service"

   @Global()
   @Module({
     providers: [PrismaService],
     exports: [PrismaService],
   })
   export class PrismaModule {}
   ```

4. 이제 설정이 끝났고 다른 서비스에서 `constructor`에 `PrismaService`를 불러와 사용하면 된다.

   ```typescript
   // ~/src/post/post.service.ts

   import { Injectable } from "@nest/common"
   import { Post } from "prisma/generated"
   import { PrismaService } from "src/prisma/prisma.service"
   //...

   @Injectable()
   export class PostService {
     constructor(private prisma: PrismaService) {}

     // ...

     async findOne(id: string): Promise<Post | null> {
       return this.prisma.post.findUnique({
         where: { id },
       })
     }

     // ...
   }
   ```

### 2. 테스트?

**GraphQL**을 버리고 처음 **NextJS**에서 기본 API routing으로 백엔드 기능을 구현했을때는 빠르게 최소한의 기능만 구축하기 위해서 테스트를 전혀 안하다가 **NestJS**로 넘어오면서 **TDD**를 해봐야겠다는 욕심(?)이 생겼다.
테스트는 기본 설정돼있는 **Jest**로 하려했는데 nomadcoder에서 NestJS + Jest 기본 강의만 듣고 시작한 나에게 **TDD**는 너무 큰 산이었다.
아직 간단한 CRUD밖에 없기때문에 테스트코드야 어떻게 짠다지만 **DB**와 **ORM**은 어떻게 해야하는지 전혀 감이 오지 않았다.
당장에 **Jest**자체도 낯설었다.
구글을 뒤져 **NestJS**와 **Prisma**를 사용해 테스트하는 예제 몇개를 구할 수 있었지만 대부분 **Prisma**를 `mocking`해서 테스트하는 방식이었다.
하지만 **Prisma**자체가 제대로 돌아갈지도 미지수인 상황에서 `mocking`만 해봤자 무슨의미인가 싶고 **TDD**를 도입하는 만큼 수동테스트를 최대한 줄이고싶었다.
문제는 현재 개발용 DB에 테스트용 더미데이터를 넣어뒀는데 계속 불필요한 테스트코드가 쌓이는것도 싫고, 만약 서비스를 배포한 후에도 같은 **DB**에 연결되면 심각한 문제를 초래하기 때문에 별도의 테스트용 DB에 연결할 필요가 있었다.
지금까지는 다른 DB에 연결할 필요가 없었기 때문에 방법을 전혀 모르는 상태였다.
여러모로 방법을 찾아봤는데 처음 생각한 방안은 **PrismaClient**를 **서비스 DB**와 **Test DB**에 각각 연결되도록 `generate`를 두번 하는 방법이었다.
하지만 방법도 복잡하고 **Prisma Engine**이나 `generate`에 대한 이해가 부족해서 맞는 방법인지 확신이 없었다.

### 3. Prisma ↔ Test용 DB 연결

한참 공식문서를 뒤지다가 단서를 얻었다. **PrismaClient**를 생성할 때 `datasource`를 `overriding`하는 방법이었다.

문제는 다시 **OOP**로 돌아온다. **FP**에서는

```typescript
const prisma = new PrismaClient({
  datasource: {
    db: { url: DatabaseUrl },
  },
})
```

처럼 간단하게 `datasource overriding`을 할 수 있는데 **NestJS**같은 **Class**에서는 어떻게 해야할지 모르겠는데 자료도 없어 막막했다.
나오지도 않는 구글을 한참 뒤지다가, **Prisma**랑 **PrismaClient**라이브러리 소스를 한참뒤지다가 겨우 단서를 찾았다.
`constructor`에 `super`로 선언해주면 되는 간단한 문제였다.

> <details>
> <summary>코드</summary>
>
> ```typescript
> // ~/src/prisma/prisma.service.ts
>
> // import ...
>
> @Injectable()
> export class PrismaService extends PrismaClient implements onModuleInit {
>   constructor() {
>     super({
>       datasources: {
>         db: {
>           url: process.env.TEST_DATABASE_URL,
>         },
>       },
>     })
>   }
> }
>
> // async onModuleInit...
> ```
>
> </details>

방법은 찾았는데 DB URL을 어떻게 입력할지를 놓고 또 한참 씨름했다.
`.env`에 넣고 `NODE_ENV`에 따라 각각 다른 주소가 들어가도록 하는데 `@nestjs/config`를 쓰니 문제가 발생했다.
`.env`와 `.env.development`, `.env.test`에 모두 `DATABASE_URL`이라는 같은 환경변수로 설정하고 돌렸더니 주구장창 `.env`에 있는 주소만 입력돼서 난감했다.
`.env`는 `DATABASE_URL`로 유지하고 개발과 테스트 환경변수를 다른 이름으로 지정했더니 제대로 불러오는데 서버를 직접 돌릴때는 `NODE_ENV`를 바꿔가며 돌려봐도 문제없이 잘 돌아가는데 `jest`로 테스트만 돌리면 `config`가 환경변수를 불러오지 못했다.
한참씨름하다가 그냥 `.env`에 `TEST_DATABASE_URL`로 Test DB 이름을 설정하고 `process.env`로 불러오는 방법을 택했다. 잘 돌아간다.

### 결과

이것저것 붙잡고 한참 씨름한 덕분에 `constructor`와 `super`에 대한 이해도와 `@nestjs/config`사용, 환경변수 적용이 익숙해졌다.
뜯어보면 별거 아니지만 누군가는 나와 같은 문제를 겪을 수도 있을 것 같아 문제 해결하고 잊어버리기 전에 급하게 적었다.
테스트도 결국 비용이고 대부분의 테스트코드가 `mocking`으로 적히는 걸로 보아 그게 효율적인 방법일 것 같아서 어느정도 DB에 직접 연결해서 테스트를 하다가 규모가 커지면 `mocking`해서 테스트 하는 방법으로 전환해야겠다.

### 참고자료

- [NestJS 공식문서](https://docs.nestjs.com/recipes/prisma)
- [Prisma 공식문서](https://www.prisma.io/docs/reference/api-reference/prisma-client-reference#datasources)
- <https://progressivecoder.com/build-a-nestjs-prisma-rest-api/>
- <https://blog.logrocket.com/how-to-use-nestjs-prisma/>
