---
title: Web3 Backend architecture that works
authorURL: ""
originalURL: https://medium.com/@miki.digital/web3-backend-architecture-that-works-d142d7d1a92e
translator: ""
reviewer: ""
---

# Web3 Backend architecture that works

<!-- more -->

[

![MiKi Digital](https://miro.medium.com/v2/resize:fill:88:88/1*8CKQcEv0WTzCKPk550eMag.png)









][1]

[MiKi Digital][2]

·

[Follow][3]

10 min read

·

Jan 24, 2024

[

][4]

\--

3

[][5]

Listen

Share

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*TopkAP2G5EB2rFKY83akkw.png)

> Building a backend for a blockchain/web3 project is a completely different beast than in web2.

**Isn’t backned mostly web2 tech, you may ask?** While that’s partially correct, there are many tricky aspects when it comes to building backends for web3 apps:

-   **Tight timelines & budget** — you don’t have time & money to burn by hiring a devops team, configuring CI/CD and making a state-of-the-art microservice architecture. Things need to be simple, yet work flawlessly
-   **Caching** — you often need to implement caching for RPC calls to the blockchain
-   **Compliance** — since blockchain involves finance, you need to stay compliant with regulation
-   **Safety** — Properly storing your private keys for Smart Contract invocations from the backend is key

With the multitude of apps, of course there’s no one size fits all solution, but we want to share a blueprint we use and some of our go-to frameworks at [MiKi Digital][6]

# Monolith vs Microservices

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*SQ9PJ4NUQQ5UeyFJMHqqQw.png)

While from time to time we all want to imagine we are buliding the next google and create a state of the art microservice architecture, often — it’s not needed! **Unless you plan on having extreme load from the start and can spare the time and money it costs to build a proper microservice architecture, and what’s more importantly — have the expertise to do so, you shouldn’t.** Sticking with Monolith will give you good enough performance for 90% of your use cases.

> _Don’t overengineer from the start, simplicity is the key to success._

# NodeJS + TypeScript

NodeJS is the industry standard solution, even though some recent competitors might take the crown from it soon. Let’s take a look at them:

-   [**Deno**][7] — a JS runtime with first-class TypeScript support. It might be a bit faster than NodeJS but doesn’t offer interoperability, so while the runtime might be mature and better than Node, the frameworks and community aren’t as good as in Node.
-   [**Bun**][8] — a JS runtime (almost)fully interoperable with Node, built in Rust for blazing-fast speed. You can use all the frameworks & libraries that NPM has, while enjoying better performance & faster compile times. While the vision is grand, at the time of writing this article Bun isn’t fully interoperable, which can bring some downsides. But if you are reading this a couple of months later — Bun might be a worthy competitor to NodeJS. **Don’t forget to check it’s compatability with all the libraries you planning to use first**

TypeScript is just how everyone does it at this point, some argue that it pollutes the code, however, with a strict config, e.g.

{  
  "compilerOptions":{  
    "strict":false,  
    "noUnusedParameters":true,  
    "noImplicitReturns":true,  
    "noFallthroughCasesInSwitch":true,  
    "forceConsistentCasingInFileNames":true  
  }  
}

It enforces you to think how you want the data to look, making it easier for other devs to understand the code.

# GraphQL Yoga + Nexus GraphQL

The majority of developers use NestJS for working with GraphQL as it has first-class support and is dead-easy to get started with. However, does it make sense to bring a clunky BE framework into a project that needs to move fast, and work fast? Not really — NestJS comes with a ton of boilerplate code, following all those OOP, Dependency Injection, IoC, etc. patterns.

> _Just because it has been the industry standart for many years, with languages such as Java and .NET having the same approach, doesn’t mean NestJS is the simplest way to solve the problem._

Also, NestJS’s bundle size is [**20** **times biggest than of the same solution but with NodeJs and some minimalistic libraries instead**][9]**.**

So we chose to go with pure NodeJS and a few libraries which allows us to have cleaner code and develop faster!

For GraphQL we can leverage [GraphQL Yoda][10] — an amazing graphql server with a ton of out of the box features like caching, cookies, etc. Apollo Server is nice, but [**GraphQL Yoga is the exact case where we can trade a bit of maturity(GraphQL Yoda is newer than Apollo) for some performance & ease-of-use.**][11]

I already made a rather [comprehensive review of code-first vs schema-first approaches][12], in-short: **I prefer code-first**. And the perfect library for that is [nexus-graphql][13]: just a damn good library with everything you need for code-first graphql approach. We’ll show some code snippets of later in the article

# Authentication

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*JRdO5b8j5YajBw2tBBrHZw.png)

While we tried many solutions: like building a custom auth for every provider, but **nothing comes close to firebase** if you want to implement social login into your app(e.g. SocialFi). It’s an easy, yet powerful solution for login allowing you to authenticate with **10+ providers** by writing only a couple of lines of code.

For example, anyone who worked with Twitter Auth API will agree that doing it is quite tricky, but firebase simplifies it greatly. Also firebase has a nice dashboard to see and moderate your users, built-in email & sms verification and super simple session management

# Account abstraction

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*0MdBZhMx803Toq_HE9fDcQ.jpeg)

MetaKeep — our favorite account abstraction SDK

If you want to leverage account abstraction: [**MetaKeep**][14] is our favorite solution by far. We’ve tried many Account Abstraction SDKs

-   Biconomy
-   Safe
-   Web3Auth

**But they all do to much: handling actual authentication logic should be your concern.**

> _Account Abstraction SDK should do just one thing: create a wallet from user’s data(e.g. email)_

To make your app as failure resistant as possible, it’s best to handle authentication yourself, and let the SDK only handle account abstraction. Which is exactly how MetaKeep does it: they have a “create wallet” [endpoint][15] which creates a unique wallet based on user email. That’s it! - an effortless experience that MetaKeep has multiple patent protections on.

# KYC

Regulations are coming to web3, whether you want it or not. Plus, KYC offers a way to prove that your userbase is legit, protect it from bots and more. But… **how do you make KYC in web3?**

> [zkMe][16] is a truly zero-knowledge web3 identity solution, allowing you to KYC without storing ANY of the users’ data. They also have an anti-bot and a profiling suite

It’s a pretty fool proof solution even compared to traditional KYC solutions, because you don’t store any of the users data, **so there’s not risk of data theft/loss!**

# Image and File storage

There’s a number of cost-efficient & speedy CDNs. We chose firebase’s [firestore][17], as it fits all of our needs, and comes with a dead-easy API

_File upload example:_

import { v4 } from 'uuid'  
import admin from 'firebase-admin'  
  
const bucket = admin.storage().bucket()  
  
export const uploadFile = async (file) => {  
  const extension = file.name.substr(file.name.lastIndexOf('.') + 1)  
  const key = \`${v4()}.${extension}\`  
  
  try {  
    const buffer = Buffer.from(await file.arrayBuffer())  
    const fileRef = bucket.file(key)  
  
    const resp = await fileRef.save(buffer, {  
      metadata: {  
        contentType: file.type  
      }  
    })  
  
    // Construct the file URL  
    const fileURL = \`https://firebasestorage.googleapis.com/v0/b/${process.env.FIREBASE\_PROJECT\_ID}.appspot.com/o/${key}?alt=media\`  
    return fileURL  
  } catch (error) {  
    console.error('Error uploading file to Firebase:', error)  
    return null  
  }  
}

**There are some honorable mentions**, which might even be more cost-efficient than Firebase:

-   [**CloudFare r2**][18] — fully AWS S3 compatible API, so if you find yourself limited by r2 you can switch easily. R2 is cheaper than firestore and AWS S3. It might be one of the best options on the market since it comes from a known player in the cloud industry — CloudFare, but is also quite cheap
-   [Bunny][19]

# Database

That’s one of the most opinionated choices, but our general, allbeit somewhat obvious guidelines are:

-   NoSQL — where you need the read/write speed and don’t have many relations, for example — a web based messenger. Our NoSQL DB of choice is **Mongo**!
-   SQL — for relations and queries, can work for analytics and systems with somewhat simple relations — an NFT marketplace. Here we chose **AWS’s AuraDB or equivalent**
-   Graph Databases — for complex relations and queries with extremely interconnected data, _although works fine for simpler relationships as well. W_e mostly leverage it in SocialFi. **Neo4j** is our tool of choice here
-   In-memory DBs — used for sessions. **Redis** is our go-to-choice here

# Architecture & Folder structure

Whatever your framework and library choices are — folder structure is what makes or breaks the app. We try to stick to a minimalist, yet versatile and easy to use structure.

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

Instead of making the folders based on functionality, I split them by domain allowing to see all of the data related to an entity easier.

At the top of _src/_ is _entities/_ and each entity has a

_db.ts_ — all the database logic

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

_type.ts_ — main file, describing our entity graphql type

    export const User = objectType({  
      name: 'User',  
      definition(t) {  
        t.nonNull.id('\_id')  
        t.nonNull.string('name')  
        //other fields  
      }  
    }

_queries.ts_ — graphql queries

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

_mutations.ts_ — graphql mutations

  export const editProfile = mutationField('editProfile', {  
    type: 'User',  
    args: {  
      name: nonNull(stringArg())  
    },  
    resolve: async (\_parent, { name }, ctx) => {  
      //Some logic  
    }  
  })

_service.ts_ — any external api call, or other “service” layer logic can be put there

  export const fetchIsUserVerified() {  
    //Some network logic, e.g. fetching an external API like zkMe you use for user verification  
  }

_index.ts_ — all of the exports

import { userMutations } from './mutation'  
import { userQueries } from './queries'  
import { User } from './type'  
  
export const userTypes = \[  
  User,  
  ...userQueries,  
  ...userMutations  
\]

And finally — _src/schema.ts_ combines all the entities in one schema

export const schema = makeSchema({  
  types: \[...userTypes, ...otherEntityTypes, ...etc\],  
  outputs: {  
    schema: \`${\_\_dirname}/generated/schema.graphql\`,  
    typegen: \`${\_\_dirname}/generated/typings.ts\`  
  }  
})

_context.ts_ — we mostly use it for handling authentication, so that every mutation/query can have access to the currently authenticated user

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

_index.ts_ — setting up GraphQL Yoda

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

# Code formatting, linting & build tools

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*_sXfCaLlx-ifnH1gofgLEg.png)

We go with the industry standard

-   **Eslint with airbnb config**
-   **editorconfig for the indendation settings**
-   **Husky for pre-commit hooks**

_Leveraging prettier can be useful, but in my mind it adds more overhead and issues that it solves, like rules conflicting with prettier_

For build we use a combination of [**esbuild**][20] and [**ts-node**][21], ts-node is the recommended approach by graphql yoga, so we use it in development. But to get that extra optimizations for production we leverage esbuild

# Cloud Deployment

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*DTgL8dg1c4CyWGoCeQFJVQ.png)

When it comes to more complex backends we choose GCP or AWS, but when we need to push out the MVP as soon as possible — **Railway is our best friend**. It’s like Vercel but for the backend, **combining fast performance with seamless deploys and out of the box CI/CD**. No need to write hundreds of IaC lines, Railway does everything for you like enabling PR environments, so you can easily share your work with the team before it’s deployed anywhere.

> However, Railway isn’t as mature as GCP, so that’s something to keep in mind if you are expecting a lot of load on launch day

# The end :)

If you read so far — thank you, it was a huge one! We’d love to hear your thoughts in the comments, how do you architecture your apps? If you have any questions, or need help building an awesome blockchain project — reach out to us and we’d be happy to help :)

_Need blockchain help? Feel free to_ [_contact us_][22] _or_ [_book a call_][23]_._

_Written by_ [_Mikhail Kedzel_][24] _for_ [_MiKi Digital_][25]

[1]: /@miki.digital?source=post_page-----d142d7d1a92e--------------------------------
[2]: /@miki.digital?source=post_page-----d142d7d1a92e--------------------------------
[3]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fsubscribe%2Fuser%2F8510bfdf9847&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40miki.digital%2Fweb3-backend-architecture-that-works-d142d7d1a92e&user=MiKi+Digital&userId=8510bfdf9847&source=post_page-8510bfdf9847----d142d7d1a92e---------------------post_header-----------
[4]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fvote%2Fp%2Fd142d7d1a92e&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40miki.digital%2Fweb3-backend-architecture-that-works-d142d7d1a92e&user=MiKi+Digital&userId=8510bfdf9847&source=-----d142d7d1a92e---------------------clap_footer-----------
[5]: /m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2Fd142d7d1a92e&operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40miki.digital%2Fweb3-backend-architecture-that-works-d142d7d1a92e&source=-----d142d7d1a92e---------------------bookmark_footer-----------
[6]: https://miki.digital
[7]: https://deno.com/
[8]: https://bun.sh/
[9]: /@miki.digital/reducing-lambda-bundle-size-with-esbuild-and-lambda-layers-c4803f1007cc
[10]: https://the-guild.dev/graphql/yoga-server
[11]: https://the-guild.dev/graphql/yoga-server/docs/comparison#graphql-yoga-and-apollo-server
[12]: https://www.linkedin.com/posts/mikhail-kedel_javascript-graphql-backend-activity-7021491967448543232-_zKJ?utm_source=share&utm_medium=member_desktop
[13]: https://nexusjs.org/
[14]: https://metakeep.xyz/
[15]: https://docs.metakeep.xyz/reference/v3getwallet
[16]: https://zk.me/
[17]: https://firebase.google.com/products/firestore
[18]: https://developers.cloudflare.com/r2/
[19]: https://bunny.net/
[20]: https://esbuild.github.io/
[21]: https://www.npmjs.com/package/ts-node
[22]: https://www.miki.digital/contact
[23]: https://calendly.com/miki-digital/15min
[24]: https://www.linkedin.com/in/mikhail-kedel/
[25]: https://www.miki.digital/