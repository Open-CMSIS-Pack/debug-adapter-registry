# debug-adapter-registry

[![License](https://img.shields.io/github/license/Open-CMSIS-Pack/debug-adapter-registry?label)](https://github.com/Open-CMSIS-Pack/debug-adapter-registry/blob/main/LICENSE)
[![Test and release ](https://img.shields.io/github/actions/workflow/status/Open-CMSIS-Pack/debug-adapter-registry/ci.yml?logo=arm&logoColor=0091bd&label=Test%20and%20release)](https://github.com/Open-CMSIS-Pack/debug-adapter-registry/tree/main/.github/workflows/ci.yml)

This repository contains the [debug adapter registry](https://open-cmsis-pack.github.io/cmsis-toolbox/build-operation/#debug-adapter-integration) and the CMSIS Solution extension specific adapter template files.

```txt
  📦
  ┣ 📂 registry               shared debug adapter registry
  ┃ ┗ 📄 debug-adapters.yml    yaml document listing all available debug adapters
  ┗ 📂 templates              command line tool source code
    ┣ 📄 *.adapter.json       json template for vscode launch and task definitions per debug adapter  
    ┗ 📄 ...
```

The adapter registry is used by CMSIS-Toolbox starting version 2.9.0 to generate `<solution>+<target-type>.cbuild-run.yml` files for the active target-set selected by

```
cbuild setup <solution> --active <target-type>[@<set>]
```

Available target-sets are listed by running

```
cbuild list target-sets <solution>
```

The `<solution>+<target-type>.cbuild-run.yml` file is then processed by the CMSIS Solution extension to generate launch.json and task.json files from the debug adapter template files.

---

## Validation

The [CI worklow](.github/workflows/ci.yml) validates the debugger adapter registry and templates by running linter and schema checker.
Such validation steps can be reproduced in the local environment according to the following instructions.
> Pre-requisite: it assumes [`npm`](https://nodejs.org/en/download) is installed in the system.

### Linter

tl:dr

```sh
npm install
npm run lint
```

Install [`eslint`](https://www.npmjs.com/package/eslint) and json/yaml plugins:

```sh
npm install --save-dev eslint eslint-plugin-jsonc eslint-plugin-yml eslint-formatter-compact
```

Lint debug adapters registry:

```sh
npx eslint --no-config-lookup --format compact --parser yaml-eslint-parser --plugin yml --ext .yml \
  --rule 'yml/quotes: ["error", { prefer: "double" }]' \
  --rule 'yml/indent: ["error", 2]' \
  --rule 'no-trailing-spaces: "error"' \
  registry
```

Lint templates:

```sh
npx eslint --no-config-lookup --format compact --parser jsonc-eslint-parser --plugin jsonc --ext .json \
  --rule 'jsonc/quotes: ["error", "double"]' \
  --rule 'jsonc/indent: ["error", 4]' \
  --rule 'no-trailing-spaces: "error"' \
  templates
```

### Schema check

tl:dr

```sh
npm install
npm run schema
```

Install [`ajv`](https://www.npmjs.com/package/ajv):

```sh
npm install --save-dev ajv ajv-cli
```

Check debug adapters registry schema:

```sh
npx ajv -s schemas/debug-adapters.schema.json -d registry/debug-adapters.yml --strict=false
```

Check templates schema:

```sh
npx ajv -s schemas/templates.schema.json -d "templates/*.json" --strict=false
```
