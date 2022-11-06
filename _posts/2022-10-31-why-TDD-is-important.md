---
layout: post
title: TDD 왜 하는 걸까?
subtitle: 자바스크립트 테스트 프레임워크 JEST
comments: true
tags: [NodeJS, Express, Jest, axios]
---

필자는 Test Driven Development를 직접 시도해보기 전까지는 그 중요성을 알지 못했다. 그러나 배우는 과정에서 테스트 코드를 실제 개발 코드와 함께 푸쉬하는 것은 협업을 할 때 문서화의 효과가 있을 뿐만 아니라 

설치해야 하는 라이브러리
jest ts-jest @types/jest
node-mocks-http faker@5.5.3 axios

jest는 es6의 모듈과 잘 동작하지 않기 때문에 commonjs로 변환해주는 바벨 플러그인을 설치해야 한다.
npm i --save-dev @babel/plugin-transform-modules-commonjs

ts test파일에서 모듈을 import할 때 extension에 js를 붙여준다.
단, jest.config.js에서 추가적인 설정이 필요함
https://kulshekhar.github.io/ts-jest/docs/guides/esm-support/

jest --init
typescript 사용한다고 설정 x
preset: 'ts-jest',
v8 엔진 사용

테스트용 환경 변수 설정
.env.test 파일 생성하고
setupFiles: ['dotenv/config'],
npm i --save-dev cross-env (윈도우의 경우)


"test": "cross-env DOTENV_CONFIG_PATH=./.env.test jest --watchAll --verbose --testPathIgnorePatterns=/tests",
    "test:integration": "cross-env DOTENV_CONFIG_PATH=./.env.test jest --watchAll --verbose --testPathPattern=/tests",

유닛 테스트
todo 
auth 미들웨어
기능별 컨트롤러

서버 생성 함수, 종료 함수를 분리하는데 성공. 테스트 환경에서 포트를 따로 지정하지 않게 설정.
이슈 : 테스트할 때마다 fake data가 insert되어 기존 데이터베이스가 pollunated되는데 얘는 어떻게 하면 좋을까? 
통합 테스트(서버를 직접 시작하고 직접 중단할 수 있는 함수를 생성)
- 회원가입, 로그인 api
- 상품 전체 불러오기, 상품 하나 불러오기
- user db 정리하기