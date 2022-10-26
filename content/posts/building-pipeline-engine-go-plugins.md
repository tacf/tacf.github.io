---
title: "Building a Pipeline Engine with Go Plugins"
date: 2022-10-22T16:35:00+01:00
tags: ["golang","cicd","pipelines"]
ShowToc: true
ShowBreadCrumbs: true
---

CI/CD tools are at the core of every companys software delivery pipeline. This tools are even the very product that some companies develop and sell. But... are we able to build our own custom CI/CD tools? Do we possess the required skills to build our own pipeline engine?

## Building a Pipeline Engine

If you’ve worked with any CI/CD platform, you’re no stranger to yaml defined pipelines. Every platform, nowadays, implements their own Yaml structure, allowing operational and development teams to design their pipelines to their hearts content.

At the core of these CI/CD platforms lies an engine, capable of picking up the **Yaml Definitions** that engineers throw their way and execute the defined instructions. 

These platforms have evolve in such a way, that they allow high levels of customisation, not limited but including the use of custom tasks. These tasks can be viewed as an equivalent to a function or method, in the sense that, they can be called by it’s name and provided with a list of parameters/arguments

Lets look at an example from GitHub actions.

```yaml
name: Workflow Name
run-name: Workflow Execution Name
on: [push]
jobs:
  Linting:
    runs-on: ubuntu-latest
      steps:
      - name: Check out repository code
        uses: actions/checkout@v3
      - run: echo "Hello World"
```

The above pipeline definition defines two tasks (seen in the `steps` section). These two tasks are a good example of a **custom** and a **native** one (we’ll talk about what **native** means in a second).

The first task is an example of a **custom** task, it specifies that the task is named `actions/checkout@v3`. This **custom** task in particular does not require any arguments.

On the other hand, the task defined by the keyword `run` is a **native** task. These tasks are usual characterised by being invoked using keywords that are part of the **Yaml Definition** specification. As you can see no keyword that allows us to select a method or function to call but a `run` keyword that is part of the **Yaml specification**.

Custom tasks like `actions/checkout@v3` are provided by either the platform developer or by the community. By design, these are typically built around respecting some contract/interface. This allows the engine of these execution platforms, among other things, to load tasks at run time. Thus giving a huge flexibility and extensibility in terms of development for these platforms.

Looking at Golang, which nowadays is being highly preferred, by projects and teams working in or with Operational oriented platforms and tools, we can see that since version 1.8, it allows to dynamically load code using their library `Plugin`.  So now the question stands,

> Can we build a system/platform like this using Golang and its Plugin library?

Short answer is **yes** !!

This article will guide you through the process on building a pipeline engine that is mainly built around [Go Plugin library](https://pkg.go.dev/plugin). 

## Our Building Blocks

Well start our guided journey by looking at the base concepts and building blocks that we’ll be using to create our **tiny pipeline engine** 🙂

### Yaml Definition

As mentioned, the **Yaml Definition** is the basis of the interaction between engineers and the execution engine. Lets start by defining our yaml specification for our pipelines.

Similarly to the example definition of Github Actions previously presented, we’ll have a list of `steps`. Each step will consist in a `task` keyword which represents the function to be called (similar to the `uses` keyword from the previous example) and a `parameters` keyword which will be a map of parameters and their respective values that will be used by the function.

```yaml
steps:
  - task: plugin2
    parameters:
      command: "echo Hello World"
  - task: plugin2
    parameters:
      command: "echo Hello Odd World"
```

Yes, this is not a specification but actually an example. For the purpose of this guide it should suffice 😉

### Plugin

A plugin in its essence works like any other library file (`.dll` in Windows, `.so` in Linux etc). In order to use it, we need capabilities to load those compiled library files, read its symbols and eventually call its functions/procedures. That’s where the `Plugin` library from Golang comes into play. 

Note: Technically there’s no hard requirement that plugins need to be written/built using Golang. As long as they respect the contract defined and can be loaded, they can be built in pretty much any language (even C 😀 ). 

In or case, plugins will consist in Go libraries that respect certain requirements like,

- Having a `NewInstance` function; and
- variable named `Name`

More on this in a minute.

### Task

A task in the context of our engine, is an instance of the *Plugin* that is initialized (including the arguments to be used) and ready to be executed by the engine.

### Engine

The engine component will be our orchestrator. Its main functions will be,

- Load the Yaml definition
- Load Plugins
- Execute the Pipeline - Invoke plugins/tasks based on the instructions of the Yaml definition

## Plugins

### Defining a Plugin Interface

As we mentioned, plugins need to agree on a standard interface. With the goal of keeping everything as simple as possible, we’ll use the following interface.

```go
type Plugin interface {
	Exec()
	GetName()
}
```

With this type interface we’re defining that our plugins will have at least a function `Exec()`  and a function `GetName()`. This is an oversimplification of course. In real world usage there’s a lot more functionality required from plugins.

### Plugin Implementation

The goal of our Plugin library will be to generate **Executable Tasks.** For this we’ll need the Plugin code to implement some properties and allow us to request the creation of the **Task** object. 

Lets take a look 👀

```go
var Name string = "<Plugin/Task Name>"

type task struct {
	Parameters map[string]string
}

func (task) Exec() { ... }

func (task) GetName() string { ... }

func NewInstance(parameters map[string]string) interface{} { ... }
```

We start by defining a variable `Name`. This variable will be useful to search and index loaded plugins in the engine, in order to be able to instantiate **Tasks** based on that plugin.

The `type task struct {...}` represents the **Task** objects that will be instantiated and will keep any state we desire related to the execution of the task.

The `NewInstance(...)` function, will allow us to request a new instance of a task (the actual executable plugin code) from the Plugin.

`Exec()` and `GetName()` are the actual method implementation of our interface.

Note that both `NewInstance()` and `Name` symbols are not directly reachable by the `task` structure, this is because these symbols have the purpose of loading plugins and instantiating tasks and will be mainly used by our engine to handle these operations.

## Engine

### Plugin Loading

Let’s jump straight into some sample code.

```go
func loadPluginLibs(pluginPaths []string) PluginLibs {
	plugLibs := PluginLibs{}
	for _, path := range pluginPaths {
		p, _ := plugin.Open(path)
	
		pluginName, _ := p.Lookup("Name")
		pluginNewInstance, _ := p.Lookup("NewInstance")
		pluginNewInstanceHandler := pluginNewInstance.(func(map[string]string) interface{})
	
		pName := *pluginName.(*string)
		plugLibs[pName] = pluginNewInstanceHandler
	}
	return plugLibs
}
```

Note that this is not the full code for the load function, this is a partial snippet, to simplify readability. For the full code check the references section.

What we’re doing here is basically iterating over the plugin paths identified previously and,

1. Lookup the existence of both `Name` and `NewInstance` symbols in the Plugin (casting `NewInstance` symbol to the actual function executable type. At this point we still can’t cast the return type since we don’t have any concrete object to match the type against (trying otherwise would give you runtime errors)
2. We store the plugins in a map for later use. Here we also need to do some casting to convert symbol `Name` to its actual `string` type.

### Task Instantiation

Task instantiation consists in picking up the yaml definition and for every task instruction we find (task name and parameters), we’ll search for the requested plugin in our loaded plugins list and create an instance of an **Executable Task** for that task instruction.

```go
func instanciateTasks(pluginLibs PluginLibs, tasks pipelineYaml.TasksYaml) ExecutableTasks {
	executableTasks := make(ExecutableTasks, len(tasks))
	for i, v := range tasks {
		pluginLib, _ := pluginLibs[strings.ToLower(v.Name)]

		executableTasks[i] = pluginLib(v.Parameters).(Plugin)
	}
	return executableTasks
}
```

The only point that may require additional clarification from this snippet is `pluginLib(v.Parameters).(Plugin)` . 

Previously we mentioned that we couldn’t immediately cast the type of the return of `NewInstance` function, but now, is exactly the time for us to do that.

 `pluginLib` is our function pointer to `NewInstance`. Since we’re now at the point where we need to get the task object populated and initialized we can execute the call to `NewIntance()` by passing `v.Parameters` (the parameters map from our yaml definition) as its argument, and then we can finally cast the return object from `interface{}` into it’s actual type, `Plugin`.

### Task Execution

After having been through all the steps. Loading the plugins and instantiating our **Executable Tasks** with the proper arguments, one could assume that executing tasks should be pretty straight forward. And those assumptions would be spot on 🙂

```go
func executeTasks(executableTasks ExecutableTasks) {
	for _, executeTask := range executableTasks {
		executeTask.Exec()
	}
}
```

Executing our tasks is as simple as iterating over our list of instantiated ones and calling its `Exec()` method.

## What’s Next?

We mentioned a lot how this is a simplistic approach to the implementation of such engine. Other features that we could invest into could be,

- Plugin Versioning - the example we started with from Github actions show an example of the possibility of invoking a specific version for a function
- State Sharing - having the ability to transfer details between steps
- Output handling and Storing - treating the output in order to provide a better view of what is happening and what triggered it.

## Conclusions

We’ve seen how “easy” it is to implement these sort of plugable systems using the plugin library from Go (well, it’s in the name of the library right? 😀 ). We’ve showed how we can implement an engine that allows us to execute customs tasks. Such an engine should and can be highly extendable. This type of work may expand into CI/CD platforms, like GitHub Actions, as well as workflows managers, like [Argo Workflows](https://argoproj.github.io/argo-workflows/), or any other type of work that we deem that could benefit from custom extensibility.

## References

Source Code: [https://github.com/tacf/tiny-pipeline-engine](https://github.com/tacf/tiny-pipeline-engine/tree/blog-post) (Blog Post Branch)
