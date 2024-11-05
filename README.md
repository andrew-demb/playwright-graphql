# Playwright-graphql
This library provide playwright integration with graphql and typescript for efficient API tests.
It helps you to get autogenerated graphql api client with autocomplete feature.

![DEMO](docs/gql-autocomplete-demo.gif)

To build graphql client, this library includes several graphql generators like: 

- [get-graphql-schema](https://www.npmjs.com/package/get-graphql-schema) to generate schema.
- [gql-generator](https://www.npmjs.com/package/gql-generator) to generate operations (queries and mutations).
- [@graphql-codegen/cli and @graphql-codegen/typescript-generic-sdk](https://the-guild.dev/graphql/codegen/plugins/typescript/typescript-generic-sdk) 
takes as input codegen config, generates ts file with all types and operations (without api client).

## Project setup:

1. Installation.
2. Generate schema and operations.
3. Generate typescript types.
4. Add graphql client fixture.
5. Wright graphql tests with joy!

#### Installation
- `npm install playwright-graphql`
or for dev dependency
- `npm install -D playwright-graphql`

#### Generate graphql schema from server under test
- `get-graphql-schema https://${baseUrl}/api/graphql > schema.gql`

Pay attention that url should have graphql endpoint, it may be custom for your project.
Now you should be able to find schema.gql file in your working directory.

#### Generate operations from schema
- `gqlg --schemaFilePath ./schema.gql --destDirPath ./gql/autogenerated-operations`

New directory with `.gql` files generated *gql/autogenerated-operations/*.

#### Create codegen.ts file
Create codegen.ts file:
```ts
import type { CodegenConfig } from '@graphql-codegen/cli';

const config: CodegenConfig = {
  overwrite: true,
  schema: './schema.gql',
  documents: [
    'gql/autogenerated-operations/**/*.gql',
  ],
  generates: {
    'gql/graphql.ts': {
      plugins: ['typescript', 'typescript-operations', 'typescript-generic-sdk'],
      config: {
        scalars: {
          BigInt: 'bigint|number',
          Date: 'string',
        },
      },
    },
  },
};

export default config;
```

#### Generate types

- `graphql-codegen --config codegen.ts` generates graphql.ts in gql directory. 

In case you need to customize output check [docs](https://the-guild.dev/graphql/codegen/plugins/typescript/typescript).

#### Add path to your tsconfig.json

Add `"@gql": ["gql/graphql"]` for easy import.

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "strict": true,
    "noUnusedLocals": false,
    "noUnusedParameters": false,
    "noFallthroughCasesInSwitch": true,
    "allowSyntheticDefaultImports": true,
    "baseUrl": "./",
    "paths": {
      "@fixtures/*": ["fixtures/gql"],
      "@gql": ["gql/graphql"]
    }
  }
}
```

#### Create gql fixture

*fixtures/gql.ts*
```ts
import { test as baseTest, expect, request, APIRequestContext } from '@playwright/test';
import { getSdkRequester } from 'playwright-graphql';
import { getSdk } from '@gql';

export { expect };

const getClient = (apiContext: APIRequestContext) => getSdk(getSdkRequester(apiContext));

type WorkerFixtures = {
    gql: ReturnType<typeof getClient>;
};

export const test = baseTest.extend<{}, WorkerFixtures>({
    gql: [
        async ({}, use) => {
            const apiContext = await request.newContext({
                baseURL: 'http://localhost:4000'
            });
            await use(getClient(apiContext));
        }, { auto: false, scope: 'worker' }
    ]
});
```

#### Now you are ready jump into writing tests!

*tests/example.test*
```ts
import { test, expect } from "@fixtures/gql";

test('playwright-graphql test', async ({ gql }) => {
    const res = await gql.getCityByName({
        name: 'Lviv'
    });

    expect(res.getCityByName).not.toBeNull();
})
```

---

#### More about configuration
In case you need to create custom operations for tests
add one more path to documents section in *codegen.ts* file
```ts
import type { CodegenConfig } from '@graphql-codegen/cli';

const config: CodegenConfig = {
  overwrite: true,
  schema: './schema.gql',
  documents: [
    'gql/autogenerated-operations/**/*.gql',
    'gql/custom-operations/**/*.gql',
  ],
  generates: {
    'gql/graphql.ts': {
      plugins: ['typescript', 'typescript-operations', 'typescript-generic-sdk'],
      config: {
        scalars: {
          BigInt: 'bigint|number',
          Date: 'string',
        },
      },
    },
  },
};

export default config;
```

And it is recommended to put all dynamic generated files and directories into *.gitignore*.
```gitignore
gql/autogenerated-operations
gql/**/*.ts
gql/**/*.js
```

#### Graphql explorer
You can use [apollo explorer](https://studio.apollographql.com/sandbox/explorer) to create custom queries.

#### Code generation scripts in package.json
```json
  "scripts": {
    "generate:schema": "get-graphql-schema http://localhost:4000/api/graphql > schema.gql",
    "generate:operations": "gqlg --schemaFilePath ./schema.gql --destDirPath ./gql/autogenerated-operations --depthLimit 6",
    "generate:types": "graphql-codegen --config codegen.ts",
    "codegen": "npm run generate:schema && npm run generate:operations && npm run generate:types",
    "test": "npx playwright test"
  },
```

Template project: https://github.com/DanteUkraine/playwright-graphql-example