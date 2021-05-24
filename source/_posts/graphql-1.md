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

