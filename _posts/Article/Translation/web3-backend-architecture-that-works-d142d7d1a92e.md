---
title: Web3 Backend architecture that works
authorURL: ""
originalURL: https://medium.com/@miki.digital/web3-backend-architecture-that-works-d142d7d1a92e
translator: "sum"
reviewer: ""
---

# Web3 Backend architecture that works
# 好用的Web3后端架构

> Building a backend for a blockchain/web3 project is a completely different beast than in web2.
> 构建区块链/web3项目的后端与构建web2项目的后端完全是两回事。

**Isn’t backned mostly web2 tech, you may ask?** While that’s partially correct, there are many tricky aspects when it comes to building backends for web3 apps:
**你可能会问后端大部分不都是web2的技术吗？** 部分正确，为web3应用构建后端时存在许多其他方面的棘手问题：

-   **Tight timelines & budget** — you don’t have time & money to burn by hiring a devops team, configuring CI/CD and making a state-of-the-art microservice architecture. Things need to be simple, yet work flawlessly
-   **时间紧&预算少** — 你可能没有时间和费用去雇佣一个专业的DevOps团队配置CI/CD并构建先进的微服务架构。服务要求简单稳定。
-   **Caching** — you often need to implement caching for RPC calls to the blockchain
-   **缓存** — 通常需要对区块链的RPC调用实现缓存
-   **Compliance** — since blockchain involves finance, you need to stay compliant with regulation
-   **合规性** — 由于区块链涉及金融，因此需要符合监管要求
-   **Safety** — Properly storing your private keys for Smart Contract invocations from the backend is key
-   **安全性** — 妥善存储私钥对于后端调用智能合约至关重要

With the multitude of apps, of course there’s no one size fits all solution, but we want to share a blueprint we use and some of our go-to frameworks at [MiKi Digital](https://miki.digital/)
由于应用程序众多，没有一种解决方案适合所有情况，但我们想分享一些我们在[MiKi Digital](https://miki.digital/)的蓝图及首选框架。

# Monolith vs Microservices
# 单体架构 vs 微服务

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*SQ9PJ4NUQQ5UeyFJMHqqQw.png)

While from time to time we all want to imagine we are buliding the next google and create a state of the art microservice architecture, often — it’s not needed! **Unless you plan on having extreme load from the start and can spare the time and money it costs to build a proper microservice architecture, and what’s more importantly — have the expertise to do so, you shouldn’t.** Sticking with Monolith will give you good enough performance for 90% of your use cases.
虽然经常会幻想我们正在构建下一个谷歌并创建最先进的微服务架构，但通常情况下 - 这是不需要的！**除非你从一开始就计划能够承受极高的负载，并且有时间和金钱去构建一个合适的微服务架构，而且更重要的是 — 具备做到这一点的专业知识，否则不应该这么做。** 单体架构的性能可以满足90%的使用场景。

> _Don’t overengineer from the start, simplicity is the key to success._
> **不要从一开始就过度设计，简单才是成功的关键。**

# NodeJS + TypeScript
# NodeJS + TypeScript

NodeJS is the industry standard solution, even though some recent competitors might take the crown from it soon. Let’s take a look at them:
NodeJS是行业标准解决方案，尽管一些竞争对手可能很快就会超过它。让我们来了解一下：

-   [Deno](https://deno.com/) — a JS runtime with first-class TypeScript support. It might be a bit faster than NodeJS but doesn’t offer interoperability, so while the runtime might be mature and better than Node, the frameworks and community aren’t as good as in Node.
-   [Deno](https://deno.com/) — 对TypeScript支持非常友好的JS运行时。它可能比NodeJS快一点，但不提供互操作性，因此虽然运行时可能比Node更成熟更好，但框架和社区不如Node。
-   [Bun](https://bun.sh/) — a JS runtime (almost)fully interoperable with Node, built in Rust for blazing-fast speed. You can use all the frameworks & libraries that NPM has, while enjoying better performance & faster compile times. While the vision is grand, at the time of writing this article Bun isn’t fully interoperable, which can bring some downsides. But if you are reading this a couple of months later — Bun might be a worthy competitor to NodeJS. **Don’t forget to check it’s compatability with all the libraries you planning to use first**
-   [Bun](https://bun.sh/) — 一个（几乎）完全可与Node互操作的JS运行时，采用Rust构建，速度极快。可以使用NPM的所有框架和库，同时性能更好和编译速度也很快。虽然愿景很宏伟，但在撰写本文时，Bun尚未完全实现互操作，这可能会带来一些缺点。但如果你几个月后读到这篇文章——Bun可能是NodeJS的有力竞争对手。**不要忘记先逐一检查它和你准备使用的库的兼容性。**
    TypeScript is just how everyone does it at this point, some argue that it pollutes the code, however, with a strict config, e.g.
    TypeScript目前已经人人都在用，有些人认为它会污染代码，但是，通过严格的配置，例如

`{  
  "compilerOptions":{  
    "strict":false,  
    "noUnusedParameters":true,  
    "noImplicitReturns":true,  
    "noFallthroughCasesInSwitch":true,  
    "forceConsistentCasingInFileNames":true  
  }  
}`

It enforces you to think how you want the data to look, making it easier for other devs to understand the code.
它迫使你思考你的代码应该是什么样的，才能其他开发人员更容易理解。

# GraphQL Yoga + Nexus GraphQL
# GraphQL Yoga + Nexus GraphQL

The majority of developers use NestJS for working with GraphQL as it has first-class support and is dead-easy to get started with. However, does it make sense to bring a clunky BE framework into a project that needs to move fast, and work fast? Not really — NestJS comes with a ton of boilerplate code, following all those OOP, Dependency Injection, IoC, etc. patterns.
大多数开发人员使用NestJS来处理GraphQL，因为它提供了一流的支持，并且非常容易上手。然而，将笨重的后端（BE，backend）框架引入到一个需要快速进行和高效工作的项目中真的合理吗？并不是这样——NestJS附带了大量的样板代码（boilerplate code），遵循了所有的面向对象编程（OOP，Object-Oriented Programming）、依赖注入（DI，Dependency Injection）、控制反转（IoC，Inverse of Control）等设计模式。

> _Just because it has been the industry standart for many years, with languages such as Java and .NET having the same approach, doesn’t mean NestJS is the simplest way to solve the problem._
> **仅仅是因为它多年来已经成为了行业标准，像Java和.NET等语言也采用相同的方法，但并不意味着NestJS是解决问题的最简单方法。**

Also, NestJS’s bundle size is [**20 times biggest than of the same solution but with NodeJs and some minimalistic libraries instead**](https://medium.com/@miki.digital/reducing-lambda-bundle-size-with-esbuild-and-lambda-layers-c4803f1007cc)
此外，NestJS的包大小是[**使用NodeJs和一些最小化库实现相同功能的20倍**](https://medium.com/@miki.digital/reducing-lambda-bundle-size-with-esbuild-and-lambda-layers-c4803f1007cc)。

So we chose to go with pure NodeJS and a few libraries which allows us to have cleaner code and develop faster!
因此，我们选择使用纯NodeJS和少量的库，这使我们的代码更清晰并且开发速度更快！

For GraphQL we can leverage [GraphQL Yoda](https://the-guild.dev/graphql/yoga-server) — an amazing graphql server with a ton of out of the box features like caching, cookies, etc. Apollo Server is nice, but [**GraphQL Yoga is the exact case where we can trade a bit of maturity(GraphQL Yoda is newer than Apollo) for some performance & ease-of-use.**](https://the-guild.dev/graphql/yoga-server/docs/comparison#graphql-yoga-and-apollo-server)
对于GraphQL，我们可以利用GraphQL Yoda— 一个令人惊叹的graphql服务器，它具有大量开箱即用的功能，如缓存、cookie等。 Apollo Server也不错，但[**GraphQL Yoda可以让我们在一定程度上牺牲一些成熟度（GraphQL Yoda比Apollo更新）以换取性能和易用性**](https://the-guild.dev/graphql/yoga-server/docs/comparison#graphql-yoga-and-apollo-server)。

I already made a rather [comprehensive review of code-first vs schema-first approaches](https://www.linkedin.com/posts/mikhail-kedel_javascript-graphql-backend-activity-7021491967448543232-_zKJ?utm_source=share&utm_medium=member_desktop), in-short: **I prefer code-first**. And the perfect library for that is [nexus-graphql](https://nexusjs.org/): just a damn good library with everything you need for code-first graphql approach. We’ll show some code snippets of later in the article
我已经对[code-first和schema-first进行了全面的评估](https://www.linkedin.com/posts/mikhail-kedel_javascript-graphql-backend-activity-7021491967448543232-_zKJ?utm_source=share&utm_medium=member_desktop)，简而言之：**我更喜欢code-first**。而对此最完美的库就是[nexus-graphql](https://nexusjs.org/)：它非常厉害，为code-first的GraphQL方法提供了你所需的一切。我们将在文章后面展示一些代码片段。

# Authentication
# 认证（Authentication）

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*JRdO5b8j5YajBw2tBBrHZw.png)

While we tried many solutions: like building a custom auth for every provider, but **nothing comes close to firebase** if you want to implement social login into your app(e.g. SocialFi). It’s an easy, yet powerful solution for login allowing you to authenticate with **10+ providers** by writing only a couple of lines of code.
虽然我们尝试了许多解决方案：比如为每个提供商构建一个定制的认证系统，但如果你想在你的应用程序（例如SocialFi）中实现社交登录，**没有什么能比得上firebase**。这是一个简单而强大的登录解决方案，允许你仅通过编写几行代码就能实现对**10多个提供商**的认证。

For example, anyone who worked with Twitter Auth API will agree that doing it is quite tricky, but firebase simplifies it greatly. Also firebase has a nice dashboard to see and moderate your users, built-in email & sms verification and super simple session management
例如，任何使用过Twitter Auth API的人都会认为这个非常复杂，但firebase大大简化了此过程。此外，firebase还提供了一个漂亮的仪表板，用于查看和管理用户，它内置了电子邮件和短信验证功能，还有非常简单的会话管理功能。

# Account abstraction
# 账户抽象（Account Abstraction）

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*0MdBZhMx803Toq_HE9fDcQ.jpeg)

MetaKeep — our favorite account abstraction SDK
MetaKeep — 我们最喜欢的账户抽象SDK

If you want to leverage account abstraction: [**MetaKeep**](https://metakeep.xyz/) is our favorite solution by far. We’ve tried many Account Abstraction SDKs
如果你想利用账户抽象（Account Abstraction），到目前为止，[**MetaKeep**](https://metakeep.xyz/)是我们最喜欢的解决方案。我们曾经尝试过许多账户抽象的SDK。

-   Biconomy
-   Safe
-   Web3Auth

**But they all do to much: handling actual authentication logic should be your concern.**
**但它们都做得太多了：实际上身份验证逻辑才应该是你的关注点。**

> _Account Abstraction SDK should do just one thing: create a wallet from user’s data(e.g. email)_
> **账户抽象SDK应该制只做一件事：根据用户数据（例如:电子邮件）创建钱包**

To make your app as failure resistant as possible, it’s best to handle authentication yourself, and let the SDK only handle account abstraction. Which is exactly how MetaKeep does it: they have a “create wallet” [endpoint](https://docs.metakeep.xyz/reference/v3getwallet) which creates a unique wallet based on user email. That’s it! - an effortless experience that MetaKeep has multiple patent protections on.
为了使你的程序尽可能具备容错性，最好自己处理身份验证逻辑，并让SDK仅处理账户抽象。这正是MetaKeep的做法：他们有一个“创建钱包”（create wallet）的[端点（endpoint）](https://docs.metakeep.xyz/reference/v3getwallet)，可以根据用户的电子邮件创建一个唯一的钱包。仅此而已！- 一个MetaKeep拥有多项专利保护的轻松体验。

# KYC
# 身份验证（KYC）

Regulations are coming to web3, whether you want it or not. Plus, KYC offers a way to prove that your userbase is legit, protect it from bots and more. But… **how do you make KYC in web3?**
无论您是否愿意，web3都将受到监管。KYC提供了一种验证用户群体合法性、保护其免受机器人攻击等的方式。但是...**在Web3中如何进行KYC呢？**

> [zkMe](https://zk.me/) is a truly zero-knowledge web3 identity solution, allowing you to KYC without storing ANY of the users’ data. They also have an anti-bot and a profiling suite
> [zkMe](https://zk.me/)是一个真正的零知识web3身份解决方案，允许您在不存储任何用户数据的情况下进行KYC。他们还有一个防机器人程序和一个用户画像套件。

It’s a pretty fool proof solution even compared to traditional KYC solutions, because you don’t store any of the users data, **so there’s not risk of data theft/loss!**
即使与传统的KYC解决方案相比，这也是一个非常简单的解决方案，因为您不存储任何用户数据，**因此不存在数据被盗/丢失的风险！**

# Image and File storage
# 图片和文件存储

There’s a number of cost-efficient & speedy CDNs. We chose firebase’s [firestore](https://firebase.google.com/products/firestore), as it fits all of our needs, and comes with a dead-easy API
有许多性价比高且速度快的CDN。我们选择了firebase的[firestore](https://firebase.google.com/products/firestore)，因为它符合我们的所有需求，并且具有非常简单易用的API。

_File upload example:_
**文件上传示例:**

```
import { v4 } from 'uuid'
import admin from 'firebase-admin'

const bucket = admin.storage().bucket()

export const uploadFile = async (file) => {
const extension = file.name.substr(file.name.lastIndexOf('.') + 1)
const key = `${v4()}.${extension}`
try {

    const buffer = Buffer.from(await file.arrayBuffer())
    const fileRef = bucket.file(key)
    const resp = await fileRef.save(buffer, {
      metadata: {
        contentType: file.type
      }
    })
    
    // Construct the file URL
    const fileURL = `https://firebasestorage.googleapis.com/v0/b/${process.env.FIREBASE_PROJECT_ID}.appspot.com/o/${key}?alt=media`
    return fileURL
} catch (error) {
    console.error('Error uploading file to Firebase:', error)
    return null
    }
}
```

**There are some honorable mentions**, which might even be more cost-efficient than Firebase:
**下面这些方案也值得一提**，它们可能比firebase更具性价比：

-   [**CloudFare r2**](https://developers.cloudflare.com/r2/) — fully AWS S3 compatible API, so if you find yourself limited by r2 you can switch easily. R2 is cheaper than firestore and AWS S3. It might be one of the best options on the market since it comes from a known player in the cloud industry — CloudFare, but is also quite cheap
-   [**CloudFare r2**](https://developers.cloudflare.com/r2/) — 完全兼容AWS S3的API，如果你发现自己受到R2的限制，可以轻松切换到AWS S3。R2的价格比firestore和AWS S3更便宜。它可能是市场上最好的选择之一，因为它来自云计算行业中的知名公司 — Cloudflare，而且价格也相当便宜。
-   [Bunny](https://bunny.net/)
-   [Bunny](https://bunny.net/)

# Database
# 数据库

That’s one of the most opinionated choices, but our general, allbeit somewhat obvious guidelines are:
这是仁者见仁的选择，但我们的指导原则，也是最显而易见的原则是：

-   NoSQL — where you need the read/write speed and don’t have many relations, for example — a web based messenger. Our NoSQL DB of choice is **Mongo**!
-   NoSQL — 适用于需要快速读写并且没有太多关系型数据的场景，例如：一个基于网络的即时通讯工具。我们选择的NoSQL数据库是Mongo！
-   SQL — for relations and queries, can work for analytics and systems with somewhat simple relations — an NFT marketplace. Here we chose **AWS’s AuraDB or equivalent**
-   SQL — 适用于需要处理关系和查询的场景，可以用于分析和关系相对简单的系统 — 如NFT市场。在这里我们选择**AWS的AuraDB或类似产品**
-   Graph Databases — for complex relations and queries with extremely interconnected data, _although works fine for simpler relationships as well. W_e mostly leverage it in SocialFi. **Neo4j** is our tool of choice here
-   图数据库 — 适用于复杂的关系和具有高度关联数据的查询，同时对于更简单的关系处理表现也不错。我们主要在SocialFi中利用它。我们选择的是**Neo4j**。
-   In-memory DBs — used for sessions. **Redis** is our go-to-choice here
-   内存数据库 — 用于会话管理。**Redis**是我们的首选。
  
# Architecture & Folder structure
# 架构和文件夹结构（Architecture & Folder structure ）

Whatever your framework and library choices are — folder structure is what makes or breaks the app. We try to stick to a minimalist, yet versatile and easy to use structure.
无论你选择哪种框架和库 — 文件夹结构是决定应用成败的关键。我们尽量坚持使用简约、多功能且易于使用的结构。

```
.  
└── src/  
    ├── entities/  
    │   ├── user/  
    │   │   ├── mutations.ts  
    │   │   ├── queries.ts  
    │   │   ├── service.ts  
    │   │   ├── db.ts  
    │   │   ├── type.ts  
    │   │   └── index.ts  
    │   └── item  
    ├── generated/  
    │   ├── schema.graphql  
    │   └── typings.ts  
    ├── util/  
    │   ├── auth/  
    │   │   └── firebase.ts <- For handling authentication  
    │   └── random.ts <- Example util for generating a random string  
    ├── context.ts <- For handling request context, like auth  
    ├── index.ts  
    ├── schema.ts <- Main file for the graphql schema  
    └── driver.ts <- Database driver
```
Instead of making the folders based on functionality, I split them by domain allowing to see all of the data related to an entity easier.
我没有根据功能来划分文件夹，而是按照域来分，这样可以更容易地查看与实体相关的所有数据。

At the top of _src/_ is _entities/_ and each entity has
src/目录里第一个是entities/，每个entity都有

a _db.ts_ — all the database logic
**db.ts** — 所有数据库逻辑

``` 
export async function getUser(session, userId: string) {  
    try {  
      //This example uses neo4j but any db is applicable  
      const result = await session.run(  
        'MATCH (u:User {\_id: $userId}) RETURN u',  
        {  
          userId  
        }  
      )  
      const user = result.records\[0\].get('u').properties  
      if(!user) {  
        //ERROR is our enum containing errors as codes for easier handling on the frontend  
        throw new GraphQLError('User not found', { extensions: { code: ERROR.NOT\_FOUND}})  
      }  
    } catch (e) {  
      console.log(e)  
    } finally {  
      session.close()  
    }  
  }
```

_type.ts_ — main file, describing our entity graphql type
**type.ts** — 主文件，描述我们实体的graphql类型

```
    export const User = objectType({  
      name: 'User',  
      definition(t) {  
        t.nonNull.id('\_id')  
        t.nonNull.string('name')  
        //other fields  
      }  
    }
```

_queries.ts_ — graphql queries
**queries.ts** — graphql查询

```
  import { getUser } from './db'  
  
  const getUserById = queryField('getUserById', {  
    type: 'User',  
    args: {  
      id: nonNull(stringArg())  
    }  
    resolve: async (\_parent, { id }, ctx) => {  
      const session = ctx.driver.session()  
      return getUser(session, id)  
    }  
  })
```

_mutations.ts_ — graphql mutations
**mutations.ts** — graphql更新

```  
export const editProfile = mutationField('editProfile', {  
    type: 'User',  
    args: {  
      name: nonNull(stringArg())  
    },  
    resolve: async (\_parent, { name }, ctx) => {  
      //Some logic  
    }  
  })
```

_service.ts_ — any external api call, or other “service” layer logic can be put there
**service.ts** —任何外部api调用或其他“服务”层逻辑都可以写在这个文件

```
export const fetchIsUserVerified() {  
    //Some network logic, e.g. fetching an external API like zkMe you use for user verification  
  }
```

_index.ts_ — all of the exports
**index.ts** — 所有出口

```
import { userMutations } from './mutation'  
import { userQueries } from './queries'  
import { User } from './type'  
  
export const userTypes = \[  
  User,  
  ...userQueries,  
  ...userMutations  
\]
```

And finally — _src/schema.ts_ combines all the entities in one schema
最后 — src/schema.ts 将所有实体合并在一个schema中。

```
export const schema = makeSchema({  
  types: \[...userTypes, ...otherEntityTypes, ...etc\],  
  outputs: {  
    schema: \`${\_\_dirname}/generated/schema.graphql\`,  
    typegen: \`${\_\_dirname}/generated/typings.ts\`  
  }  
})
```

_context.ts_ — we mostly use it for handling authentication, so that every mutation/query can have access to the currently authenticated user
**context.ts** — 我们主要在这个文件中处理身份验证，以便每个更新/查询都可以访问当前经过身份验证的用户。

```
  export interface Context {  
    currentUser: VerifyPayload,  
  }  
  
  export async function createContext(initialContext: YogaInitialContext): Promise<Context> {  
    return {  
      //Some function getting the token from header/cookies of the request and returning the signed-in user based on that token  
      currentUser: await verify(  
        driver,  
        initialContext.request,  
      )  
    }  
  }
```

_index.ts_ — setting up GraphQL Yoda
**index.ts** — 设置 GraphQL Yoda

```  
const yoga = createYoga({  
    schema,  
    context: createContext,  
    plugins: \[  
      EnvelopArmorPlugin,  
      useCookies(),  
      fieldAuthorizePlugin()  
      useResponseCache({  
         // cache based on the authentication header(can also cache via a cookie)  
         session: (request) => request.headers.get('authentication')  
      })  
    \]  
  })
  
  // Pass it into a server to hook into request handlers.  
  const server = createServer(yoga)  
  
  // Start the server and you're done!  
  const port = process.env.PORT || 4000  
  server.listen(port, () => {  
    console.info(\`Server is running on ${port}\`)  
  })
```

# Code formatting, linting & build tools
# 代码格式化、静态代码分析和构建工具（Code formatting, linting & build tools）

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*_sXfCaLlx-ifnH1gofgLEg.png)

We go with the industry standard
我们遵循行业规范

-   **Eslint with airbnb config**
-   **用eslint配置airbnb**
-   **editorconfig for the indendation settings**
-   **用editorconfig规范代码缩进**
-   **Husky for pre-commit hooks**
-   **用Husky配置前置提交钩子（pre-commit hooks）**

_Leveraging prettier can be useful, but in my mind it adds more overhead and issues that it solves, like rules conflicting with prettier_
**prettier可能是有用的，但在我看来，它带来的额外问题要远大于它所解决的问题，比如规则冲突。**

For build we use a combination of [**esbuild**](https://esbuild.github.io/) and [**ts-node**](https://www.npmjs.com/package/ts-node), ts-node is the recommended approach by graphql yoga, so we use it in development. But to get that extra optimizations for production we leverage esbuild
在构建方面，我们同时使用了[**esbuild**](https://esbuild.github.io/)和[**ts-node**](https://www.npmjs.com/package/ts-node)，ts-node是graphql yoga推荐的开发方式，因此我们在开发环境中使用它。但是为了在生产环境的性能优化，我们又用了esbuild。

# Cloud Deployment
# 云部署

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*DTgL8dg1c4CyWGoCeQFJVQ.png)

When it comes to more complex backends we choose GCP or AWS, but when we need to push out the MVP as soon as possible — **Railway is our best friend**. It’s like Vercel but for the backend, **combining fast performance with seamless deploys and out of the box CI/CD**. No need to write hundreds of IaC lines, Railway does everything for you like enabling PR environments, so you can easily share your work with the team before it’s deployed anywhere.
在处理更复杂的后端时，我们会选择GCP或AWS，但当我们需要尽快推出MVP时，Railway是我们最好的选择。它就像后端的Vercel，同时具备了性能快、无缝部署以及开箱即用的CI/CD。不需要编写数百行IaC，Railway帮你做好了一切，比如启用PR环境，这样在你的服务正式部署之前都可以轻松地与团队分享你的工作。

> However, Railway isn’t as mature as GCP, so that’s something to keep in mind if you are expecting a lot of load on launch day
> 不过，Railway的成熟度不如GCP，所以如果你预期在上线初期就有很大负载的话，需要慎重考虑这一点。

# The end :)
# 最后 :)

If you read so far — thank you, it was a huge one! We’d love to hear your thoughts in the comments, how do you architecture your apps? If you have any questions, or need help building an awesome blockchain project — reach out to us and we’d be happy to help :)
如果你已经读到这里了——非常感谢，这真是一篇长文！我们想在评论中听到你的想法，你是如何构建你的应用程序的？如果你有任何问题，或者需要帮助构建一个很棒的区块链项目——请联系我们，我们将很乐意提供帮助 :)