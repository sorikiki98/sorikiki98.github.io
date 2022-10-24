---
layout: post
title: Make it your own server with NodeJS, Express, Typescript 
subtitle: Basic structure to build up web server
comments: true
tags: [NodeJS, Express, Typescript]
---

## Node.js, Express, Typescript ## 
Node.js is a javascript runtime build on Chrome's V8 engine. It means that we are able to apply javascript to both client side and server side at the same time. And It's common to use Express framework when we build web server using Node.js to make it convenient to use APIs. So I'm gonna focus on the APIs provided by Express. 

Typescript is a superset of pure javascript, a statically typed language and an object oriented language. I mainly used typescript for this project so it enabled me to avoid confusing typed coding by defining some customized types(data entity, callback, params, return type... etc) 

Here, I omit previous procedures to set up my package and required dependencies.

```json
// ...
 "scripts": {
    // ...
    "start": "concurrently \"tsc -w\" nodemon dist/app"
  },
```
This command was very helpful.😁 It converts ts file to js file automatically and run nodemon sequentially.

{: .box-note}
**Note:** When we import external modules, which are written in pur e javascript, we also have to install the extra libraries via npm for typescript-version as dev-dependencies. (They are prefixed with "@types/...")

## Sign-up, Sign-in feature on MVP structure ## 
My ready-made project follows this below structure. Suppose that we are only concerned with how to register a new user and enable the user to log-in. 

## Asynchronous processing

## Connecting to MySQL

### Authentication




