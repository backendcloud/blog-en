release time :2022-04-13 20:17

> When managing multi-module management, some modules may still be under development and have not been published on github. Before Go 1.18, it was done through the replace of go mod. The go1.18 officially released in February 2022 provides another more convenient solution for multi-module management due to the new workspace feature.

# Past replace pattern
```bash
hanwei@hanweideMacBook-Air golang]$ go version
go version go1.18 darwin/arm64
hanwei@hanweideMacBook-Air golang]$ mkdir go1.18-workspace/
hanwei@hanweideMacBook-Air golang]$ cd go1.18-workspace/
hanwei@hanweideMacBook-Air go1.18-workspace]$ mkdir mypkg example
hanwei@hanweideMacBook-Air go1.18-workspace]$ tree
.
├── example
└── mypkg

2 directories, 0 files
hanwei@hanweideMacBook-Air go1.18-workspace]$ go mod init github.com/go1.18-workspace/mypkg
go: creating new go.mod: module github.com/go1.18-workspace/mypkg
go: to add module requirements and sums:
        go mod tidy
hanwei@hanweideMacBook-Air go1.18-workspace]$ cd mypkg/
hanwei@hanweideMacBook-Air mypkg]$ touch bar.go
hanwei@hanweideMacBook-Air mypkg]$ vi bar.go 
hanwei@hanweideMacBook-Air mypkg]$ cat bar.go 
package mypkg

func Bar() {
        println("This is package mypkg")
}
hanwei@hanweideMacBook-Air mypkg]$ cd ../example/
hanwei@hanweideMacBook-Air example]$ go mod init github.com/go1.18-workspace/example
go: creating new go.mod: module github.com/go1.18-workspace/example
go: to add module requirements and sums:
        go mod tidy
hanwei@hanweideMacBook-Air example]$ touch main.go
hanwei@hanweideMacBook-Air example]$ vi main.go 
hanwei@hanweideMacBook-Air example]$ cat main.go 
package main

import (
    "github.com/go1.18-workspace/mypkg"
)

func main() {
    mypkg.Bar()
}
```

At this time, if we run go mod tidy, an error will be reported, because our mypkg package has not been submitted to github at all, and it must not be found.

    hanwei@hanweideMacBook-Air example]$ go mod tidy
    go: finding module for package github.com/go1.18-workspace/mypkg
    github.com/go1.18-workspace/example imports
            github.com/go1.18-workspace/mypkg: cannot find module providing package github.com/go1.18-workspace/mypkg: module github.com/go1.18-workspace/mypkg: git ls-remote -q origin in /Users/hanwei/GoProjects/pkg/mod/cache/vcs/2c423cac5ebc1b2d018ef93a87560d369abd7dec6c155b46cddb11299415bc09: exit status 128:
            remote: Repository not found.
            fatal: repository 'https://github.com/go1.18-workspace/mypkg/' not found
    hanwei@hanweideMacBook-Air example]$ go run main.go
    main.go:4:5: no required module provides package github.com/go1.18-workspace/mypkg; to add it:
            go get github.com/go1.18-workspace/mypkg


go run main.go is also unsuccessful.

Of course we can submit mypkg to github, but every time we modify mypkg, we need to submit, otherwise the latest version cannot be used in the example.

In view of this situation, it is currently recommended to solve it by replace, that is, add the following replace to the go.mod in the example:

    hanwei@hanweideMacBook-Air example]$ go mod edit -replace=github.com/go1.18-workspace/mypkg=../mypkg
    hanwei@hanweideMacBook-Air example]$ cat go.mod 
    module github.com/go1.18-workspace/example

    go 1.18

    replace github.com/go1.18-workspace/mypkg => ../mypkg

(v1.0.0 is modified according to the specific situation, it has not been submitted yet, you can use v1.0.0)

    hanwei@hanweideMacBook-Air example]$ cat go.mod 
    module github.com/go1.18-workspace/example

    go 1.18

    require github.com/go1.18-workspace/mypkg v1.0.0
    replace github.com/go1.18-workspace/mypkg => ../mypkg
    hanwei@hanweideMacBook-Air example]$ tree ..
    ..
    ├── example
    │   ├── go.mod
    │   └── main.go
    └── mypkg
        ├── bar.go
        └── go.mod

    2 directories, 4 files


Run go run main.go again, the output is as follows:

    hanwei@hanweideMacBook-Air example]$ go run main.go 
    This is package mypkg

# workspace mode
Comment the replace above, and then execute go run main.go, and report an error

    hanwei@hanweideMacBook-Air example]$ cat go.mod 
    module github.com/go1.18-workspace/example

    go 1.18

    //require github.com/go1.18-workspace/mypkg v1.0.0
    //replace github.com/go1.18-workspace/mypkg => ../mypkg
    hanwei@hanweideMacBook-Air example]$ go run main.go 
    main.go:4:5: missing go.sum entry for module providing package github.com/go1.18-workspace/mypkg; to add:
            go mod download github.com/go1.18-workspace/mypkg


Initialize workspace

    hanwei@hanweideMacBook-Air go1.18-workspace]$ go version
    go version go1.18 darwin/arm64
    hanwei@hanweideMacBook-Air go1.18-workspace]$ go work init example mypkg
    hanwei@hanweideMacBook-Air go1.18-workspace]$ tree
    .
    ├── example
    │   ├── go.mod
    │   └── main.go
    ├── go.work
    └── mypkg
        ├── bar.go
        └── go.mod

    2 directories, 5 files
    hanwei@hanweideMacBook-Air go1.18-workspace]$ cat go.work 
    go 1.18

    use (
            ./example
            ./mypkg
    )
    hanwei@hanweideMacBook-Air example]$ go run main.go 
    This is package mypkg

It can be seen that using the workspace is much more convenient than replace.

> The syntax of the go.work file is similar to go.mod, so it also supports replace.

> Note that go.work does not need to be committed to Git, as it is only used for local development.

    cat .gitignore
    # ---> Go
    # If you prefer the allow list template instead of the deny list, see community template:
    # https://github.com/github/gitignore/blob/main/community/Golang/Go.AllowList.gitignore
    #
    # Binaries for programs and plugins
    *.exe
    *.exe~
    *.dll
    *.so
    *.dylib
    # Test binary, built with `go test -c`
    *.test
    # Output of the go coverage tool, specifically when used with LiteIDE
    *.out
    # Dependency directories (remove the comment below to include it)
    # vendor/
    # Go workspace file
    go.work

> In the GOPATH era, multiple GOPATHs are a headache. At that time, there was no good solution, and the Module appeared, so the multi-GOPATH problem disappeared. But the multi-Module problem then appeared. The Workspace solution better solves this problem.
