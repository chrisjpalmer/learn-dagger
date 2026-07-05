# Learn Dagger

This tutorial will run demonstrate how to use [dagger](dagger.io) to build a simple go app.

## What is Dagger?

Dagger is a workflow engine that allows developers to write in their favourite programming
lanuguage to interact with containers, files, directories and secrets.
Dagger uses technology similar to that of buildkit, to cache operations and speed up execution.
Workflows are executed by the engine, leading to a consistent experience locally and in CI.

## Why use it?

Typically, CI pipelines are created in YAML and cannot be tested locally. This leads to lots of "push and pray"
to verify the pipeline works, which is slow and prone to error. With the need to "shift left", more burden is 
placed on CI, and eventually pipelines become unmaintainable. Dagger allows the
use of normal programming languages to define pipelines, and guarantees a consistent execution
environment through containerisation. Code is packaged into Dagger modules, for effecient re-use
of pipeline code.

## What we'll cover

- How to create a dagger module
- How to add a function to the module
- How to convert our function into a check

## Prerequisites

- Docker / Docker Desktop

## Install Dagger

**Mac**

```sh
brew install dagger/tap/dagger
```

**Linux**

```sh
curl -fsSL https://dl.dagger.io/dagger/install.sh | BIN_DIR=$HOME/.local/bin sh
```

**Windows**

```sh
winget install Dagger.Cli
```

**asdf**

```sh
asdf plugin add dagger
asdf install
```

## Test installation

```sh
dagger core version
```

## Create a new module

Pipeline code is packaged into *dagger modules* and modules 
are installed into a project via the `dagger.json` file.

1. Create a module called `main`:

```sh
dagger init --sdk=go --name=main .
```

This has created the `main` module under `.dagger`.

2. Before editing a module, it is important to get the *dagger generated code*,
which will give IDE support while working on the module.

```sh
dagger develop
```

This command will drop new files into the module directory:
- `internal/dagger` package
- `dagger.gen.go`

With these files in place, our module is now a *complete go project*. Though you
can't run it directly, all go tools will work on it.

## Create a dagger function

Dagger modules are composed of *dagger functions*. Dagger functions are methods
on the "default struct type" of our module (which is the same as the module name).

1. Open `.dagger/main.go`

2. Delete the functions named `ContainerEcho` and `GrepDir`.

3. Make the following changes to add a new `Build` function to the module:

```diff
+ // Main - contains functions for building a docker image
  type Main struct{}

+ // Build - builds the source and returns the release container
+ func (m *Main) Build() *dagger.Container {
+ 	return dag.Container()
+ }
```

4. Call the function:

```sh
dagger call build
```

### What just happened?

1. The code was compiled into a binary.
2. The engine spawned the binary and told it to run its `Build` function.
3. The container API was used to create an empty container: `dag.Container()`.

## Passing parameters

Dagger functions can accept parameters. Lets edit our build function to accept `name`:

```go
// Build - builds the source and returns the release container
func (m *Main) Build(name string) *dagger.Container {
	fmt.Println("Hi", name)

	return dag.Container()
}
```

Call the build function with a name:

```sh
dagger call build --name=<your name> -E

▶ ✔ .build(name: "chris"): Container! 1.8s
```

To see the hello statement, select the `.build` stage in the TUI and hit enter:

```sh
▼ ✔ Main.build(name: "chris"): Container! 0.1s
  ● $ container: Container! 0.0s CACHED

  Hi chris  
```

If you see `CACHED` next to `.build`, it means the function execution has been cached.
Try running the command with a different name.

```sh
$ .build(name: "chris"): Container! 0.0s CACHED
```

## Lets build the sample app

This part of the tutorial will take you through how to build the sample app
in this repo.

### Get the source

By default the module can't see the source of the repo. 
We can change that by using the `*dagger.Workspace` parameter.
A `*dagger.Workspace` is injected automatically by the engine.

1. Remove the `name` parameter and replace it with `ws *dagger.Workspace`.

```diff
- func (m *Main) Build(name string) *dagger.Container {
+ func (m *Main) Build(ws *dagger.Workspace) *dagger.Container {
```

2. Store the source directory in a variable for later use:

```diff
func (m *Main) Build(ws *dagger.Workspace) *dagger.Container {
+	src := ws.Directory("/", dagger.WorkspaceDirectoryOpts{Exclude: []string{".dagger"}})
```

### Build the source

Commands can be run in containers to build the source.

1. Create a build container based on `golang:latest`:

```diff
func (m *Main) Build(ws *dagger.Workspace) *dagger.Container {
	...
+	dag.Container().
+		From("golang:latest")
}
```

2. Set the working directory:

```diff
func (m *Main) Build(ws *dagger.Workspace) *dagger.Container {
	...
	dag.Container().
		From("golang:latest").
+		WithWorkdir("/src")
```

3. Mount the source in the container:

```diff
func (m *Main) Build(ws *dagger.Workspace) *dagger.Container {
	...
	dag.Container().
		From("golang:latest").
		WithWorkdir("/src").
+		WithDirectory(".", src)
```

4. Run `go build` to build the source:

```diff
func (m *Main) Build(ws *dagger.Workspace) *dagger.Container {
	...
	dag.Container().
		From("golang:latest").
		WithWorkdir("/src").
		WithDirectory(".", src).
+		WithExec([]string{"go", "build", "-o", "app", "."})
```

5. Store the binary in a variable for later use.

```diff
func (m *Main) Build(ws *dagger.Workspace) *dagger.Container {
	...
-	dag.Container()
+	app := dag.Container().
		From("golang:latest").
		WithWorkdir("/src").
		WithDirectory(".", src).
		WithExec([]string{"go", "build", "-o", "app", "."}).
+		File("app")
}
```

> [NOTE]
> `app` doesn't store the binary, but is actually a graphql query that describes
> the graph of operations to get the binary. It will be executed later by the engine.

### Create the release container

Files from previous stages can be mounted in other containers. This can be used
to create a release container.

1. Create a release container with the binary:

```diff
func (m *Main) Build(ws *dagger.Workspace) *dagger.Container {
	src := ws.Directory("/", dagger.WorkspaceDirectoryOpts{Exclude: []string{".dagger"}})
	...
	app := ...

+	return dag.Container().
+		From("alpine:latest").
+		WithWorkdir("/app").
+		WithFile("app", app)
}
```

2. Set the entrypoint for the container:

```diff
func (m *Main) Build(ws *dagger.Workspace) *dagger.Container {	
	src := ws.Directory("/", dagger.WorkspaceDirectoryOpts{Exclude: []string{".dagger"}})
	...
	app := ...

	return dag.Container().
		From("alpine:latest").
		WithWorkdir("/app").
		WithFile("app", app).
+		WithEntrypoint([]string{"./app"})
}
```

### Run it!

The resulting build function should look like this:

```go
// Build - builds the source and returns the release container
func (m *Main) Build(ws *dagger.Workspace) *dagger.Container {
	src := ws.Directory("/", dagger.WorkspaceDirectoryOpts{Exclude: []string{".dagger"}})

	app := dag.Container().
		From("golang:latest").
		WithWorkdir("/src").
		WithDirectory(".", src).
		WithExec([]string{"go", "build", "-o", "app", "."}).
		File("app")

	return dag.Container().
		From("alpine:latest").
		WithWorkdir("/app").
		WithFile("app", app).
		WithEntrypoint([]string{"./app"})
}
```

Let's run it:

```sh
dagger call build
```

### Export the image

By default, running the build function simply evaluates a container graph but does nothing more with it.
The container must be exported to the host docker engine.

1. Append the `export-image` command passing in the desired tag:

```sh
dagger call build export-image --name=learn-dagger
```

2. Run the image in docker:

```sh
docker run learn-dagger
```

### Running as a *check*

Most developers don't need the docker image, but may want to *check* the build
is passing. Dagger allows funtions to be annotated as *checks* and will run
all checks via single command: `dagger checks`.

1. Add the `// +check` pragma to our function:

```diff
// Build - builds the source and returns a container with the binary
+ // +check
func (m *Main) Build(ws *dagger.Workspace) *dagger.Container {
```

2. Evaluate the checks:

```sh
dagger check
```