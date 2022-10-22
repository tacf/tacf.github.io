---
title: "Building a Pipeline Engine with Go Plugins"
date: 2022-10-22T16:35:00+01:00
draft:
ShowToc: true
ShowBreadCrumbs: true
---
# Building a Pipeline Engine with Go Plugins

Status: In progress
Tags: golang, software development

If you’ve worked with any ci/cd platform you’re no stranger to yaml defined pipelines. For example

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

represents a simple Github Actions workflow. Looking closely at `uses: actions/checkout@v3` we immediately identify this as a method/task/functions that his available in some way in order to simplify our work. Usually platforms like Github Actions, Travis CI and so on have a set of this methods readily available and more often than not community ones. One can easily assume that most of this usually relies on some plugin/addon system. So now the question that presents itself…

> Can we build a system like this using Golang Plugin library?
> 

Short anwser is yes !!

This article will guide you through my though process on building this type of tool using Go Plugins. 

## Our Building Blocks

### Yaml Definition

The yaml definition will be, as per usual, our instruction set on how our pipeline should behave. Let’s use something quite simple.

```yaml
steps:
  - task: plugin2
    parameters:
      command: "echo Hello World"
  - task: plugin2
    parameters:
      command: "echo Hello Odd World"
```

Keeping it as simple as possible, our yaml definition consist in a list of tasks. These tasks are defined by,

1. Task name - represented by the `task` property
2. Parameters list - identified by the `parameters` object

### Plugin

Plugins will consist in Go libraries that respect certain requirements like,

- Having a `NewInstance` function; and
- variable named `Name`

These two symbols will helps us, 1) Finding the library corresponding to the defined task in the *YAML Definition,* and 2) Generating a handler object already populated with values and ready to execute the task. We’ll call this object ************Executable Task************ or simply *****Task.*****

### Task

A task in the context of our engine is an instance of the *Plugin* that is initialized (including the arguments to be used) and ready to be executed by the engine.

### Engine

The engine component itself will be our orchestrator. It’s main functions will be,

- Load & Validate the Yaml definition
- Load Plugins
- Execute the Pipeline - Invoke plugins based on the instructions of the Yaml definition

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

As we mentioned, the goal of our Plugin  is to generate ****************Executable Tasks.**************** For this we’ll need the Plugin code to implement some properties and allow us to request the creation of the ***Task*** object. 

Lets take a look 👀

```go
var Name string = "PluginName"

type task struct {
	Parameters map[string]string
}

func (task) Exec() { ... }

func (task) GetName() string { ... }

func NewInstance(parameters map[string]string) interface{} { ... }
```

We start by defining a variable `Name` this will be useful to search and index loaded plugins in the engine for later consumption to generate ******Tasks.******

The `type task struct {...}` represents the *****Task***** objects that will be instanciated.

`Exec()` and `GetName()` are the actual method implementation of our interface.

And last, but not least, the `NewInstance(...)` function. This function will allow us to request a new instance of a task (the actual executable plugin code) from the Plugin.

Note that both `NewInstance()` and `Name` symbols are not directly reachable by the `task` structure, this is mainly because these symbols serve only purposes of loading and creating new executable instances of the plugin (our *******Tasks*******)*******.*******

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

Note that this is not the actual full code for the load function, this is a cleanup, no error checking version, to simplify readability. For the full code check the references section.

What we do here is basically iterating over the plugin paths identified previously and,

1. Lookup the existance of both `Name` and `NewInstance` symbols in the Plugin (casting `NewInstance` symbol to the actual function executable type. At this point we still can’t cast the return type since we don’t have any concrete object to match the type against (trying otherwise would give you runtime errors)
2. We store the plugins in a map for later use. Here we also need to do some casting to convert symbol `Name` to it’s actual `string` type.

### Task Instantiation

Task instantiation consisting in picking up the yaml definition and for every task instruction we find, we search for the requested plugin in our loaded plugins and create an instance of an ***********Executable Task*********** for that task instruction.

```go
func instanciateTasks(pluginLibs PluginLibs, tasks pipelineYaml.TasksYaml) ExecutableTasks {
	executableTasks := make(ExecutableTasks, len(tasks))
	i := 0
	for _, v := range tasks {
		pluginLib, _ := pluginLibs[strings.ToLower(v.Name)]

		executableTasks[i] = pluginLib(v.Parameters).(Plugin)
		i++
	}
	return executableTasks
}
```

The code is pretty straight forward. The only point that may need addressing is `pluginLib(v.Parameters).(Plugin)` . Remember during plugin loading that we couldn’t immediately cast the type of the return of `NewInstance` ? Well now we can ! This is exactly what this instruction is doing. `pluginLib` is our function pointer to `NewInstace`, we call it with `v.Parameters` in order to get an ****************Executable Task**************** already populated with the arguments needed and then we cast the `interface{}` into it’s actual type `Plugin`

### Task Execution

After having been through all the steps. Loading the plugins and instantiating our ****************Executable Tasks**************** with the proper arguments, one could assume that executing tasks should be pretty straight forward. And those assumptions would be spot on 🙂

```go
func executeTasks(executableTasks ExecutableTasks) {
	for _, executeTask := range executableTasks {
		executeTask.Exec()
	}
}
```

Executing our tasks is as simple as iterating over our list of instantiated ones and calling its `Exec()` method.

## What’s Next?

- Plugin Versioning
- State Sharing
- Output handling and Storing

## References

Source Code: [https://github.com/tacf/tiny-pipeline-engine](https://github.com/tacf/tiny-pipeline-engine)
