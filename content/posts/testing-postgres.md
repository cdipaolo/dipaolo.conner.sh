+++
date = "2015-08-11T15:53:51-07:00"
title = "quick and efficiently unit testing Postgres interfaces in Go for better scalability"

+++

Writing tests isn't always fun. Having your backend break when you change something three functions deep is less fun. Most of the time the Go compiler will fix mistakes for you, but what about when you a parameter to a `SELECT this, and, that FROM table` query in Postgres? Variadic arguments aren't checked at compile time, and integration tests won't help with isolate functions at the bottom level. That's why writing testable code is always better than it's more 'agile' counterpart.

This post will walk you through writing data access layer functions that are testable, tested, and easy to extend. Let's begin...

## getting started

We're going to assume our database has a schema called 'test' with a table called 'users' with two fields, a Primary Key serial as 'user_id' and a username (called such) as text.

Say we have a simple program that interfaces with a postgres database, upon running it inserts a new customer, then selects all customers and prints them to STDOUT. We're going to keep this all in one file, because this is a simple program. In most cases when working with databases in Go, you'll have a global `*sql.DB` struct that you'll query from. Let's write the functionality first using that methodology:

This is main.go.

```go
package main

import (
    "database/sql"
    "fmt"
    "os" // we're going to be storing our PG URL
         // as an environment variable

    _ "github.com/lib/pq" // need to import database driver
)

var globalDB = PostgresDBFactory()

// PostgresDBFactory connects to the postgres
// DB using an env var for the URL and returns
// the connected *sql.DB
func PostgresDBFactory() *sql.DB {
    PostgresURL := os.GetEnv("PG_URL")

    db, err := sql.Open("postgres", PostgresURL)
    if err != nil {
        panic(fmt.Sprintf("ERROR: error connecting to postgres database!\n\t%v\n", err))
    }

    // ping database to make sure everything's good
    err = db.Ping()
    if err != nil {
        panic(fmt.Sprintf("ERROR: error pinging postgres database postgres database!\n\t%v\n", err))
    }

    // now we're all good. Return DB
    return db
}

// now let's declare what a user _is_.

// User holds just a UserID and
// it's corresponding Username
type User struct {
    UserID   int64  `json:"user_id,omitempty"`
    Username string `json:"username,omitempty"`
}

// now we're going to define functions about how to add and get users

// GetUsers gets all users from the database
// and returns them as a slice
func GetUsers() ([]*User, error) {
    // query database
    rows, err := globalDB.Query("SELECT user_id, username FROM test.users")
    if err != nil {
        return nil, err
    }

    var users []*User
    for rows.Next() {
        user := &User{}
        err = rows.Scan(&user.UserID, &user.Username)
        if err != nil {
            return nil, err
        }
        users = append(user, users)
    }

    if err = rows.Err(); err != nil {
        return nil, err
    }

    return users
}

// InsertUser inserts a user into the database
// having a username as given
func InsertUser(username string) error {
    _, err := globalDB.Exec("INSERT INTO test.users (username) VALUES ($1)", username)
    return err
}

// now let's define our program
func main() {
    // give the user a username of the current time
    err := InsertUser(fmt.Sprintf("%v", time.Now().UTC()))
    if err != nil {
        panic(err.Error())
    }

    // return all users
    users, err := GetUsers()
    if err != nil {
        panic(err.Error())
    }

    fmt.Printf("Users:\n\t%v\n", users)
}
```

## the problem

Ok that's great, but how do we test it? Conventional wisdom would say to run a database while you're testing, and query against that database for expected output after the program runs. That's great, but now you have to run a database at the same time. Note that you should end up doing that as well, but use unit tests to provide quick checks of your code.

But back to our little problem – traditional mocking sucks. And especially in Go. We want to test how our code behaves to different outputs by the database without the overhead of running a databaseseparately and querying it. This is where [`github.com/DATA-DOG/go-sqlmock`](https://github.com/DATA-DOG/go-sqlmock) comes in. It lets you create mock database which expect queries and return expected results to those queries. Find and dandy, but how are we going to load that into our functions!? We could redefine the global database, but then we would be dependent on the order tests are being run and then we wouldn't be able to test in parallel!

Luckily we have a simple fix. Add in a database parameter to all the functions which, if nil, will just set to the global database. This is a simple fix that will let you take advantage of easy testing and allow for easy integration with Master-Slave schemes down the line (this application will get _that_ much traffic!)

## the fix

So let's integrate those changes into our application!

Replace the following functions:

```go
func GetUsers(db *sql.DB) ([]*User, error) {
    if db == nil {
        db = globalDB
    }

    // query database
    rows, err := db.Query("SELECT user_id, username FROM test.users")
    if err != nil {
        return nil, err
    }

    var users []*User
    for rows.Next() {
        user := &User{}
        err = rows.Scan(&user.UserID, &user.Username)
        if err != nil {
            return nil, err
        }
        users = append(user, users)
    }

    if err = rows.Err(); err != nil {
        return nil, err
    }

    return users
}

func InsertUser(username string, db *sql.DB) error {
    if db == nil {
        db = globalDB
    }

    _, err := globalDB.Exec("INSERT INTO test.users (username) VALUES ($1)", username)
    return err
}

// now we have to call differently from main()
func main() {
    // give the user a username of the current time
    err := InsertUser(fmt.Sprintf("%v", time.Now().UTC()), nil)
    if err != nil {
        panic(err.Error())
    }

    // return all users
    users, err := GetUsers(nil)
    if err != nil {
        panic(err.Error())
    }

    fmt.Printf("Users:\n\t%v\n", users)
}
```

That's it only 10 lines changed!

## testing postgres effectively

Let's write comprehensive tests for just one of our two handlers, `GetUsers(db)` (it's more intricate.) Testing `InsertUser(username, db)` is the same, except instead of `ExpectQuery` you would use `ExpectExec`.

I'll be using [github.com/stretchr/testify](https://github.com/stretchr/testify) because it'll clean things up, but any testing package will work (or just go in raw!)

This is unit_test.go:

```go
package main

import (
    "testing"
    "fmt"

    sqlmock "github.com/DATA-DOG/go-sqlmock"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

var userCols = []string{"user_id", "username"}

func TestGetUsersShouldPass1(t *testing.T) {
    db, err := sqlmock.New()
    assert.Nil(t, err, "opening database connection should work")
    require.NotNil(t, db, "mock database should not be nil")

    // declare expected queries
    sqlmock.ExpectQuery("SELECT (.+) FROM test.users").WillReturnRows(sqlmock.NewRows(userCols).FromCSVString("10, username_here\n11, another_user"))

    users, err := GetUsers(db)
    assert.Nil(t, err, "postgres error should be nil")

    // test content of returned values
    require.Equal(t, 2, len(users), "users array should not be nil")
    
    // test first user
    assert.Equal(t, int64(10), users[0].UserID, "user_id should match")
    assert.Equal(t, "username_here", users[0].Username, "username should match")

    // test second user
    assert.Equal(t, int64(11), users[1].UserID, "user_id should match")
    assert.Equal(t, "another_user", users[1].Username, "username should match")
}

func TestGetUsersShouldPass2(t *testing.T) {
    db, err := sqlmock.New()
    assert.Nil(t, err, "opening database connection should work")
    require.NotNil(t, db, "mock database should not be nil")

    // declare expected queries
    sqlmock.ExpectQuery("SELECT (.+) FROM test.users").WillReturnRows(sqlmock.NewRows(userCols).FromCSVString(""))

    users, err := GetUsers(db)
    assert.Nil(t, err, "postgres error should be nil")
    assert.Equal(t, 0, len(users), "users array should be nil")
}

func TestGetUsersShouldFail1(t *testing.T) {
    db, err := sqlmock.New()
    assert.Nil(t, err, "opening database connection should work")
    require.NotNil(t, db, "mock database should not be nil")

    // declare expected queries
    sqlmock.ExpectQuery("SELECT (.+) FROM test.users").WillReturnRows(sqlmock.NewRows(userCols).FromCSVString("SCAN ERROR, username_here"))

    users, err := GetUsers(db)
    assert.NotNil(t, err, "postgres error should not be nil")
    assert.Equal(t, 0, len(users), "users array should be nil")
}

func TestGetUsersShouldFail2(t *testing.T) {
    db, err := sqlmock.New()
    assert.Nil(t, err, "opening database connection should work")
    require.NotNil(t, db, "mock database should not be nil")

    // declare expected queries
    sqlmock.ExpectQuery("SELECT (.+) FROM test.users").WillReturnError(fmt.Errorf("Query Error"))
    users, err := GetUsers(db)
    assert.NotNil(t, err, "postgres error should not be nil")
    assert.Equal(t, 0, len(users), "users array should be nil")
}
```

## conclusion

And there we have it! Really good test coverage (I don't think you even _can_ test that last little 'if not nil' error check for multi-row selects in Go). That runs **really**, **really** fast. Going through tests of the entire backend I've been working on takes only about 10-15s for unit tests, and 100+s for integration tests. This won't make sure your queries _work_, but it will let you know what your function is handling things well internally, before opening up dependency hell.

As briefly mentioned before, if you want to run multiple databases in a Master-Slave scenario (where only the Read DB takes computation-heavy queries) then this setup lets you scale to that extremely easily by just connecting to multiple databases!
