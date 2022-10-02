---
layout: post
title: How to get started with Go (tested on macOS, Ubuntu 20.04, WSL Ubuntu 22.04)
date: 2022-10-01 21:33 -0400
tags: go
img_path: ../../images
---

## Introduction

I wanted to try out [Go](https://go.dev/) and the first step was to install it. Unlike other languages I've used, say python for example, I had an extremely hard time installing it. Installing Go itself was simple, but when I'd go to use packages, things just didn't see to work as I expected.  In a few different cases I ended up uninstalling Go and giving up. But I really wanted to try Go, so I did some research and from a few different articles, I was able to piece a process together that worked perfectly on both my Ubuntu 20.04 laptop and my M1 MacBook Pro. For this article, I am going to be installing the latest version of Go on my WSL 2 Ubuntu 22.04 distro.

## Installation

1) Download the latest version of Go from their [website](https://go.dev/dl/)

```shell
curl -OL https://go.dev/dl/go1.19.1.linux-amd64.tar.gz
```

Optional but encouraged, verify the integrity of the file you downloaded by comparing the checksum to the one on the Go releases page:

```shell
sha246sum https://go.dev/dl/go1.19.1.linux-amd64.tar.gz
# output
acc512fbab4f716a8f97a8b3fbaa9ddd39606a28be6c2515ef7c6c6311acffde go1.19.1.linux-amd64.tar.gz
```

![Go releases page](go_release_pg.png)

2) Extract the contents of the file and move them to a location in your $PATH (e.g. `/usr/local`)

```shell
sudo tar -C /usr/local -xvf go1.19.1.linux-amd64.tar.gz
```

1) Set the Go path for your environment

```shell
# Open up your .bashrc, .zshrc, or .profile
nano ~/.zshrc

# Add Go's root value to your path by adding this info to the end of the file
export PATH=$PATH:/usr/local/go/bin

# refresh from session
source ~/.zshrc
```

4) Confirm Go is installed

```shell
go version
# output
go version go1.19.1 linux/amd64
```

## Go Workspace

1) Create your Go workspace where you'll build and store your projects.
*Note: you can call this directory whatever you like and store wherever works best for you*

```shell
# create workspace directory
mkdir ~/go
```

2) Set your Go workspace as your $GOPATH  in your environment

```shell
# Open up your .bashrc, .zshrc, or .profile
nano ~/.zshrc

# Add Go's root value to your path by adding this info to the end of the file
export GOPATH=$HOME/go

# refresh from session
source ~/.zshrc
```

3) Within the Go workspace directory create a folder called `src`

```shell
cd go/
mkdir src
cd src/
```

4) Within the `src` folder create your first project; we'll call ours `hello`

```shell
mkdir hello
cd hello/
```

5) Within the `hello` directory create a module file for managing dependencies by running
*Note: for 'your_domain' I used my github username*

```shell
go mod init your_domain/hello
```

After running this you will see you now have a `go.mod` file in your folder
6) Next create a simple hello world application

```shell
nano hello.go

#contents
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```

7) Test that the file runs

```shell
go run hello.go
# output
Hello, World!
```

## Create Binary Executable

The beauty of Go, in my opinion, is the ease at which you can create a binary executable and enable your program so that it can run from anywhere on your system. Here is how we do this.

1) Within the `hello` directory compile your program into an executable binary

```shell
go build
```

2) You'll now have an executable file in your folder that can run with

```shell
./hello
# output
Hello, World!
```

You've successfully turned your code into an executable binary. However, your code can only be run from this directory. You could run the program from another location by passing the full path to the executable but that could quickly become annoying. Instead, Go provide us with a way to install the program so that it can be run from anywhere.
3) From within the `hello` folder Install your program by running

```shell
go install
```

This creates a `bin` folder within our $GOPATH and installs the binary executable there.
4) Add the location of this folder to your $PATH

```shell
# Open up your .bashrc, .zshrc, or .profile
nano ~/.zshrc

# Add Go's root value to your path by adding this info to the end of the file
export PATH=$PATH:$HOME/go/bin/

# refresh from session
source ~/.zshrc
```

5) Change to a random directory and try and run the executable

```shell
cd $HOME
hello
# output
Hello, World
```

Hopefully, everything worked and you got the expected output. You've now successfully installed Go. Happy coding!

## Extra Credit

If you want to double check that go is installed correctly then try these additional steps.

1) Create a new project within your Go workspace

```shell
cd /go/src/
mkdir newProject
```

2) Initialize the project by creating a `go.mod` file

```shell
go mod init github.com/git_username/newProject
```

Now, let's make sure we can install and run a library. In this case, I'm going to install a library for making modern CLI applications called [Cobra](https://github.com/spf13/cobra). Prior to finding the "correct" install method, I would install the library, attempt to initialize my project with it, but nothing would happen. Here's what should happen.
3) First, install the library from within our project folder

```shell
go install github.com/spf13/cobra-cli@latest
```

The library may take a moment to install.
4) Cobra has its own CLI, verify it was installed correctly by running

```shell
cobra-cli
# output
Cobra is a CLI library for Go that empowers applications.

This application is a tool to generate the needed files

to quickly create a Cobra application.

Usage:

cobra-cli [command]
...
```

If Go is installed correctly then you should see output from Cobra explaining how to use it. This is the part that not working for me before. I would run `cobra-cli` and nothing would happen. Once I figured out how to properly install and configure Go, then everything worked as it should.

5) Finally, initialize our new project with Cobra

```shell
cobra-cli init
```

Now you can build your own super awesome CLI tool.

6) At this point our go workspace should be made up of the following directories

```shell
tree -L 1
# output
.
├── bin
├── pkg
└── src
```

I hope you found something in this post helpful and that you have a blast coding in Go.
