# Setting up Go
Whilst us Asgardians currently don't use Go in the development of any of our applications, a lot of our tooling does e.g. Terraform. In some cases, you may need to build some Go projects from source so they can work with Apple Silicon (M1) Macs. This guide will cover setting up your machine with Go.

## Installing Go
The quickest and easiest way to install go on your machine is to grab it from the [Go download page](https://go.dev/dl/). Select the `.pkg` that corresponds to your OS and Arch. In this guide, we want the latest Apple Silicon version which right now is: `go1.18.4.darwin-arm64.pkg`. Once downloaded, run through the installation wizard. Once completed you can verify your installation by running:
```bash
$ go version
go version go1.18.4 darwin/arm64
```
At this point, Go is installed!

## Writing your first Go application
In your code directory (I use `~/git`), create a new directory for your first Go application: 
```bash
mkdir wellickbot && cd $_
```

When your code imports packages contained in other modules, you manage those dependencies through your code's own module. That module is defined by a `go.mod` file that tracks the modules that provide those packages. That go.mod file stays with your code, including in your source code repository.

To enable dependency tracking for your code by creating a `go.mod` file, run the `go mod init` command, giving it the name of the module your code will be in. The name is the module's module path.

For the purposes of this tutorial, just use `example/wellickbot`.

```bash
$ go mod init example/wellickbot
go: creating new go.mod: module example/wellickbot
```

In your text editor of choice, create a new file called `wellickbot.go` where you will write your code.

Paste or type in the following code into your `wellickbot.go` file:
```go
package main

import "fmt"

func main() {
    fmt.Println("Bonsoir, Eliot.")
}
```
This is your Go code. In this code, you:

* Declared a main package (a package is a way to group functions, and it's made up of all the files in the same directory).
* Imported the `fmt` package, which contains functions for formatting text, including printing to the console.
* Implement a main function to print a message to the console. A main function executes by default when you run the main package.

Run your code to see the message:
```bash
$ go run .
Bonsoir, Eliot.
```

Congrats. You just wrote a Go application!

## Setting your $PATH
Whilst we have now created a "Hello World" application, we can only run this is in the context of the directory we have created it in. Using `go install`, we can install our completed application and run it anywhere. So you can make your terminal say "Hello, Friend" from anywhere. How neat!

### Get a list of your Go Environment Variables

### Add Go Environment Variables to your ~/.zshrc
Open your `~/.zshrc` or `~/.bash_profile` and add in the values for your GOPATH, GOROOT and GOBIN environment variables. We then want to add these to our `$PATH`. In my case, this will look like this:
```bash
# GoLang ENV
export GOPATH=$HOME/go
export GOROOT=/usr/local/go
export GOBIN=$GOPATH/bin
export PATH=$PATH:$GOPATH/bin
```

Source your `~/.zshrc` or `bash_profile` or open a new terminal session to consume these changes. You should be all set!

#### Testing your PATH

Remember the `weliickbot` Go application we created up in the Writing your first Go application section. What if you wanted a Tyrell quote no matter where you are in the terminal? We now that we have `$GOBIN` on our `$PATH` we, can call any built go applications that reside in your `$GOBIN` directory.

Navigate back to your `wellickbot` directory and run `go install` to compile and install your `wellickbot` package.

Now if you run `wellickbot` anywhere in the terminal you'll see the message:
```bash
$ wellickbot
Bonsoir, Eliot.
```

