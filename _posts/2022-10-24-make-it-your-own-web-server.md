---
layout: post
title: Make it my own server with NodeJS, Express, Typescript
subtitle: MVP structure to build up web server and Authentication
comments: true
tags: [NodeJS, Express, Typescript]
---

## Node.js, Express, Typescript 

NodeJS는 브라우저와 마찬가지로 Chrome이 만든 V8 엔진이 내장되어 자바스크립트를 실행할 수 있는 자바스크립트 런타임 환경이다. 
자바스크립트 언어를 사용하여 프론트엔드와 백엔드 프로젝트 모두에 적용시킬 수 있다는 의미로 NodeJS가 백엔드 프로그래밍 언어로 각광받는 이유 중 하나이다. 실제로 나 역시 대학교 2학년 때 React 프레임워크를 활용하여 웹 프론트엔드 개발자로서 동아리 메인 사이트를 만드는 프로젝트에 참여한 적이 있기 때문에 자바스크립트 비동기 처리, 타입스크립트 사용 경험이 있었기 때문에 이를 기반으로 자신있게 개발할 수 있었다. 타입스크립트를 적용했을 때가 단순히 자바스크립트 언어만을 사용했을 때보다 타입 에러를 체크하면서 코딩해나갈 수 있기 때문에 개발 생산성이 훨씬 높다는 것을 느꼈다. 더군다나 모바일 클라이언트와 통신하기 위해 필요한 요청과 응답, MySQL에 접근하기 위해 필요한 로직을 구현하기 위해 커스텀 타입, 커스텀 함수를 만드는데 타입스크립트가 매우 유용하게 사용되었다. (앞으로 자바스크립트를 쓰는 프로젝트에 타입스크립트를 배제할 이유가 전혀 없을 것 같다.)

이 포스팅에서는 학교 친구들과 진행하였던 Debugging의 서버 사이드 프로젝트에서 내가 적용시켰던 주요 기술들, 1. **MVP 아키텍처를 기반으로 설계된 프로젝트 전반의 구조**를 살펴보고 2. **토큰과 미들웨어를 사용하여 인증을 어떻게 처리했는지**에 대해 정리해볼 것이다.

아래 코드는 npm start를 cmd창에 입력하면 타입스크립트를 자바스크립트로 컴파일해주고 변경사항이 생길 때마다 서버를 자동으로 재시작할 수 있도록 해주는 내가 사용했던 아주 유용한 명령어이다. 
```json
 "scripts": {
    "start": "concurrently \"tsc -w\" nodemon dist/app"
  }
```

{: .box-note}
**Note:** 순수 자바스크립트 언어로 작성된 외부 모듈을 타입스크립트 파일에서 import 해오면 에러가 발생한다. 그 이유는 사용하려고 하는 api들의 타입이 정의되지 않았기 때문이다. 따라서 이러한 라이브러리들의 경우 @types/ 로 시작하는 추가적인 라이브러리를 별도로 설치해줘야 한다. 

## MVP 아키텍처 기반으로 구현된 회원가입, 로그인 로직 

**app.ts**

```javascript
import express from 'express';
import UserRouter from './routes/user.js';

const app = express();

app.use(express.json());

app.use('/', (req: Request, res: Response, next: NextFunction) => {
	res.sendStatus(404);
});

app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
	if (err) {
		console.log(err);
		res.status(500).send('Internal Server Error...');
	}
});

app.listen(config.host.port);
```

**/routes/user.ts**

```javascript
import express from 'express';
import * as UserController from '../controller/user.js';

const userRouter = express.Router();

userRouter.post('/signup', UserController.createUser);
userRouter.post('/login', UserController.login);

export default userRouter;
```

**/controller/user.ts**

```javascript
import { Request, Response, NextFunction } from 'express';
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';
import * as UserRepository from '../data/user.js';
import { config } from '../config.js';
import { UserRegistration, UserLogin} from '../types/index.js';

export async function createUser(
	req: Request,
	res: Response,
	next: NextFunction
) {
	const { userName, password } = req.body as UserRegistration;
	const user = await UserRepository.findUserByName(userName);
	if (user) {
		return res.sendStatus(409);
	}

	const hashed = await bcrypt.hash(
		password,
		parseInt(config.bcrypt.saltsRound)
	);
	const userId = await UserRepository.createUser({
		...req.body,
		password: hashed,
	});

	const token = createJWT(userId);

	res.status(201).json({
		token,
		userName,
	});
}

export async function login(req: Request, res: Response, next: NextFunction) {
	const { userName, password } = req.body as UserLogin;
	const user = await UserRepository.findUserByName(userName);
	if (!user) return res.sendStatus(401);

	const match = await bcrypt.compare(password, user.password);
	if (!match) return res.sendStatus(401);

	const token = createJWT(user.id);

	res.status(200).json({
		token,
		userName,
	});
}

function createJWT(userId: number): string {
	return jwt.sign({ userId }, config.jwt.privateKey, {
		expiresIn: config.jwt.expirSecs,
	});
}

```

**/data/user.ts**

```javascript
import { UserRegistration, User } from '../types/index.js';
import createPromiseWithDBQuery from '../util/promise.js';

export function createUser(user: UserRegistration): Promise<number> {
	return (
		createPromiseWithDBQuery <
		number >
		('INSERT INTO users SET ?',
		user,
		(resolve, result) => resolve(result['insertId']))
	);
}

export function findUserByName(userName: string): Promise<User> {
	return (
		createPromiseWithDBQuery <
		User >
		('SELECT * FROM users WHERE userName = ?',
		userName,
		(resolve, result) => resolve(result[0]))
	);
}
```

**/util/promise.ts**

```javascript
import { pool } from '../db/database.js';
import {
	ResolveCallback,
	QueryParamType,
	PromiseReturnType,
} from '../types/index.js';

export default function createPromiseWithDBQuery<T = PromiseReturnType>(
	query: string,
	params: QueryParamType,
	callback: ResolveCallback<T>
): Promise<T> {
	return new Promise((resolve, reject) => {
		pool.query(query, params, (error, result) => {
			if (error) reject(error);
			else callback(resolve, result);
		});
	});
}
```

**/types/index.d.ts**

```javascript
export type ResolveCallback<T> = (
	resolve: (value: T | PromiseLike<T>) => void,
	result: any
) => any;
```

**/db/database.ts**

```javascript
import mysql from 'mysql';
import { config } from '../config.js';

export const pool = mysql.createPool({
	host: config.db.host,
	user: config.db.user,
	password: config.db.password,
	database: config.db.database,
	port: config.db.port,
});
```

자바스크립트는 **Asynchronous I/O**의 특성때문에 MySQL에 접근하여 원하는 결과를 받아오는 처리를 해주어야 한다. 회원가입, 로그인 기능 외에도 서비스 정보를 불러오고 관심 상품을 추가/삭제하는 등 db에 접근하는 작업이 빈번하므로 createPromiseWithDBQuery라는 유틸리티 함수로 따로 정의해주었다. 프로미스를 직접 반환하고 접근하는데 성공하면 resolve 함수를 콜백에 넘겨주는 방식이다. 내가 개발하는 시점에는 typescript와 mysql2가 호환이 되지 않았기 때문에 이렇게 하는 것이 내 선에서는 최선이었다. 

db에 접근하는데 실패하거나 SQL 쿼리문이 잘못 작성되었을 경우 등 서버 내부에서 오류가 발생하여 통신이 실패할 때 상태코드 500과 함께 'Internal Server error...'라는 메시지를 출력해주는 최종적인 에러 핸들러를 app.js에서 정의해주었다. 원래는 next(error)를 호출해주어야 하지만 사전에 'external-async-errors' 라이브러리를 추가해주어 자동적으로 에러 핸들러가 호출될 수 있다. 이 경우 에러가 발생할 수 있는 미들웨어의 반환 타입이 반드시 프로미스여야한다. 이렇게 외부 라이브러리를 추가해주는 대신 express 라이브러리의 version을 5.0.0-alpha.8 이상으로 업그레이드 시켜주는 것 역시 같은 효과를 지닌다. 

## Authentication

[Once the user is logged in, each subsequent request will include the JWT, allowing the user to access routes, services, and resources that are permitted with that token.](https://jwt.io/introduction) 그렇다. 회원가입 혹은 로그인 이후 추가적인 요청들은 인증된 회원만이 리소스에 접근할 수 있어야한다. 회원가입이나 로그인에 성공하면 정해진 시간동안 유효한 토큰을 클라이언트에게 넘겨주고 이를 이용하여 클라이언트가 요청을 할 때마다 헤더에 담아 서버에서 유효한 사용자인지 아닌지의 여부를 확인할 수 있게 해야 한다. [해당 페이지](https://github.com/auth0/node-jsonwebtoken)는 JSON 웹 토큰을 구체적으로 사용하는 방법에 대해 잘 알려주고 있다. 

```javascript
function createJWT(userId: number): string {
	return jwt.sign({ userId }, config.jwt.privateKey, {
		expiresIn: config.jwt.expirSecs,
	});
}
```
위의 코드는 유저의 고유한 아이디를 넘겨 받아 비동기적으로 토큰을 생성하는 함수이다. 

**/middleware/auth.ts**

```javascript
import { NextFunction, Request, Response } from 'express';
import jwt from 'jsonwebtoken';
import { config } from '../config.js';
import { JwtPayloadWithUserId } from '../types/index.js';
import * as UserRepository from '../data/user.js';

export const isAuth = (req: Request, res: Response, next: NextFunction) => {
	const authHeader = req.get('Authorization');

	let token;

	if (authHeader && authHeader.split(' ')[0].startsWith('Bearer')) {
		token = authHeader.split(' ')[1];
	}

	if (!token) {
		return res.status(401).json({ message: 'Authorization header is invalid.' });
	}

	jwt.verify(
		token,
		config.jwt.privateKey,
		async (err: jwt.VerifyErrors | null, decoded: JwtPayloadWithUserId) => {
			if (err) {
				return res.status(401).json({ message: err.message });
			}
			
			const user = await UserRepository.findUserById(decoded.userId);
			if (!user) {
				return res.status(401).json({ message: 'User does not exist.'});
			}

			req.userId = decoded.userId;
			next();
		}
	);
};
```
이는 사용자 인증을 하는 미들웨어 함수이다. 헤더의 유효한 토큰이 있는지를 체크하고 비밀키를 이용해 토큰을 해독하여 사용자가 등록이 되어있다면 req에 해독하여 얻어낸 사용자의 고유 아이디를 정의해주고 다음 미들웨어가 정상적으로 호출될 수 있도록 해주었다. 


**/routes/user.ts**

```javascript
userRouter.post('/signup', UserController.createUser);
userRouter.post('/login', UserController.login);
userRouter.delete('/', isAuth, UserController.remove);
userRouter.get('/mypage', isAuth, UserController.getMyPage);
```
마지막으로 해야할 일은 인증이 필요한 router에 isAuth 미들웨어를 컨트롤러 미들웨어 이전에 인자로 추가해주는 것이다. 이렇게 하면 다음 컨트롤러 미들웨어에서 req 파라미터를 통해 userId 속성에 접근할 수 있다. 


## Epilogue
이 프로젝트는 나의 첫번째 서버 사이드 프로젝트라는 점에서 큰 의의가 있다. 여전히 리팩토링이 필요한 부분이 많고 배워야할 심화 기술이 존재하지만 데이터 베이스 스키마, REST APIs를 설계한 일부터 최종적인 배포까지 백엔드의 기본 A-Z를 터득한 느낌이라 매우 뿌듯하다. 기술의 부족함으로 인해 설문조사 결과를 분석하여 예측 대상을 반환해주는 우리 프로젝트의 메인 서비스 기능을 구현하지 못한 것이 아쉽다. 그래서 다음 백엔드 서비스를 구현할 때에는 최근에 공부한 머신러닝 알고리즘과 파이썬 Flask 프레임워크를 토대로 보다 유용한 분석 및 추천 기능을 개발하는 것을 목표로 삼을 것이다. 

