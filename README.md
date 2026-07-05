# Learn Dagger

This tutorial will run demonstrate how to use [dagger](http://dagger.io) to build a simple go app.

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