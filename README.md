[![Build Status](https://travis-ci.org/dimiro1/health.svg?branch=master)](https://travis-ci.org/dimiro1/health)
[![Go Report Card](https://goreportcard.com/badge/github.com/dimiro1/health)](https://goreportcard.com/report/github.com/dimiro1/health)
[![GoDoc](https://godoc.org/github.com/dimiro1/health?status.svg)](https://godoc.org/github.com/dimiro1/health)


# Go Health Check

A Easy to use, extensible health check library

**Table of Contents**

- [Example](#example)
- [Motivation](#motivation)
- [Inspiration](#inspiration)
- [Instalation](#instalation)
- [API](#api)
- [Testing](#testing)
- [Implementing custom checkers](#implementing-custom-checkers)
- [Implemented health check indicators](#implemented-health-check-indicators)
- [LICENSE](#license)

# Example

```go
package main

import (
    "net/http"
    "database/sql"

    "github.com/dimiro1/health"
    "github.com/dimiro1/health/url"
    "github.com/dimiro1/health/db"
    _ "github.com/go-sql-driver/mysql"
)

func main() {
    database, _ := sql.Open("mysql", "/")
	mysql := db.NewMySQLChecker(database)
    
    companies := health.NewCompositeChecker()
    companies.AddChecker("Microsoft", url.NewChecker("https://www.microsoft.com/"))
    companies.AddChecker("Oracle", url.NewChecker("https://www.oracle.com/"))
    companies.AddChecker("Google", url.NewChecker("https://www.google.com/"))

    handler := health.NewHandler()
    handler.AddChecker("Go", url.NewChecker("https://golang.org/"))
    handler.AddChecker("Big Companies", companies)
    handler.AddChecker("MySQL", mysql)

    http.Handle("/health/", handler)
    http.ListenAndServe(":8080", nil)
}
```

```sh
$ curl localhost:8080/health/
```

If everything is ok the server must return the following json.

```json
{
    "Big Companies": {
        "Google": {
            "code": 200,
            "status": "UP"
        },
        "Microsoft": {
            "code": 200,
            "status": "UP"
        },
        "Oracle": {
            "code": 200,
            "status": "UP"
        },
        "status": "UP"
    },
    "Go": {
        "code": 200,
        "status": "UP"
    },
    "MySQL": {
        "status": "UP",
        "version": "10.1.9-MariaDB"
    },
    "status": "UP"
}
```

# Motivation

It is very important to verify the status of your system, not only the system itself, but all its dependencies, 
If your system is not Up you can easily know what is the cause of the problem only looking the health check.

Also it serves as a kind of basic itegration test between the systems.

# Inspiration

I took a lot of ideas from the [spring framework](http://spring.io/).

# Instalation

This package is a go getable packake.

```sh
$ go get github.com/dimiro1/health
```

# API

The API is stable and I do not have any plans to break compatibility, but I recommend you to vendor this dependency in your project, as it is a good practice.

# Testing

You have to install the test dependencies.

```sh
$ go get gopkg.in/DATA-DOG/go-sqlmock.v1
```

or you can go get this package with the -t flag

```sh
go get -t github.com/dimiro1/health
```

# Implementing custom checkers

The key interface is health.Checker, you only have to implement a type that satisfies that interface.

```go
type Checker interface {
	Check() Health
}
```

Here an example of Disk Space usage (unix only).

```go
package main

import (
    "syscall"
    "os"
)

type DiskSpaceChecker struct {
	Dir       string
	Threshold uint64
}

func NewDiskSpaceChecker(dir string, threshold uint64) DiskSpaceChecker {
	return DiskSpaceChecker{Dir: dir, Threshold: threshold}
}

func (d DiskSpaceChecker) Check() health.Health {
	health := health.NewHealth()
	health.Up() // Default is Down

	var stat syscall.Statfs_t

	wd, err := os.Getwd()

	if err != nil {
		health.Down()
		health.Info = map[string]interface{}{
			"error": err.Error(), // Why the check is Down 
		}
        
        return health
	}

	syscall.Statfs(wd, &stat)

	diskFreeInBytes := stat.Bavail * uint64(stat.Bsize)

	if diskFreeInBytes < d.Threshold {
		health.Down()
	}

	health.Info = map[string]interface{}{
		"free":      diskFreeInBytes,
		"threshold": d.Threshold,
	}

	return health
}
```

## important

The **status** key in the json have priority over a **status** key added by a Checker, so if some checker add a **status** key to the json, it will not be rendered  

# Implemented health check indicators

| Health      | Description                            | Package                                           |
|-------------|----------------------------------------|---------------------------------------------------|
| url.Checker | Check the connection with some URL     | https://github.com/dimiro1/health/tree/master/url |
| db.Checker  | Check the connection with the database | https://github.com/dimiro1/health/tree/master/db  |

# LICENSE

The MIT License (MIT)

Copyright (c) 2016 Claudemiro

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
