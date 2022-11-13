---
layout: post
title: The way how I design and document Web APIs
subtitle: Swagger
comments: true
tags: [Swagger]
---

보통은 서비스를 본격적으로 구현하기 위해 REST API들을 먼저 설계하는 것이 사용자 입장에서 요구사항을 제대로 이해할 수 있다. 이는 개발 목표를 정확하게 정해 두고 개발을 시작할 수 있어 협업을 하면서 도움이 많이 된다. (**Not code first, Design first**) 그러므로 시간을 들여서라도 데이터 스키마를 먼저 정의하고 필요한 리소스를 접근하기 위한 API들을 사전에 정의해두는 것이 좋은데, 이때 유용한 것이 Open API Specification과 Swagger Tool을 이용하는 것이다. 이를 이용하면 어플리케이션 수준에서 별도의 코드 작성없이도 라우팅과 validation을 지원해주기도 하지만 이러한 기능은 이번 프로젝트에서는 사용하지 않고 문서화를 위해 채택하였다. Open API를 작성하기 위한 구현 방법은 [해당 페이지](https://swagger.io/specification/)를 참고하였다. 

**Before demo**
프로젝트 내부의 api 폴더에 **open.yaml** 파일을 만들었다. JSON으로 작성하는 것보다 yaml로 작성해서 JSON으로 변환하는 것이 가독성 측면에서 좋다. 또한 별도의 url을 통해 UI적으로 요청 및 응답 스키마, 요청별 발생할 수 있는 상태코드, 그룹화된 태그 등을 시각적으로 확인할 필요가 있다. 따라서 'yamljs'와 'swagger-ui-express'를 프로젝트에 추가해주었다.

**npm install yamljs swagger-ui-express**

아래는 [Swagger Editor](https://swagger.io/tools/swagger-editor/)을 이용해 open api 문서를 코드로 작성해준 결과이다. 크게 openapi(version), info, server, tags, path, components로 구성되어 있다. 어떻게 쓰였는지를 이해하기 위해 코드의 일부 중요한 부분을 살펴보도록 하자.

```yaml
tags:
  - name: user
    description: Methods to access and manage users
  - name: bugs
    description: Methods to access bug information and add a survey result
  - name: companies
    description: Methods to access company information and manage company interests and reservation
  - name: products
    description: Methods to access product information and manage product interests
```

이용가능한 API들은 크게 네 범주로 나뉜다. 접근하는 리소스 종류에 따라 user, bugs, companies, products로 구분하였다. 이를 통해 어플리케이션이 제공하는 메인 서비스들을 쉽게 확인할 수 있다.

```yaml
paths:
  /user/signup:
    post:
      tags:
        - user
      summary: Signs up a user to the Debugging service
      description: Creates a user account for the given user details.
      operationId: signup
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserRegistration'
        required: true
      responses:
        '201':
          description: Account Created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserAuthenticationSuccess'
        '400':
          description: Bad request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorApiResponse'
        '409':
          description: User already exists
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorApiResponse'

  /user/login:
    post:
      tags:
        - user
      summary: Logs in a user to the Debugging service
      description: Verify if a logged-in user is valid or not.
      operationId: login
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserLogin'
        required: true
      responses:
        '200':
          description: Login Succeeded
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserLogin'
        '401':
          description: Unauthorized
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorApiResponse'
```

단순성을 위해 회원 가입과 로그인 기능과 관련된 코드만 가져왔다. 각각의 메소드 별로 operationId를 정의하는 것은 해당 메소드명을 가진 함수를 invoke하여 라우팅을 할 수 있다. 그러나 이 기능은 사용하지 않고 직접 프로그래밍을 해서 작성하였다.  

```yaml
components:
  schemas:
    UserRegistration:
      type: object
      properties:
        username:
          type: string
        password:
          type: string
          format: password
          minLength: 5
        name:
          type: string
        contactNumbers:
          type: string
        email:
          type: string
          format: email
        address:
          type: string
          nullable: true
        sizeOfHouse:
          type: number
          nullable: true
        numOfRooms:
          type: integer
          nullable: true
      required:
        - username
        - password
        - name
        - contactNumbers
        - email
      example:
        username: 김다솔
        password: abc1234
        name: Dasol Kim
        contactNumbers: 010-1111-2222
        email: sorikiki98@sookmyung.ac.kr
    UserAuthenticationSuccess:
      type: object
      properties:
        token:
          type: string
        username:
          type: string
      required:
        - token
        - username
      example:
        username: 김다솔
    UserLogin:
      type: object
      properties:
        username:
          type: string
        password:
          type: string
          format: password
          minLength: 5
      required:
        - username
        - password
      example:
        username: 김다솔
        password: abc1234
```

components 섹션에 회원가입 시 body에 들어갈 UserRegistration과 로그인 시 필요한 UserLogin의 스키마를 정의하였다. 

```javascript
// app.ts
// add these below codes
const apiJSDocument = yamljs.load('./api/openapi.yaml');

app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(apiJSDocument));
```
최종적으로 app.ts 파일에 yamljs을 변환하여 json 형식으로 반환된 문서를 /api-docs 경로의 미들웨어로서 추가해주었다. 

