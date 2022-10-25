---
layout: post
title: Make it my own server with NodeJS, Express, Typescript
subtitle: MVP structure to build up web server and Authentication
comments: true
tags: [NodeJS, Express, Typescript]
---

## Node.js, Express, Typescript

Node.js is a javascript runtime build on Chrome's V8 engine. It means that we are able to apply javascript to both client side and server side at the same time. And It's common to use Express framework when we build web server using Node.js to make it convenient to use APIs. So I'm gonna focus on the APIs that I used provided by Express.

Typescript is a superset of pure javascript, a statically typed language and an object oriented language. I mainly used typescript for this project so it enabled me to avoid confusing typed coding by defining some customized types(data entity, callback, params, return type... etc)

Here, I omit previous procedures to set up my package and required dependencies.

```json
 "scripts": {
    "start": "concurrently \"tsc -w\" nodemon dist/app"
  }
```

This command was very helpful.😁 It converts ts file to js file automatically and run nodemon sequentially.

{: .box-note}
**Note:** When we import external modules, which are written in pure javascript, we also have to install the extra libraries via npm for typescript-version as dev-dependencies. (They are always prefixed with "@types/...")

## Sign-up, Sign-in feature on MVP structure

My ready-made project follows this below structure. Suppose that we are only concerned with how to register a new user and enable the user to log-in for simplicity.

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

It is important to get a return value after querying a database asynchronously. I frequently access to a mysql database so I made a separate function to create a new promise object. This is because I did not use 'mysql2' module which was not compatible with using typescript.(But please check if someone is going to build a project with typescript.)

Note that I improve consistency when it comes to final error-handling, sending a message 'Internal Server error...'. If a promise object calls a reject method, it automatically calls this final method, provided that we already add 'external-async-errors' dependencies to our project package and the return type of middlewares we defined is a promise.

## Authentication

[Once the user is logged in, each subsequent request will include the JWT, allowing the user to access routes, services, and resources that are permitted with that token.](https://jwt.io/introduction) Yes, that's what I want. Specifically if someone wants to delete his own account, upload posts or do something meaningful to our application, authorization must be guaranteed for some undesirable scenarios not to happen. [This page](https://github.com/auth0/node-jsonwebtoken) introduces how to use JSON Web tokens.

```javascript
function createJWT(userId: number): string {
	return jwt.sign({ userId }, config.jwt.privateKey, {
		expiresIn: config.jwt.expirSecs,
	});
}
```

I bring a createJWT function above to check how to create token asynchronously.

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

	if (authHeader && authHeader.split(' ')[1]) {
		token = authHeader.split(' ')[1];
	}

	if (!token) {
		return res.sendStatus(401);
	}

	jwt.verify(
		token,
		config.jwt.privateKey,
		async (err: jwt.VerifyErrors | null, decoded: JwtPayloadWithUserId) => {
			if (err) {
				console.log(err.message);
				return res.sendStatus(401);
			}

			const user = await UserRepository.findUserById(decoded.userId);
			if (!user) {
				return res.sendStatus(401);
			}

			req.userId = decoded.userId;
			next();
		}
	);
```

This is how I define a new middleware function for authentification. We have to check how it sends a request with a received token on the client-side. However, I'll post about this topic next time.

**/routes/user.ts**

```javascript
userRouter.post('/signup', UserController.createUser);
userRouter.post('/login', UserController.login);
userRouter.delete('/', isAuth, UserController.remove);
userRouter.get('/mypage', isAuth, UserController.getMyPage);
```

The last thing we should do is to add this middleware 'isAuth' before predefined series of middlewares. Now on the next middleware, we are able to access to a 'userId' property through a 'req' parameter.

## Epilogue

I'm so glad that I finally made my first server-side project, however it still needs a refactoring to make the code more concise. One thing that I've not applied to my project is to analyze survey and predict which item is supposed to be on the result. I realized that I can resolve this problem with a trained machine-learning model. For this reason, I'm going to invest time studying a python web framework, Flask to build a new machine learning application in the end.
