# frourio-express
<br />
<br />
<div align="center">
  <img src="https://frouriojs.github.io/frourio/assets/images/ogp.png" width="1280" alt="frourio-express" />
</div>

<div align="center">
  <a href="https://www.npmjs.com/package/frourio-express">
    <img src="https://img.shields.io/npm/v/frourio-express" alt="npm version" />
  </a>
  <a href="https://www.npmjs.com/package/frourio-express">
    <img src="https://img.shields.io/npm/dm/frourio-express" alt="npm download" />
  </a>
  <a href="https://github.com/frouriojs/frourio-express/actions?query=workflow%3A%22Node.js+CI%22">
    <img src="https://github.com/frouriojs/frourio-express/workflows/Node.js%20CI/badge.svg?branch=master" alt="Node.js CI" />
  </a>
  <a href="https://codecov.io/gh/frouriojs/frourio-express">
    <img src="https://img.shields.io/codecov/c/github/frouriojs/frourio-express.svg" alt="Codecov" />
  </a>
  <a href="https://lgtm.com/projects/g/frouriojs/frourio-express/context:javascript">
    <img src="https://img.shields.io/lgtm/grade/javascript/g/frouriojs/frourio-express.svg" alt="Language grade: JavaScript" />
  </a>
</div>

<p align="center">Fast and type-safe full stack framework, for TypeScript</p>
<br />
<br />
<br />

## Why frourio-express ?

Even if you write both the frontend and backend in TypeScript, you can't statically type-check the API's sparsity.

We are always forced to write "Two TypeScript".  
We waste a lot of time on dynamic testing using the browser and server.

<div align="center">
   <img src="https://frouriojs.github.io/frourio/assets/images/problem.png" width="1200" alt="Why frourio ?" />
</div>
<br />
<br />

Frourio-express is a framework for developing web apps quickly and safely in **"One TypeScript"**.

<div align="center">
   <img src="https://frouriojs.github.io/frourio/assets/images/architecture.png" width="1200" alt="Architecture of create-frourio-app" />
</div>
<br />
<br />

## Documents

https://frourio.io/docs

## Table of Contents

- [Install](#Install)
- [Controller](#Controller)
  - [Case 1 - Define GET: /tasks?limit={number}](#Controller-case1)
  - [Case 2 - Define POST: /tasks](#Controller-case2)
  - [Case 3 - Define GET: /tasks/{taskId}](#Controller-case3)
- [Hooks](#Hooks)
  - [Lifecycle](#Lifecycle)
  - [Directory level hooks](#Hooks-dir)
  - [Controller level hooks](#Hooks-ctrl)
- [Validation](#Validation)
- [Deployment](#Deployment)
  - [Frontend](#Deployment-frontend)
  - [Server](#Deployment-server)
- [Dependency Injection](#DI)
- [License](#License)

## Install

Make sure you have [npx](https://www.npmjs.com/package/npx) installed (`npx` is shipped by default since [npm](https://www.npmjs.com/get-npm) `5.2.0`)

```sh
$ npx create-frourio-app
```

Or starting with npm v6.1 you can do:

```sh
$ npm init frourio-app
```

Or with [yarn](https://yarnpkg.com/en/):

```sh
$ yarn create frourio-app
```

## Controller

<a id="Controller-case1"></a>

### Case 1 - Define GET: /tasks?limit={number}

`server/types/index.ts`

```ts
export type Task = {
  id: number
  label: string
  done: boolean
}
```

`server/api/tasks/index.ts`

```ts
import { Task } from '$/types' // path alias $ -> server

export type Methods = {
  get: {
    query: {
      limit: number
    }

    resBody: Task[]
  }
}
```

`server/api/tasks/controller.ts`

```ts
import { defineController } from './$relay' // '$relay.ts' is automatically generated by frourio-express
import { getTasks } from '$/service/tasks'

export default defineController(() => ({
  get: async ({ query }) => ({
    status: 200,
    body: (await getTasks()).slice(0, query.limit)
  })
}))
```

<a id="Controller-case2"></a>

### Case 2 - Define POST: /tasks

`server/api/tasks/index.ts`

```ts
import { Task } from '$/types' // path alias $ -> server

export type Methods = {
  post: {
    reqBody: Pick<Task, 'label'>
    status: 201
    resBody: Task
  }
}
```

`server/api/tasks/controller.ts`

```ts
import { defineController } from './$relay' // '$relay.ts' is automatically generated by frourio-express
import { createTask } from '$/service/tasks'

export default defineController(() => ({
  post: async ({ body }) => {
    const task = await createTask(body.label)

    return { status: 201, body: task }
  }
}))
```

<a id="Controller-case3"></a>

### Case 3 - Define GET: /tasks/{taskId}

`server/api/tasks/_taskId@number/index.ts`

```ts
import { Task } from '$/types' // path alias $ -> server

export type Methods = {
  get: {
    resBody: Task
  }
}
```

`server/api/tasks/_taskId@number/controller.ts`

```ts
import { defineController } from './$relay' // '$relay.ts' is automatically generated by frourio-express
import { findTask } from '$/service/tasks'

export default defineController(() => ({
  get: async ({ params }) => {
    const task = await findTask(params.taskId)

    return task ? { status: 200, body: task } : { status: 404 }
  }
}))
```

## Hooks

Frourio-express can use all of Express' middleware as hooks.  
There are four types of hooks, onRequest / preParsing / preValidation / preHandler.

### Lifecycle

```
Incoming Request
  │
  └─▶ Routing
        │
  404 ◀─┴─▶ onRequest Hook
              │
    4**/5** ◀─┴─▶ preParsing Hook
                    │
          4**/5** ◀─┴─▶ Parsing
                          │
                4**/5** ◀─┴─▶ preValidation Hook
                                │
                      4**/5** ◀─┴─▶ Validation
                                      │
                                400 ◀─┴─▶ preHandler Hook
                                            │
                                  4**/5** ◀─┴─▶ User Handler
                                                  │
                                        4**/5** ◀─┴─▶ Outgoing Response
```

<a id="Hooks-dir"></a>

### Directory level hooks

Directory level hooks are called at the current and subordinate endpoints.

`server/api/tasks/hooks.ts`

```ts
import { defineHooks } from './$relay' // '$relay.ts' is automatically generated by frourio-express

export default defineHooks(() => ({
  onRequest: [
    (req, res, next) => {
      console.log('Directory level onRequest first hook:', req.path)
      next()
    },
    (req, res, next) => {
      console.log('Directory level onRequest second hook:', req.path)
      next()
    }
  ],
  preParsing: (req, res, next) => {
    console.log('Directory level preParsing single hook:', req.path)
    next()
  }
}))
```

<a id="Hooks-ctrl"></a>

### Controller level hooks

Controller level hooks are called at the current endpoint after directory level hooks.

`server/api/tasks/controller.ts`

```ts
import { defineHooks, defineController } from './$relay' // '$relay.ts' is automatically generated by frourio-express
import { getTasks, createTask } from '$/service/tasks'

export const hooks = defineHooks(() => ({
  onRequest: (req, res, next) => {
    console.log('Controller level onRequest single hook:', req.path)
    next()
  },
  preParsing: [
    (req, res, next) => {
      console.log('Controller level preParsing first hook:', req.path)
      next()
    },
    (req, res, next) => {
      console.log('Controller level preParsing second hook:', req.path)
      next()
    }
  ]
}))

export default defineController(() => ({
  get: async ({ query }) => ({
    status: 200,
    body: (await getTasks()).slice(0, query.limit)
  }),
  post: async ({ body }) => {
    const task = await createTask(body.label)

    return { status: 201, body: task }
  }
}))
```

## Validation

Query, reqHeaders and reqBody are validated by specifying Class with [class-validator](https://github.com/typestack/class-validator).  
The class needs to be exported from `server/validators/index.ts`.

`server/validators/index.ts`

```ts
import { MinLength, IsString } from 'class-validator'

export class LoginBody {
  @MinLength(5)
  id: string

  @MinLength(8)
  pass: string
}

export class TokenHeader {
  @IsString()
  @MinLength(10)
  token: string
}
```

`server/api/token/index.ts`

```ts
import { LoginBody, TokenHeader } from '$/validators'

export type Methods = {
  post: {
    reqBody: LoginBody
    resBody: {
      token: string
    }
  }

  delete: {
    reqHeaders: TokenHeader
  }
}
```

```sh
$ curl -X POST -H "Content-Type: application/json" -d '{"id":"correctId","pass":"correctPass"}' http://localhost:8080/api/token
{"token":"XXXXXXXXXX"}

$ curl -X POST -H "Content-Type: application/json" -d '{"id":"abc","pass":"12345"}' http://localhost:8080/api/token -i
HTTP/1.1 400 Bad Request

$ curl -X POST -H "Content-Type: application/json" -d '{"id":"incorrectId","pass":"incorrectPass"}' http://localhost:8080/api/token -i
HTTP/1.1 401 Unauthorized
```

## Deployment

Frourio-express is complete in one directory, but not monolithic.  
Frontend and server are just statically connected by a type and are separate projects.  
So they can be deployed in different environments.

<a id="Deployment-frontend"></a>

### Frontend

```sh
$ npm run build:front
$ npm run start:front
```

<a id="Deployment-server"></a>

### Server

```sh
$ npm run build:server
$ npm run start:server
```

or

```sh
$ cd server
$ npm run build
$ npm run start
```

<a id="DI"></a>

## Dependency Injection

Frourio-express use [frouriojs/velona](https://github.com/frouriojs/velona) for dependency injection.

`server/api/tasks/index.ts`

```ts
import { Task } from '$/types'

export type Methods = {
  get: {
    query?: {
      limit?: number
      message?: string
    }

    resBody: Task[]
  }
}
```

`server/service/tasks.ts`

```ts
import { PrismaClient } from '@prisma/client'
import { depend } from 'velona' // dependency of frourio
import { Task } from '$/types'

const prisma = new PrismaClient()

export const getTasks = depend(
  { prisma: prisma as { task: { findMany(): Promise<Task[]> } } }, // inject prisma
  async ({ prisma }, limit?: number) => // prisma is injected object
    (await prisma.task.findMany()).slice(0, limit)
)
```

`server/api/tasks/controller.ts`

```ts
import { defineController } from './$relay'
import { getTasks } from '$/service/tasks'

const print = (text: string) => console.log(text)

export default defineController(
  { getTasks, print }, // inject functions
  ({ getTasks, print }) => ({ // getTasks and print are injected function
    get: async ({ query }) => {
      if (query?.message) print(query.message)

      return { status: 200, body: await getTasks(query?.limit) }
    }
  })
)
```

`server/test/server.test.ts`

```ts
import controller from '$/api/tasks/controller'
import { getTasks } from '$/service/tasks'

test('dependency injection into controller', async () => {
  let printedMessage = ''

  const injectedController = controller.inject({
    getTasks: getTasks.inject({
      prisma: {
        task: {
          findMany: () =>
            Promise.resolve([
              { id: 0, label: 'task1', done: false },
              { id: 1, label: 'task2', done: false },
              { id: 2, label: 'task3', done: true },
              { id: 3, label: 'task4', done: true },
              { id: 4, label: 'task5', done: false }
            ])
        }
      }
    }),
    print: (text: string) => {
      printedMessage = text
    }
  })()

  const limit = 3
  const message = 'test message'
  const res = await injectedController.get({
    query: { limit, message }
  })

  expect(res.body).toHaveLength(limit)
  expect(printedMessage).toBe(message)
})
```

```sh
$ npm test

PASS server/test/server.test.ts
  ✓ dependency injection into controller (4 ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.67 s, estimated 8 s
Ran all test suites.
```

## License

Frourio-express is licensed under a [MIT License](https://github.com/frouriojs/frourio-express/blob/master/LICENSE).
