# debug-adapter-registry

This repository contains the [debug adapter registry](https://open-cmsis-pack.github.io/cmsis-toolbox/build-operation/#debug-adapter-integration)
and the CMSIS Solution extension specific adapter template files.

```txt
  📦
  ┣ 📂 registry               shared debug adapter registry
  ┃ ┗ 📄 debug-adapters.yml    yaml document listing all available debug adapters
  ┗ 📂 templates              command line tool source code
    ┣ 📄 *.adapter.json       json template for vscode launch and task definitions per debug adapter  
    ┗ 📄 ...
```

The adapter registry is used by CMSIS-Toolbox starting version 2.9.0 to generate `<solution>+<target-type>.cbuild-run.yml`
files for the active target-set selected by

```sh
cbuild setup <solution> --active <target-type>[@<set>]
```

Available target-sets are listed by running

```sh
cbuild list target-sets <solution>
```

The `<solution>+<target-type>.cbuild-run.yml` file is then processed by the CMSIS Solution extension to generate
`launch.json` and `task.json` files from the debug adapter template files.

---

## Debug Adapter Specification

The `registry/debug-adapters.yml` lists out all known debug adapters with additional details according to the following schema:

```yml
debug-adapters:
  - name: "<unique name>"
    alias-name: ["<alias name>", ... ]
    template: <adapter json template>
    gdbserver:
    defaults:
      <property>: <value>
    user-interface:
      - section: <section label>
        description: <section description>>
        yaml-node: <optional section yaml node>
        options:
          - name: <property label>
            description: <property description>
            yml-node: <yaml node name>
            type: <input field type>
            range: [<min value for number fields>, <max value for number fields>]
            scale: <scale factor for number fields>
            default: <default value>
            values: [<values for enum fields>]
            values: # alternative long format
              - name: <display label for enum value, defaults to value>
                value: <yaml text for enum value, defaults to name>
                description: <optional description for enum value>
```

```txt
  YAML Node             Value Type         Description                                                           
 --------------------- ------------------ -----------------------------------------------------------------------
  debug-adapters        object             Top level node for debug-adapters.yml                               
  ┣ name                string             Unique name to identify a debug adapter                               
  ┣ alias-name          array of strings   Optional alias names to identify the debug adapter                    
  ┣ template            string             Reference to the .adapter.json template file                        
  ┣ gdbserver           object             Optional, once specified additional gdb server handling is triggered  
  ┣ defaults            object             Optional default values for debug adapter specific settings           
  ┗ user-interface      array of objects   Definition for UI representation for relevant debug adapter settings  
    ┣ section           string             Label for a UI settings section                                       
    ┣ description       string             Optional section description                                          
    ┣ yaml-node         string             Optional yaml node name to group section options                  
    ┗ options           object             Options to group into the section                                     
      ┣ name            string             UI label for the option                                               
      ┣ description     string             UI description for the option (e.g., for tooltips)                    
      ┣ yaml-node       string             YAML node node name to store the option value to                      
      ┣ type            enum               UI input control type: string, number, file, enum                     
      ┣ range           [number, number]   Value range (for number type options)                               
      ┣ scale           number             Value display scale (for number type options)                       
      ┣ default         any                Default value used as UI preset                                       
      ┗ values          array of strings   Accepted values (for enum type options), short form.                
        ┃               array of objects   Accepted values (for enum type options), long form.                 
        ┣ name          string             Display label for the enum value, if omitted value is used.         
        ┣ value         string             Storage value for the enum value, if omitted name is used.          
        ┗ description   string             Optional description text.                                            
```

## Template Development

### Template input files

The `.adapter.json` files provide snippets to be injected into the workspace `launch.json` and `tasks.json` files:

```json
{
  "data": {
    "cwd": "${workspaceFolder}/<%= solution_folder %>",
  },
  "launch": {
    "singlecore-launch": { },
    "singlecore-attach": { },
    "multicore-start-launch": { },
    "multicore-start-attach": { },
    "multicore-other": { }
  },
  "tasks": [
    {
      "label": "CMSIS Erase"
    },
    {
      "label": "CMSIS Load"
    },
    {
      "label": "CMSIS Run",
    }, 
    {
      "label": "CMSIS Load+Run",
    },
    {
      "label": "CMSIS TargetInfo",
    }
  ]
}
```

```txt
  JSON Node                  Value Type         Description                                                           
 ---------------------      ------------------ -----------------------------------------------------------------------
  data                       object             User data processed on a global scope
  launch                     object             Template launch configurations
  ┣ singlecore-launch        object             Launch template for launching a single core debug session
  ┣ singlecore-attach        object             Launch template for attaching to a single core debug session
  ┣ multicore-start-launch   object             Launch template for launching a multi core debug session
  ┣ multicore-start-attach   object             Launch template for attaching to the primary core of a multi core debug session
  ┗ multicore-other          object             Launch template for attaching to other cores of a multi core debug session
  tasks                      array of objects   Template tasks
  ┗ label                    string             Unique name to identify the task, specific names are used for CMSIS debug related tasks
```

### Template processing

All string elements are processed one-by-one with the [ETA JS template engine](https://eta.js.org/). The `data` section
is processed in the first pass and the result is passed into subsequent template processing. Hence, once can collect
reused parts under `data` section to avoid duplication.

Arrays-of-strings loaded from the template are joined into a multi-line string, processed at onces and split based on linebreaks after processing.
This can be used to generate arrays using a multiline template if required:

```json
{
  "template": [
    "<% for elem in array %> {",
    "<%= elem.name %>\\n",
    "<% } %>"
  ]
}
```

Notice the explicit newline character inserted after the template tag. The implicit newline resulting from
concatenation is swallowed by the template processing if a template output data tag runs until the end of a line.

Template output data tags are typically used to create strings, similar to JavaScript format strings (backtick notation).
When outputting objects or arrays, these are stringified using JavaScript Object Notation (JSON). If JSON is detected
in a resulting launch or task configuration object, its converted back. E.g., one can achieve the following:

```json
{
  "data": {
    "shell": {
      "win32": {
        "executable": "cmd.exe",
        "args": [ "/d", "/c" ]
      },
      "linux": {
        "executable": "/bin/bash",
        "args": [ "-c" ]
      },
      "darwin": {
        "executable": "/bin/bash",
        "args": [ "-c" ]
      }
    },
  },
  "tasks": [
    {
      "options": {
        "shell": "<%= data.shell[process.platform] %>"
      },
    }
  ]
}
```

To be transformed into

```json
{
  "tasks": [
    {
      "options": {
        "shell": {
          "executable": "cmd.exe",
          "args": [ "/d", "/c" ]
        },
      },
    }
  ]
}
```

### Dynamic data structure

The following data structure is available to the templates:

```yml
data: { <custom data as specified> }
platform: <JavaScript process.platform>
solution_path: <path to .csolution.yml relative to workspace>
solution_folder: <base directory of .csolution.yml relative to workspace>
device_name: <name of target device>
target_type: <name of active csolution target-type>
start_pname: <processor name of primary processor if multi core device>
image_files:  # image files used to program the device (image, image+symbol)
  - file: <image file name relative to solution_folder>
    pname: <processor name>
symbol_files: # symbol files with debug information (symbol, image+symbol)
  - file: <symbol file name relative to solution_folder>
    pname: <processor name>
ports:        # GDB debug port mapping for all processors
  <pname>: <gdb-port>
config:       # entire debugger node from .cbuild-run.yml
processors:   # List of processors on a multi-core system
 - <pname> 
```

In template expressions all functions in JavaScript global scope can be used. In addition, the following helper functions are available:

| Function                                                 | Description      |
|----------------------------------------------------------|------------------|
| `Array<T>.groupedBy((e: T) => string): Map<string, T[]>` | Group elements of an array into a Map according to the given key-generator function. |
| `Map<K, V>.every((v: V, k: K) => boolean): boolean`      | Determines if all elements satisfy the given condition. |
| `Map<K, V>.mapValues<T>((v: V, k: K) => T): Map<K, T>`   | Transforms all elements with given function.  |
| `Map<K, V>.toObject(): Record<string, T>`                | Converts the Map into a JavaScript object  |

## Validation

The [CI worklow](.github/workflows/ci.yml) validates the debugger adapter registry and templates by running linter and
schema checker. Such validation steps can be reproduced in the local environment according to the following
instructions.

> **prerequisite:**
> [`npm`](https://nodejs.org/en/download) to be installed in the system.

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
