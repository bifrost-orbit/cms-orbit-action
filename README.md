# CMS Orbit types for GitHub Actions

Keep your website's TypeScript types in step with your [CMS Orbit](https://bifrostorbit.com/products/cms-orbit/) space. The action runs `cms-orbit generate` from [`@bifrost-orbit-cms/sdk`](https://www.npmjs.com/package/@bifrost-orbit-cms/sdk), which reads your space's components and content types and writes a type for each. Use it on a schedule to open a pull request when editors change the schema, or on every pull request to catch types that have fallen behind.

It needs nothing installed in your repository: it sets up Node.js and runs the SDK with `npx`.

## Before you start

1. **Create a public key.** In CMS Orbit, open **Configuration › API keys** and create a public key for the environment your website reads, such as Production.
2. **Add it to GitHub.** In your repository, open **Settings › Secrets and variables › Actions** and add the key as the secret `CMS_PUBLIC_KEY`. Add the API address, `https://api.bifrostorbit.com/cms`, as the variable `CMS_API_URL`.

## Keep types in sync on a schedule

Every morning this checks the Northwilds space for new or changed components and, when the types differ, opens a pull request with the new `src/cms/types.ts`. Running it again updates the same pull request.

```yaml
name: CMS types

on:
  schedule:
    - cron: "0 6 * * 1-5"
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  types:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - name: Generate types from CMS Orbit
        id: types
        uses: bifrost-orbit/cms-orbit-action@v1
        with:
          api-url: ${{ vars.CMS_API_URL }}
          public-key: ${{ secrets.CMS_PUBLIC_KEY }}
          output: src/cms/types.ts

      - name: Open a pull request
        if: steps.types.outputs.changed == 'true'
        uses: peter-evans/create-pull-request@v8
        with:
          branch: cms-orbit/types
          commit-message: Update the types from CMS Orbit
          title: Update the types from CMS Orbit
          body: The components or content types in the Northwilds space have changed, so `${{ steps.types.outputs.types-path }}` has been generated again.
          add-paths: ${{ steps.types.outputs.types-path }}
```

To let GitHub Actions open pull requests, turn on **Allow GitHub Actions to create and approve pull requests** under **Settings › Actions › General**. Pull requests opened with the built-in `GITHUB_TOKEN` don't start other workflows, so if you want your checks to run on them, give `create-pull-request` a `token` from a GitHub App or a fine-grained token.

## Check types on every pull request

This fails the pull request when `src/cms/types.ts` no longer matches the space, and shows the difference in the log, so nobody merges code written against fields that have moved on.

```yaml
name: CMS types

on:
  pull_request:

permissions:
  contents: read

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - name: Check the types against CMS Orbit
        uses: bifrost-orbit/cms-orbit-action@v1
        with:
          api-url: ${{ vars.CMS_API_URL }}
          public-key: ${{ secrets.CMS_PUBLIC_KEY }}
          output: src/cms/types.ts
          check: true
```

When they differ, the run fails with:

> The types in src/cms/types.ts are out of date with CMS Orbit. Run cms-orbit generate and commit the result.

Run `npx cms-orbit generate --out src/cms/types.ts` on your computer and commit the file to fix it. Pull requests from forks don't receive your secrets, so the check fails there until someone with access runs it.

## Inputs

| Input | Required | Default | What it does |
| --- | --- | --- | --- |
| `api-url` | Yes | | The CMS Orbit API address, `https://api.bifrostorbit.com/cms` |
| `public-key` | Yes | | A public `cms_pk_` key for the space. Keep it in a secret. |
| `output` | No | `src/cms/types.ts` | The types file to write or check |
| `check` | No | `false` | `true` fails the run when the file differs from the space, without changing it |
| `node-version` | No | `24` | The Node.js version to set up |
| `sdk-version` | No | `latest` | The version of `@bifrost-orbit-cms/sdk` to run, such as `0.13.1`. Pin it to the version your website uses so the types match. |

## Outputs

| Output | What it holds |
| --- | --- |
| `changed` | `true` when the generated types differ from the file in the repository, otherwise `false` |
| `types-path` | The path of the types file, the same as `output` |

## Permissions

The action only reads your repository and the space's schema, so `contents: read` is enough for checking. To open pull requests with the types, the workflow needs `contents: write` and `pull-requests: write`, as in the first example. In CMS Orbit, a public key without **Read drafts** is all the action needs.

## More

- [The SDK](https://docs.bifrostorbit.com/cms/developers/sdk/) explains `cms-orbit generate` and the types it writes.
- [Field types](https://docs.bifrostorbit.com/cms/developers/field-types/) shows the type each field becomes.

## Licence

MIT. See [LICENSE](LICENSE).
