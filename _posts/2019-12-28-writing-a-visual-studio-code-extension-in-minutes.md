---
tags: NodeJS Typescript
---
![captionless image](https://miro.medium.com/v2/resize:fit:960/format:webp/1*MGcLJS1ZvMFcBA94PXn16Q.png)

**Writing a Visual Studio code extension in minutes**
=====================================================

Ever wanted to quickly learn how to develop an extension for vs code ? You have come to the right place!

> Autobots, transform and roll out! — Optimus Prime, Transformers

**STEP 0: Setup and Installation**
----------------------------------

*   Install [Visual Studio Code](https://code.visualstudio.com/)
*   Install [NodeJS](https://nodejs.org/en/)
*   Install [Git](https://git-scm.com/)
*   Install [NPM](https://www.npmjs.com/get-npm)

After we are done with these setups, we can now start building our project. Open a cmd and write ,

```
npm install -g yo generator-code
```

This installs the vs code official project scaffolding package

STEP 1: Create a project
------------------------

```
yo code
```

Write yo code and then fill out the questions

```sh
? What type of extension do you want to create? New Extension (TypeScript)
? What's the name of your extension? vscode-extension-example
? What's the identifier of your extension? vscode-extension-example
? What's the description of your extension? example
? Initialize a git repository? No
? Which package manager to use? npm
```

This will create the **vscode-extension-example** project complete with required files.

STEP 2: Extension basics
------------------------

So far our project directory somewhat looks like this,

```
├───.vscode
├───src
    │   extension.ts
    │
    └───test
        │   runTest.ts
        │
        └───suite
                extension.test.ts
                index.ts
├───CHANGELOG.md├───README.md
├───package.json
├───...
```

There are a few key points here. If you look into the `package.json`file, it contains a property called **contributes**.

> **Contribution Points** are a set of JSON declarations that you make in the `contributes` field of the `package.json`. Your extension registers **Contribution Points** to extend various functionalities within Visual Studio Code.

We are going to use this to register our [commands](https://code.visualstudio.com/api/references/contribution-points#contributes.commands). Commands are just an action which we want to perform such as create files, delete files, change theme…etc

The second thing, is to define an associated event w.r.t the command. Thing of it as the function which would be called when we execute our command.

> **Activation Events** is a set of JSON declarations that you make in the `activationEvents` field of `package.json`. Your extension becomes activated when the **Activation Event** happens

The third is your `extension_._ts` file which is the entry point of your code. This is where we define the event declared in package.json file and write our logic.

STEP 3: The Extension
---------------------

Let’s create a **file-maker** extension! The idea is this, we would add a **create new file** command when user right clicks on any folder inside vs code and upon entering a filename, a new file would be created. Sounds simple enough?

First let’s define our commands in package.json file.

Add the following command in contributes array,

```json
"contributes": {"commands": [ {  "command": "extension.CreateFile",  "title": "Create a new file" }]}
```

Now in order to make this command available on right click in explorer menu, we need to add it to the [**menus**](https://code.visualstudio.com/api/references/contribution-points#contributes.menus) property inside contributes array.

```json
"contributes": {
 "commands": [  {
    "command": "extension.CreateFile",
    "title": "Create a new file"
  }
 ]
 "menus": {
  "explorer/context": [  {
    "command": "extension.CreateFile",
    "group": "2_workspace"
  }
 ]
 }
}
```

We are not done yet with package.json. There’s a final thing remaining. We need to add an activation event to the [**activationEvents**](https://code.visualstudio.com/api/references/activation-events#onCommand) array

```json
"activationEvents": [ "onCommand:extension.CreateFile"
]
```

So far so good. Now we only need to write some code to get our extension working!

Open extension.ts file and add the following,

```js
import { window, ExtensionContext, commands, workspace, Uri } 
from 'vscode';
import { TextEncoder } from 'util';

export function activate(context: ExtensionContext) {

let disposableFileCommand =  commands.registerCommand('extension.CreateFile', (resource) => {
    window.showInputBox({ placeHolder: "Please enter a file name" })
    .then((fileName) => {
             if (fileName !== undefined) {
                  return workspace.fs.writeFile(Uri.parse(resource.path + '/' +   fileName + '.txt'), new TextEncoder().encode('Hello World'));
             }
        });
    });
context.subscriptions.push(disposableFileCommand);
}

export function deactivate() { }
```

Here we use the [**commands.registerCommand**](https://code.visualstudio.com/api/references/vscode-api#commands.registerCommand) function to register our command. The first argument is the command name we specified in package.json file and the second argument is lambda function in which we have written our code. The **resource** parameter is the file object we get when we right click on any folder complete with details like it’s path.

We also have use the **window.showInputBox** function to prompt the user for file name.

Finally we use vs code API’s own [**filesystem**](https://code.visualstudio.com/api/references/vscode-api#FileSystem) to write a new file.

If you press F5, a new vs code window would open up. Right click on any folder and voila we have our command !

![captionless image](https://miro.medium.com/v2/resize:fit:790/format:webp/1*MCHbvmhvx-Ro4Z9DRSDlDw.png)

Clicking on it would give you the prompt,

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*UvBtXduJBN3kP-pq2Kc9xg.png)

type a filename and press enter after which a new file would be created

![hello.txt](https://miro.medium.com/v2/resize:fit:748/format:webp/1*zTyVRPH1-nvLSk_gUv9aVw.jpeg)

This concludes our quick tutorial on developing an extension for visual studio code. Hope you had fun!

Repo: [https://github.com/SpartanX1/vscode-extension-example](https://github.com/SpartanX1/vscode-extension-example)

> There’s more to them than meets the eye — Optimus Prime, Transformers
