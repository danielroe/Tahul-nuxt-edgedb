# Nuxt EdgeDB

[![npm version][npm-version-src]][npm-version-href]
[![npm downloads][npm-downloads-src]][npm-downloads-href]
[![License][license-src]][license-href]
[![Nuxt][nuxt-src]][nuxt-href]

[Nuxt 3](https://nuxt.com) integration for [EdgeDB](https://www.edgedb.com) that aims at being the fastest way to add a fully typed database layer to your Nuxt 3 project.

- [✨ &nbsp;Release Notes](/CHANGELOG.md)

## Features

- 🍱 &nbsp;Zero-config setup; just add `nuxt-edgedb` to your `modules`
- 🧙 &nbsp;[EdgeDB CLI install](https://www.edgedb.com/docs/cli/index) and [project init](https://www.edgedb.com/docs/cli/edgedb_project/edgedb_project_init) wizards
- 🎩 &nbsp;Watchers on `dbschema/*`, `queries/*`, `dbschema/migrations/*` bringing _HMR_ to your database
- 🛟 &nbsp;Auto-imported typed database query client provided by [`@edgedb/generate`](https://www.edgedb.com/docs/clients/js/generation)
- 🍩 &nbsp;[`edgedb ui`](https://www.edgedb.com/docs/cli/edgedb_ui) injected into [Nuxt Devtools](https://github.com/nuxt/devtools)

## Quick Setup

1. Add `nuxt-edgedb` dependency to your project

```bash
# Using pnpm
pnpm add -D nuxt-edgedb

# Using yarn
yarn add --dev nuxt-edgedb

# Using npm
npm install --save-dev nuxt-edgedb
```

2. Add `nuxt-edgedb` to the `modules` section of `nuxt.config.ts`

```js
export default defineNuxtConfig({
  modules: [
    'nuxt-edgedb'
  ],
  // Optional, all options has sufficient defaults.
  edgedb: {
    devtools: true,
    watch: true,
    dbschemaDir: 'dbschema',
    queriesDir: 'queries',
    generateInterfaces: true,
    generateQueries: true,
    generateQueryBuilder: true,
  }
})
```

That's it! You can now use Nuxt EdgeDB in your Nuxt app. ✨

If you do not already have [EdgeDB](https://www.edgedb.com) installed on your machine, the install wizard will prompt you to install it. 🧙

## Usage

The module provides 2 auto-imported composables available anywhere inside `server/` context of your Nuxt app.

### useEdgeDb

`useEdgeDb` exposes the raw client from `edgedb` import, that properly uses your Nuxt environment configuration.

```typescript
// server/api/blogpost/[id].ts
import { defineEventHandler, getRouterParams } from 'h3'

export default defineEventHandler(async (req) => {
  const params = getRouterParams(req)

  const id = params.id

  const client = useEdgeDb()

  const blogpost = await client.querySingle(`
    select BlogPost {
      title,
      description
    } filter .id = ${id}
  `)

  return blogpost
})
```

### useEdgeDbQueries

`useEdgeDbQueries` exposes all your queries from `dbschema/queries.ts` except you do not need to pass them a client.

They will use the one generated by `useEdgeDb` and is scoped to the current request.

```esdl
// queries/getBlogPost.edgeql
select BlogPost {
  title,
  description
} filter .id = <uuid>$blogpost_id
```

```typescript
// server/api/blogpost/[id].ts
import { defineEventHandler, getRouterParams } from 'h3'

export default defineEventHandler(async (req) => {
  const params = getRouterParams(req)

  const id = params.id

  const { getBlogpPost } = useEdgeDbQueries()

  const blogPost = await getBlogpost({ blogpost_id: id })

  return blogpost
})
```

You can still import [queries](https://www.edgedb.com/docs/clients/js/queries) directly from `@db/queries` and pass them the client from `useEdgeDb()`.

```typescript
// server/api/blogpost/[id].ts
import { getBlogPost } from '@db/queries'
import { defineEventHandler, getRouterParams } from 'h3'

export default defineEventHandler(async (req) => {
  const params = getRouterParams(req)

  const id = params.id

  const client = useEdgeDb()

  const blogPost = await getBlogpost(client, { blogpost_id: id })

  return blogpost
})
```

### useEdgeDbQueryBuilder

`useEdgeDbQueryBuilder` exposes the generated [query builder](https://www.edgedb.com/docs/clients/js/querybuilder) directly to your `server/` context.

```typescript
// server/api/blogpost/[id].ts
import { defineEventHandler, getRouterParams } from 'h3'

export default defineEventHandler(async (req) => {
  const params = getRouterParams(req)

  const id = params.id

  const client = useEdgeDb()
  const e = useEdgeDbQueryBuilder()

  const blogPostQuery = e.select(
    e.BlogPost,
    (blogPost) => ({
      id: true,
      title: true,
      description: true,
      filter_single: { id }
    })
  )

  const blogPost = await blogPostQuery.run(client)

  return blogpost
})
```

### Typings

All the interfaces generated by EdgeDB are available throuh imports via `@db/interfaces`.

```vue
<script setup lang="ts">
import type { BlogPost } from '@db/interfaces'

defineProps<{ blogPost: BlogPost }>()
</script>
```

## Production

If you want to get out of development and deploy your database to prodution, you must follow [EdgeDB guides](https://www.edgedb.com/docs/guides/deployment/index).

[EdgeDB](https://www.edgedb.com) is an open-source database that is designed to be self-hosted.

However, they also offer a [Cloud](https://www.edgedb.com/docs/guides/cloud), which is fully compatible with this module thanks to environment variables.

## Q&A

### Will my database client be exposed in userland?

No, `useEdgeDb` and `useEdgeDbQueries` are only available in [server/](https://nuxt.com/docs/guide/directory-structure/server) context of Nuxt.

You can, as an **opt-in** feature, import queries from `@dbschema/queries` on the client.

You will need to provide these queries with a client from `createClient()`.

```vue
<script setup lang="ts">
import { createClient } from 'edgedb'
import { getUser } from '@dbschema/queries'

const client = createClient()

const user = await getUser(client, 42)
</script>
```

You can also, still as an **opt-in** feature, import the query builder to the client.

I guess that can be useful for a super-admin/internal dashboard, but use it at your own risks in terms of security access.

```vue
<script setup lang="ts">
import e, { type $infer } from '@db/builder'

const query = e.select(e.Movie, () => ({ id: true, title: true }));
type result = $infer<typeof query>;
//   ^ { id: string; title: string }[]
</script>
```

Be careful with these imports, as if you import wrong queries, you might end up with write operations available to the client, potentially exposing your database.

### How do I run my migrations in production?

- Clone your Nuxt project on your production environment
- Ensure you have [EdgeDB CLI](https://www.edgedb.com/docs/cli/index) installed on the server
- Add `edgedb migrate --quiet` to your CLI script

### Should I version generated files?

No, as they are generated with your Nuxt client, you should add them to your `.gitignore`

```.gitignore
**/*.edgeql.ts
dbschema/queries.*
dbschema/query-builder
```

You must change these paths accordingly if you change the `**Dir` options.

### Is HMR for my database schema really safe?

Well, it depends on when you want to use it.

I would suggest keeping `watchPrompt` enabled while you casually dev on your project.

That will prevent from running any unwanted migration, and will only prompt when you add new things to your schemas.

If you want to go fast and know what you are doing, you can set `watchPrompt` to false, and profit from automatic migration creation and applying on any change on your schemas.

If you do not want any of these features, just set `watch` to false and you are free to feel safe about changes applied to your development database.

> HMR on your database obviously has **NO** effect in production context.

## Development

```bash
# Install dependencies
npm install

# Generate type stubs
npm run dev:prepare

# Develop with the playground
npm run dev

# Build the playground
npm run dev:build

# Run ESLint
npm run lint

# Run Vitest
npm run test
npm run test:watch

# Release new version
npm run release
```

<!-- Badges -->
[npm-version-src]: https://img.shields.io/npm/v/nuxt-edgedb/latest.svg?style=flat&colorA=18181B&colorB=28CF8D
[npm-version-href]: https://npmjs.com/package/nuxt-edgedb

[npm-downloads-src]: https://img.shields.io/npm/dm/nuxt-edgedb.svg?style=flat&colorA=18181B&colorB=28CF8D
[npm-downloads-href]: https://npmjs.com/package/nuxt-edgedb

[license-src]: https://img.shields.io/npm/l/nuxt-edgedb.svg?style=flat&colorA=18181B&colorB=28CF8D
[license-href]: https://npmjs.com/package/nuxt-edgedb

[nuxt-src]: https://img.shields.io/badge/Nuxt-18181B?logo=nuxt.js
[nuxt-href]: https://nuxt.com
