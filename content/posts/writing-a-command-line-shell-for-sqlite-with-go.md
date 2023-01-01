---
title: "Writing a Command Line Shell for Sqlite With Go"
date: 2021-06-18T20:44:45+05:30
draft: false
---

In this guide we are creating a simple command line shell in Golang that would let us create a SQLite DB and run SQL queries in that database. Think of it as a simple version of MySQL CLI application. This is to give you some idea how DB connections in Golang works. Let’s dig in.

## Initializing the project
First of all, let’s create a directory for our application. I’m going to use *nix commands but you can, however, follow this with Windows using GUI.

```bash
mkdir clite
cd clite
go mod init github.com/{your_username}/clite
vim main.go
```

## Creating the shell
To create the shell, we need to import few packages.

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strings"
)
```

And in our main function we should create an infinite loop with an exit condition to handle user input. When getting the user input, it is coming with the terminating newline character, so we have to remove that. Then we are converting it to upper to handle user input easier.

```go
func main() {
   reader := bufio.NewReader(os.Stdin)

   for {
       fmt.Print("> ")
       cmd, _ := reader.ReadString('\n')
       cmd = strings.ToUpper(strings.TrimRight(cmd, "\n"))
      
       if cmd == "EXIT" {
           fmt.Println("Good bye")
           break
       }
   }
}
```

Now we have a basic shell. We have to add few options to the shell to handle creating the DB, running select, and other queries. So it would end up like this. We have to split up the user input by words to check what kind of command the user have entered.

```go
func main() {
   reader := bufio.NewReader(os.Stdin)

   for {
       fmt.Print("> ")
       cmd, _ := reader.ReadString('\n')
       cmd = strings.ToUpper(strings.TrimRight(cmd, "\n"))
      
       if cmd == "EXIT" {
           fmt.Println("Good bye")
           break
       }

       cmdArr := strings.Fields(cmd)
       if len(cmdArr) > 1 {
           if cmdArr[0] == "OPEN" {
                // open database
                continue
           }

           if cmdArr[0] == "SELECT" {
               // run select queries and handle result
              continue
           }

           // handle rest of the queries
       }
   }
}
```

Now the shell part is almost done.

## Handling the SQLite DB and queries
Next we have to import go-sqlite3 package and create the database internal package.

```bash
go get github.com/mattn/go-sqlite3
mkdir -p internal/database  
vim internal/database/database.go
```

Lets import some packages into the database.go

```go
import (
   "database/sql"
   "os"

    _ "github.com/mattn/go-sqlite3"
)
```

Then there should be two functions to create a new database and run some queries.

```go
func CreateDB(db *sql.DB, dbPath string) (sql.DB, error) {
     file, err := os.Create(dbPath)
     if err != nil {
         return nil, err
     }
     file.Close()
     return sql.Open("sqlite3", dbPath)
 }
 ```
 
The CreateDB function gets a sql.DB pointer and a path to DB to be created. It creates the SQLite DB file, and opens it. And returns an error if it cannot be done.

```go
func RunQuery(db *sql.DB, query string) (int64, error) {
     tx, _ := db.Begin()
     statement, err := tx.Prepare(query)
     if err != nil {
         return 0, err
     }
     defer statement.Close()
     res, err := statement.Exec()
     if err != nil {
         tx.Rollback()
         return 0, err
     }
     defer tx.Commit()
     return res.RowsAffected()
 }
```
 
In the RunQuery function, we get the pointer to the DB and the query. We are using transactions to execute the query and if there are any errors, it is going to rollback and return an error. If all went well it will return the number of affected rows.

Okay, lets call those functions from the main.go

```go
import(
    ...
    "github.com/{your_username}/clite/internal/database"
)

func main() {
    reader := bufio.NewReader(os.Stdin)
    var (
        db  *sql.DB
        err error
    )
    dbOpened := false
    for {
        // ...
        if len(cmdArr) > 1 {
            if cmdArr[0] == "OPEN" {
                db, err = database.CreateDB(db, cmdArr[1])
                if err != nil {
                    fmt.Printf("Error opening DB: %v\n", err.Error())
                    continue
                }

               dbOpened = true
                fmt.Printf("Database %v opened\n", cmdArr[1])
                continue
            }

            if cmdArr[0] == "SELECT" && dbOpened {
                continue
            }

            affectedRows, err := database.RunQuery(db, cmd)
            if err != nil {
                fmt.Printf("Error in %v : %v\n", cmd, err.Error())
                continue
            }

            fmt.Printf("%v row(s) affected...\n", affectedRows)
       }
}
```

That’s done. Now we have to deal with the select queries. In Golang, we have to provide the variables which the results from a select query would be assigned to. Since we don’t really know what kind of queries and data will be handled, we have to use the result and interfaces to handle it. If all went well, it will output a result with columns, data. If not it will return an error.

```go
func GetData(db *sql.DB, query string) ([]string, [][]string, error) {
     row, err := db.Query(query)
     if err != nil {
         return []string{}, [][]string{}, err
     }
     defer row.Close()
     columns, _ := row.Columns()
     output := make([][]string, 0)
     rawResult := make([][]byte, len(columns))
     dest := make([]interface{}, len(columns))
 
     for i := range rawResult {
         dest[i] = &rawResult[i]
     }
 
     for row.Next() {
         row.Scan(dest...)
         res := make([]string, 0)
         for _, raw := range rawResult {
             if raw != nil {
                 res = append(res, string(raw))
             }
         }
         output = append(output, res)
     }
     return columns, output, nil
 }
```

Nice. Lets add that to the main.go.

```go
if cmdArr[0] == "SELECT" && dbOpened {
     columns, data, err := database.GetData(db, cmd)
     if err != nil {
         fmt.Printf("Error in %v : %v\n", cmd, err.Error())
         continue
     }
     fmt.Println(columns)
     fmt.Println(data)
     continue
 }
```

Now lets run this

```bash
go run main.go
> OPEN test.db
Database TEST.DB opened
> CREATE TABLE test(id INT, name TEXT)
0 row(s) affected...
> SELECT * FROM test
[ID NAME]
[]
> 
```

Great! It is working. But as you can see the output of the select query is not pretty. Let’s fix that next. To that, we need to create a new package.

## Show output in a table

```bash
go get github.com/olekukonko/tablewriter
mkdir -p internal/display
vim internal/display/display.go
```

```go
package display
import (
    "os"
    "github.com/olekukonko/tablewriter"
)

func PrintTable(columns []string, data [][]string) {
    table := tablewriter.NewWriter(os.Stdout)
    table.SetHeader(columns)
    table.SetAutoFormatHeaders(false)
    for _, v := range data {
        table.Append(v)
    }
    table.Render()
}
```

Here we are importing the tablewriter package and use that to print our result as a table. Now we should add it to the main.go

```go
import (
    //...
    "github.com/dhamith93/clite/internal/display"
)
if cmdArr[0] == "SELECT" && dbOpened {
       columns, data, err := database.GetData(db, cmd)
        if err != nil {
            fmt.Printf("Error in %v : %v\n", cmd, err.Error())
            continue
        }

        display.PrintTable(columns, data)
        continue
}
```

We are done. Let’s build and run this.

```bash
go build
./clite
```

Okay here is the end result.

![](/clite_end.png)

This is a very basic implementation of a SQLite shell. There are plenty of improvements to be done. But I hope this gave you an idea how you can use SQLite DBs for your applications.

You can find the full code at my github: https://github.com/dhamith93/clite

Also, there is a similar project I’ve done with some more features: https://github.com/dhamith93/csv-sql

