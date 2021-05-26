---
title: GraphQL 实践(1)
date: 2021-05-24 16:29:00
tags: GraphQL
---

## overview

> GraphQL is the rising star of backend technologies. It replaces REST as an API design paradigm and is becoming the new standard for exposing the data and functionality of a web server. is the rising star of backend technologies. It replaces REST as an API design paradigm and is becoming the new standard for exposing the data and functionality of a web server.

其实就是日常都是用的RESTFul API，感觉有点过于单调了，就想来尝试一下GraphQL

## using packages

当然用的是`nodejs`啦，还有下面几个👇

> * `Apollo server`:Fully-featured GraphQL Server with focus on easy setup, performance and great developer experience, graphql-js and more.
> * `Prisma`: Replaces traditional ORMs. Use Prisma Client to access your database inside of GraphQL resolvers.
> * GraphQL Playground: A “GraphQL IDE” that allows you to interactively explore the functionality of a GraphQL API by sending queries and mutations to it. It’s somewhat similar to Postman which offers comparable functionality for REST APIs.

## goal

实现一个简单的[hacker news](https://news.ycombinator.com/news) ，包含CRUD，注册登陆等基本功能

废话不多说，Let’s get started 🚀

## get started

```json
// package.json
{
  "name": "hackernews-node",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "dependencies": {
    "apollo-server": "^2.24.1",
    "graphql": "^15.5.0"
  }
}
```

```javascript
// src/main.js
const {ApolloServer} = require('apollo-server');
// 定义数据格式
const typeDefs = `
  type Query {
    info: String!
  }
`
// 具体的数据操作放在resolvers里面，eg:CRUD
const resolvers = {
  Query: {
    info: () => `This is the API of a Hackernews Clone`
  }
}
// 把typeDefs和resolvers传给ApolloServer
const server = new ApolloServer({
  typeDefs,
  resolvers,
})
server
  .listen()
  .then(({url}) =>
    console.log(`Server is running on ${url}`)
  );
```

### test server

在根目录运行

```bash
node src/index.js`,
```

就打开了`GraphQL Playground`，在左边输入：

```graphql
query {
    info
}
```

得到的应该是这样的结果：
![1](/images/graphQL.png)
这就是一个简单的`query`了，接下来可以考虑加上更多的查询：

```graphql
type Query{
    info:String!
    feed:[Link!]!
    link(id: ID!): Link
}
type Mutation{
    post(url:String!,description:String!):Link!
    update(id:ID!,url:String!,description:String!):Link
    delete(id:ID!):Link
}
type Link{
    id:ID!
    description:String!
    url:String!
}
```

`Query`就是查询，`feed`返回的是所有的links,而`link(id)`则是根据id返回对应的link，同理`Mutation`里面就是 增改删了,回到`index.js`:

```javascript
// src/index.js
...
let links = [{
  id: 'link-0',
  url: 'www.howtographql.com',
  description: 'Fullstack tutorial for GraphQL'
}]
let idCount = links.length
const resolvers = {
  Query: {
    info: () => 'this is the api of hackerNews!',
    feed: () => links,
    link: (parent, args) => links.find(l => l.id === args.id)
  },
  Mutation: {
    post: (parent, args) => {
      const link = {
        id: `link-${idCount++}`,
        description: args.description,
        url: args.url
      }
      links.push(link)
      return link
    },
    update: (parent, args) => {
      links = links.map(l => l.id === args.id ? args : l)
      return args
    },
    delete: (parent, args) => {
      links = links.filter(l => l.id !== args.id)
      return args
    },
  },
}
...
```

可以在playground里面尝试一下：

```graphql
mutation {
    post(url: "www.prisma.io", description: "Prisma replaces traditional ORMs") {
        id
    }
}

mutation {
    update(id:"link-0",url: "www.baidu.com", description: "this is baidu"){
        id
    }
}

mutation {
    delete(id:"link-0"){
        id
    }
}
```

这样就实现了简单的crud了，但是到现在还没有用数据库保存数据，所以现在每次重启数据都没啦， 那么下一步就需要用到`prisma` 来连接数据库了。
### add database
```
yarn add -D prisma
npx prisma init
```
在创建的`prisma/schema.prisma`文件中，修改成`sqlite`:
```prisma
datasource db {
  provider = "sqlite"
  url      = "file:./dev.db"
}

generator client {
  provider = "prisma-client-js"
}

model Link {
  id          Int      @id @default(autoincrement())
  createdAt   DateTime @default(now())
  description String
  url         String
}
```
之后就可以`migrate`和`generate`了
```
npx prisma migrate dev
npx prisma generate
```
这样数据库就准备好了，下一步开始连接。
### connecting server and database

在GraphQL的resolver函数里面接受的是4个参数，第三个就是`context`, 就和字面意思一样，它是不同resolver之间沟通的桥梁

```javascript
const {PrismaClient} = require('@prisma/client')
const prisma = new PrismaClient()

const server = new ApolloServer({
  typeDefs: fs.readFileSync(
    path.join(__dirname, 'schema.graphql'),
    'utf8'
  ),
  resolvers,
  context: {
    prisma,
  }
})
```

然后，就能通过`context.prisma`在所有的resolver里面访问到了:

```javascript
const resolvers = {
  Query: {
    info: () => `This is the API of a Hackernews Clone`,
    feed: async (parent, args, context) => {
      return context.prisma.link.findMany()
    },
  },
  Mutation: {
    post: (parent, args, context, info) => {
      const newLink = context.prisma.link.create({
        data: {
          url: args.url,
          description: args.description,
        },
      })
      return newLink
    },
  },
}
```

关于`prisma`的CRUD，可以看看 [官方文档](https://www.prisma.io/docs/concepts/components/prisma-client/crud)

然后可以运行`npx prisma studio`在 [http://localhost:5555](http://localhost:5555) 查看到所有的数据

### Authentication
第一件事，需要添加一个User model，并且将其与Link关联：
```prisma
model Link {
  id          Int      @id @default(autoincrement())
  createdAt   DateTime @default(now())
  description String
  url         String
  postedBy    User?    @relation(fields: [postedById], references: [id])
  postedById  Int?
}

model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  password  String
  links     Link[]
}
```
接着重新运行`migrate`和`generate`，是不是感觉有点蠢，每次修改prisma都需要运行一下下面两个命令，在之后会把它做成自动化的，
现在就先重新运行一下:
```
npx prisma migrate dev --name "add-user-model"
npx prisma generate
```
在`src/schema.graphql`里面添加signUp和logIn，同时在Link里面添加`postedBy`使其与User关联：
```graphql
type User{
    id:ID!
    name:String!
    email:String!
    links:[Link!]!
}
type AuthPayload {
    token:String
    user:User
}

type Query{
    feed:[Link!]!
    link(id: ID!): Link
}
type Mutation{
    post(url:String!,description:String!):Link!
    updatePost(id:ID!,url:String!,description:String!):Link
    deletePost(id:ID!):Link
    signUp(email:String!,name:String!,password:String!):AuthPayload
    logIn(email:String!,password:String!):AuthPayload
}

type Link{
    id:ID!
    description:String!
    url:String!
    postedBy:User!
}
```
因为操作逐渐变得复杂起来，所以需要重构一下，把对应的方法分离：
```
mkdir src/resolvers
touch src/resolvers/Query.js
touch src/resolvers/Mutation.js
touch src/resolvers/User.js
touch src/resolvers/Link.js
```
在对应的文件做对应的事情，具体的代码就是一些copy&paste，就不展开，主要看一下`Mutation.js`里面的`signUp`和`logIn`:

```javascript
//src.Mutation.js
async function signup(parent, args, context, info) {
  const password = await bcrypt.hash(args.password, 10)
  const user = await context.prisma.user.create({ data: { ...args, password } })
  const token = jwt.sign({ userId: user.id }, APP_SECRET)
  return {
    token,
    user,
  }
}

async function login(parent, args, context, info) {
  const user = await context.prisma.user.findUnique({ where: { email: args.email } })
  if (!user) {
    throw new Error('No such user found')
  }
  const valid = await bcrypt.compare(args.password, user.password)
  if (!valid) {
    throw new Error('Invalid password')
  }
  const token = jwt.sign({ userId: user.id }, APP_SECRET)
  return {
    token,
    user,
  }
}

module.exports = {
  signup,
  login,
}
```
关于signup，就是先hash加盐密码，然后创建user，生成 [JSON Web Token](https://github.com/auth0/node-jsonwebtoken) ，最后返回user和token;login的话，就是找user，对比密码，拿到token。

之后需要在`context`里面传入`userId`:
```javascript
// index.js
const server = new ApolloServer({
  typeDefs: fs.readFileSync(
    path.join(__dirname, 'schema.graphql'),
    'utf8'
  ),
  resolvers,
  context: ({ req }) => {
    return {
      ...req,
      prisma,
      userId:
        req && req.headers.authorization
          ? getUserId(req)
          : null
    };
  }
});
```
```javascript
// utils.js
const jwt = require('jsonwebtoken');
const APP_SECRET = 'GraphQL-is-aw3some';

function getTokenPayload(token) {
  return jwt.verify(token, APP_SECRET);
}

function getUserId(req, authToken) {
  if (req) {
    const authHeader = req.headers.authorization;
    if (authHeader) {
      const token = authHeader.replace('Bearer ', '');
      if (!token) {
        throw new Error('No token found');
      }
      const { userId } = getTokenPayload(token);
      return userId;
    }
  } else if (authToken) {
    const { userId } = getTokenPayload(authToken);
    return userId;
  }

  throw new Error('Not authenticated');
}

module.exports = {
  APP_SECRET,
  getUserId
};
```

那么对于post，就可以通过`context`拿到userId了：
```javascript
// mutation.js
async function post(parent, args, context) {
  const { userId } = context;

  return await context.prisma.link.create({
    data: {
      url: args.url,
      description: args.description,
      postedBy: { connect: { id: userId } },
    }
  })
}
```
还需要做的一件事是把`Link`里面的`postedBy`和`User`里面的`links`关联起来，否则的话graphQL不知道应该从哪里去获取数据
```javascript
// src/resolvers/Link.js
function postedBy(parent, args, context) {
  return context.prisma.link.findUnique({ where: { id: parent.id } }).postedBy()
}

module.exports = {
  postedBy,
}
```
注意，最后调用的`postedBy()`必须得叫这个名字，它需要与`schema.graphql`里面的`type Link`中的定义一致。
同样的，在User中定义links：
```javascript
// src/resolvers/User.js
function links(parent, args, context) {
  return context.prisma.user.findUnique({ where: { id: parent.id } }).links()
}

module.exports = {
  links,
}
```
最后把它们引入resolver:
```javascript
const resolvers = {
  Query,
  Mutation,
  User,
  Link
}
```
ok,可以开始测一测鉴权了！ 🔓
```
mutation {
  signup(name: "Alice", email: "alice@prisma.io", password: "graphql") {
    token
    user {
      id
    }
  }
}
```
返回的东西应该是类似于这样子:
```
{
  "data": {
    "signUp": {
      "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjMsImlhdCI6MTYyMjAxMTE1MX0.9yvVPC-9JPzeEALoaDS8sUyFDPzhvQCs30b_GI2cL_4",
      "user": {
        "id": "3"
      }
    }
  }
}
```
再试试login:
```
mutation {
  logIn(email: "alice@prisma.io", password: "graphql") {
    token
    user {
      email
      links {
        url
        description
      }
    }
  }
}
```
正确的返回值：
```
{
  "data": {
    "login": {
      "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJjanBzaHVsazJoM3lqMDk0NzZzd2JrOHVnIiwiaWF0IjoxNTQ1MDYyNTQyfQ.KjGZTxr1jyJH7HcT_0glRInBef37OKCTDl0tZzogekw",
      "user": {
        "email": "alice@prisma.io",
        "links": []
      }
    }
  }
}
```
如果两个都成功的话,就可以在HTTP HEADER里面加上,`__TOKEN__`换成上面返回的token
```
{
  "Authorization": "Bearer __TOKEN__"
}
```
然后也可以试试post,update,delete操作是否可行了～





