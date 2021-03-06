# Step by Step: Node Task with Typescript API

This step by step will show how to manually create, debug and test a cross platform task with no service, server or agent!

Tasks can be created using tfx as a convenience, but it's good to understand the parts of a task.  
This tutorial walks through each manual step of creating a task.

Note: The tutorial was done on a mac.  Attempted to make generic for all platforms.  Create issues in the repo.

## Video

https://youtu.be/O_34c7p4GlM?t=1173

## Tools 

[Typescript Compiler 2.2.0 or greater](https://www.npmjs.com/package/typescript)  

[Node 4.4.7 (LTS) or greater](https://nodejs.org/en/)  

[Npm 3.0 or greater recommended](https://www.npmjs.com/package/npm3) (comes with node >=5. creates flat dependencies in tasks)   

This tutorial uses [VS Code](https://code.visualstudio.com) for great intellisense and debugging support

## Sample Files

Files used for this walk through are [located here in this gist](https://gist.github.com/bryanmacfarlane/154f14dd8cb11a71ef04b0c836e5be6e)

## Create Task Scaffolding

Create a directory and package.json.  Accept defaults of npm init for sample.
```bash
$ mkdir sampletask && cd sampletask
$ npm init
```

### Add vsts-task-lib

Add vsts-task-lib to your task.  Remember your task must carry the lib.  Ensure it's at least 0.9.5 which now carries typings.  

The package.json should have dependency with ^.  Ex: ^0.9.5.  This means you are locked to 0.9.x and will pick up patches on npm install.

The npm module carries the .d.ts typecript definition files so compile and intellisense support will just work.

```
$ npm install vsts-task-lib --save
...
└─┬ vsts-task-lib@2.0.5 
... 
```

### Add Typings for Externals

Ensure typings are installed for external dependencies

```bash
$ npm install @types/node --save-dev
$ npm install @types/q --save-dev
```

Create a .gitignore and add node_modules to it.  Your build process should do `npm install` and `typings install`.  No need to checkin dependencies.

```bash
$ cat .gitignore
node_modules
```

### Create tsconfig.json Compiler Options

```bash
$ tsc --init
```

Change `tsconfig.json` file to ES6 to match the sample gist.  ES6 is for async await support.

## Task Implementation

Now that the scaffolding is out of the way, let's create the task!

Create a `task.json` file using `sample_task.json` as a starting point. The full task JSON schema is [here](https://aka.ms/vsts-tasks.schema.json).

Replace the `{{placeholders}}`.  The most important being a [unique guid](http://www.guidgen.com/).
Note: copy from web view since file needs property names in quotes (browser might strip in raw view)

Create a `index.ts` file using index.ts from the gist as a starting point.
Create a `taskmod.ts` file using taskmod.ts from the gist as a starting point.

Instellisense should just work in [VS Code](https://code.visualstudio.com)

The code is straight forward.  As a reference:

```javascript
import tl = require('vsts-task-lib/task');
import trm = require('vsts-task-lib/toolrunner');
import mod = require('./taskmod');

async function run() {
    try {
        console.log(process.env["INPUT_SAMPLESTRING"]);
        let tool: trm.ToolRunner;
        if (process.platform == 'win32') {
            let cmdPath = tl.which('cmd');
            tool = tl.tool(cmdPath).arg('/c').arg('echo ' + tl.getInput('samplestring', true));
        }
        else {
            let echoPath = tl.which('echo');
            tool = tl.tool(echoPath).arg(tl.getInput('samplestring', true));
        }

        let rc1: number = await tool.exec();
        
        // call some module which does external work
        if (rc1 == 0) {
            mod.sayHello();
        }
        
        console.log('Task done! ' + rc1);
    }
    catch (err) {
        tl.setResult(tl.TaskResult.Failed, err.message);
    }
}

run();
```

taskmod.ts:

```javascript
export function sayHello() {
    console.log('Hello World!');
}
```

Key Points:

 - Async code is linear.  Note the two executions written one after each other in linear fashion.
 - Must be wrapped in async function.
 - Greatly simplifies error handling.
 - Never process.exit your task.  You can sometimes lose output and often the last bit of output is critical

If we did our job well, the code should be pretty self explanatory.    
But, see the [API Reference](vsts-task-lib.md) for specifics.

## Compile

Just type `tsc` from the root of the task.  That should have compiled an index.js

## Run the Task

The task can be run by simply running `node index.js`.  Note that is exactly what our agent will do.

```bash
$ node index.js
##vso[task.debug]agent.workFolder=undefined
##vso[task.debug]loading inputs and endpoints
##vso[task.debug]loaded 0
##vso[task.debug]task result: Failed
##vso[task.issue type=error;]Input required: samplestring
##vso[task.complete result=Failed;]Input required: samplestring
``` 

The task failed!  That's exactly what would happen if the task run and inputs were not supplied.

The agent runs the task and reads key information from command output output over stdout and stderr.  This allows for consistency in node, powershell or even ad-hoc scripts.  The lib is mostly a thin wrapper generating those commands.

Let's supply one of the inputs and try again.

```bash
$ export INPUT_SAMPLESTRING="Hello World"
$ node index.js
##vso[task.debug]agent.workFolder=undefined
##vso[task.debug]loading inputs and endpoints
##vso[task.debug]loading INPUT_SAMPLESTRING
##vso[task.debug]loaded 1
##vso[task.debug]samplestring=Hello World
##vso[task.debug]echo arg: Hello World
##vso[task.debug]exec tool: echo
##vso[task.debug]Arguments:
##vso[task.debug]   Hello World
[command]echo Hello World
Hello World
##vso[task.debug]rc:0
##vso[task.debug]success:true
##vso[task.debug]samplebool=null
Task done! 0,-1
$ 
```

> TIP: be careful with chars like ! in env vars.  [Example here](http://superuser.com/questions/133780/in-bash-how-do-i-escape-an-exclamation-mark) 

> TIP: if you are using Visual Studio Code on Windows, you can also update the inputs using the following PowerShell command instead of EXPORT:
```
$env:INPUT_SAMPLESTRING="Hello World"
```

Now let's set the sample bool.  This should fail since if sample bool is true, it should need the other input.  See the code.

```bash
$ export INPUT_SAMPLEBOOL=true
$ node index.js
##vso[task.debug]agent.workFolder=undefined
##vso[task.debug]loading inputs and endpoints
##vso[task.debug]loading INPUT_SAMPLEBOOL
##vso[task.debug]loading INPUT_SAMPLESTRING
##vso[task.debug]loaded 2
##vso[task.debug]samplestring=Hello World
##vso[task.debug]echo arg: Hello World
##vso[task.debug]exec tool: echo
##vso[task.debug]Arguments:
##vso[task.debug]   Hello World
[command]echo Hello World
Hello World
##vso[task.debug]rc:0
##vso[task.debug]success:true
##vso[task.debug]samplebool=true
##vso[task.debug]task result: Failed
##vso[task.issue type=error;]Input required: samplepathinput
##vso[task.complete result=Failed;]Input required: samplepathinput
$ 
```

So, as you can see, this offers powerful testing automation options to test all arcs with no agent or server.

## Interactive Use from Command Line (Advanced)

Node offers an interactive console and since the task lib is in your node_modules folder, you can interactively poke around.

```bash
$ node
> var tl = require('vsts-task-lib/task');
##vso[task.debug]agent.workFolder=undefined
##vso[task.debug]loading inputs and endpoints
##vso[task.debug]loaded 0
undefined
> tl.which('echo');
##vso[task.debug]echo=/bin/echo
'/bin/echo'
> tl.tool('echo').arg('Hello World!').args
##vso[task.debug]echo arg: Hello World!
[ 'Hello World!' ]
> .exit

```
## Debugging

Coming soon

## Unit testing your task scripts

This requires vsts-task-lib 0.9.15 or greater.


### Goals:  

- Unit test the script, not the external tools it's calling.
- Run subsecond (often < 200 ms)
- Validate all arcs of the script, including failure paths.
- Run your task script unaltered the exact way the agent does (envvar as contract, run node against your script)
- Assert all aspects of the execution (succeeded, issues, std/errout etc...) by intercepting command output

### Install test tools

We will use mocha as the test driver in this examples.  Others exist.
```bash
npm install mocha --save-dev -g
npm install @types/mocha --save-dev
```

### Create test suite

Creates tests folder and _suite.ts.  [Example here](https://gist.github.com/bryanmacfarlane/154f14dd8cb11a71ef04b0c836e5be6e#file-_suite-ts)

### Success test

The success test will validate that the task will succeed if the tool returns 0.  It also confirms that no error or warning issues are added to the build summary.  Finally it validates that the task module is called if the tool succeeds.

The success test [from _suite.ts](https://gist.github.com/bryanmacfarlane/154f14dd8cb11a71ef04b0c836e5be6e#file-_suite-ts) looks like:

```javascript
    it('should succeed with simple inputs', (done: MochaDone) => {
        this.timeout(1000);

        let tp = path.join(__dirname, 'success.js');
        let tr: ttm.MockTestRunner = new ttm.MockTestRunner(tp);

        tr.run();
        assert(tr.succeeded, 'should have succeeded');
        assert.equal(tr.invokedToolCount, 1);
        assert.equal(tr.warningIssues.length, 0, "should have no warnings");
        assert.equal(tr.errorIssues.length, 0, "should have no errors");
        assert(tr.stdout.indexOf('atool output here') >= 0, "tool stdout");
        assert(tr.stdout.indexOf('Hello Mock!') >= 0, "task module is called");

        done();
    });
```

key code from [success.ts test file](https://gist.github.com/bryanmacfarlane/154f14dd8cb11a71ef04b0c836e5be6e#file-success-ts)

```javascript

let taskPath = path.join(__dirname, '..', 'index.js');
let tmr: tmrm.TaskMockRunner = new tmrm.TaskMockRunner(taskPath);

tmr.setInput('samplestring', "Hello, from task!");
tmr.setInput('samplebool', 'true');

// provide answers for task mock
let a: ma.TaskLibAnswers = <ma.TaskLibAnswers>{
    "which": {
        "echo": "/mocked/tools/echo",
        "cmd": "/mocked/tools/cmd"
    },
    "exec": {
        "/mocked/tools/echo Hello, from task!": {
            "code": 0,
            "stdout": "atool output here",
            "stderr": "atool with this stderr output"            
        },
        "/mocked/tools/cmd /c echo Hello, from task!": {
            "code": 0,
            "stdout": "atool output here",
            "stderr": "atool with this stderr output"            
        }
    }
};
tmr.setAnswers(a);

// mock a specific module function called in task 
tmr.registerMock('./taskmod', {
    sayHello: function() {
        console.log('Hello Mock!');
    }
});

tmr.run();

```

### Fail test

This test validates that the task will fail if the tool returns 1.  It also validates that an error issue will be added to the build summary.  Finally, it validated that the task module is not called if the tool fails.

The fail test [from _suite.ts](https://gist.github.com/bryanmacfarlane/154f14dd8cb11a71ef04b0c836e5be6e#file-_suite-ts) looks like:

```javascript
    it('it should fail if tool returns 1', (done: MochaDone) => {
        this.timeout(1000);

        let tp = path.join(__dirname, 'failrc.js');
        let tr: ttm.MockTestRunner = new ttm.MockTestRunner(tp);

        tr.run();
        assert(!tr.succeeded, 'should have failed');
        assert.equal(tr.invokedToolCount, 1);
        assert.equal(tr.warningIssues, 0, "should have no warnings");
        assert.equal(tr.errorIssues.length, 1, "should have 1 error issue");
        if (process.platform == 'win32') {
            assert.equal(tr.errorIssues[0], '/mocked/tools/cmd failed with return code: 1', 'error issue output');
        }
        else {
            assert.equal(tr.errorIssues[0], '/mocked/tools/echo failed with return code: 1', 'error issue output');
        }
        assert(tr.stdout.indexOf('atool output here') >= 0, "tool stdout");
        assert.equal(tr.stdout.indexOf('Hello Mock!'), -1, "task module should have never been called");

        done();
    });
```

The [failrc.ts test file is here](https://gist.github.com/bryanmacfarlane/154f14dd8cb11a71ef04b0c836e5be6e#file-failrc-ts)

### Fail on required inputs

We also typically write tests to confirm the task fails with proper actionable guidance if required inputs are not supplied.  Give it a try.

### Running the tests

```bash
mocha tests/_suite.js
```

```bash
$ mocha tests/_suite.js


  Sample task tests
    ✓ should succeed with simple inputs (127ms)
    ✓ it should fail if tool returns 1 (121ms)


  2 passing (257ms)
```

If you want to run tests with verbose task output (what you would see in the build console) set envvar TASK_TEST_TRACE=1

```bash
export TASK_TEST_TRACE=1
```

## Add Task to an Extension

### Create your publisher

All extensions are identified as being provided by a publisher. If you aren't already a member of an existing publisher, you'll create one. Sign in to the [Visual Studio Marketplace Publishing Portal](http://aka.ms/vsmarketplace-manage). If you're not prompted to create a publisher, scroll down to the bottom of the page and select *Publish Extensions* underneath **Related Sites**.

### Create an extension manifest file

You will need to create the extension manifest file in the directory above `sampletask`.

Create a file `vss-extension.json`:
```json
{
    "manifestVersion": 1,
    "id": "sample-task",
    "name": "Sample Build Tools",
    "version": "0.0.1",
    "publisher": "samplepublisher",
    "targets": [
        {
            "id": "Microsoft.VisualStudio.Services"
        }
    ],    
    "description": "Sample tools for building. Includes one build task.",
    "categories": [
        "Build and release"
    ],
    "//uncomment 'icons' below to include a custom icon": "",
    "//icons": {
        "default": "images/extension-icon.png"
    },
    "files": [
        {
            "//Relative path of the task directory": "",
            "path": "sampletask"
        }
    ],
    "contributions": [
        {
            "//Identifier of the contribution. Must be unique within the extension. Does not need to match the name of the build task, but typically the build task name is included in the ID of the contribution.": "",
            "id": "sample-build-task",
            "type": "ms.vss-distributed-task.task",
            "targets": [
                "ms.vss-distributed-task.tasks"
            ],
            "properties": {
                "//Name of the task. This must match the folder name of the corresponding self-contained build task definition.": "",
                "name": "sampletask"
            }
        }
    ]
}
```

### Publish, Install, Publish

Publish the extension to the Marketplace and grant your account the ability to see it. Sharing the extension with your account allows you to install and test it.

You will need a personal access token scoped to `All accessible` accounts.

```
tfx extension publish --manifest-globs your-manifest.json --share-with youraccountname
```

Install the extension into your account.

```
tfx extension install --vsix .\samplepublisher.sample-task-0.0.1.vsix --accounts youraccountname
```

Future publishes will automatically update the extension installed into your account. Add `--rev-version` when publishing updates to the extension, and also rev the task version (in the `task.json`).
