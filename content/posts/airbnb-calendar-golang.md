---
date: "2018-01-16T15:41:15Z"
title: "Airbnb calendar integration with golang"
authors: []
tags:
  - go
draft: false
toc: true
typora-copy-images-to: ../../static/imgs
typora-root-url: ../../static
---

## Background

This is for a client of mine at [Fluid Media](https://fluidmedia.wales), they run a holiday rental in Scotland. The problem being solved here is that they list on both [Airbnb](https://airbnb.com) and [their website](http://laganglas.co.uk), but the two booking calendars are separate. So we need to come up with a way of synchronizing both. Luckily there is a solution.

Initially I had a look for an API, and while they do have one it's only open to large property management companies. Boo hiss. After some googling I found that they allow exporting of the booking calender via a HTTPS endpoint in iCal format, so thats one direction done. The other thing they allow is importing from a HTTP(S) exposed iCal into the listings calendar. Woop, problem solved. Now it's time to implement.

## Step 1: Loading in the listing's calendar

I created a blank listing on Airbnb for testing, they probably hate me now but hey. I set the calendar to be completely blocked so no one would try and book at my (fake) listing. We now go into the listing availability section and find this bit:

![Screenshot-2018-1-16 Edit Availability for ‘Test’ - Airbnb](/imgs/Screenshot-2018-1-16 Edit Availability for ‘Test’ - Airbnb.png)

This contains every bit we'll need to touch to set this up. Click on the **Export Calendar** link and you'll get a URL that looks something like this: `https://www.airbnb.co.uk/calendar/ical/xxxxxxxx.ics?s=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` where all the `x` values are different for each property. When I `curl` this I get this output.

```
BEGIN:VCALENDAR
PRODID;X-RICAL-TZSOURCE=TZINFO:-//Airbnb Inc//Hosting Calendar 0.8.8//EN
CALSCALE:GREGORIAN
VERSION:2.0
BEGIN:VEVENT
DTEND;VALUE=DATE:20190117
DTSTART;VALUE=DATE:20170116
UID:f7xyme6s3fww--dfhqc1b0cf0t@airbnb.com
SUMMARY:Not available
END:VEVENT
END:VCALENDAR
```

Let's explain the important lines.

The `BEGIN:VCALENDAR` and `END:VCALENDAR` show us where all the calendar data is contained and also identifies the format.

The `BEGIN:VEVENT` and `END:VEVENT` show us where each block of data for each event starts and ends. In this case we juts have one blocking out the entire calendar.

The `DTSTART;VALUE=DATE:20170116` and `DTEND;VALUE=DATE:20190117` within the event block show us when this event starts and ends. The date is formatted in a special iCal format but it is still readable. 

`UID:f7xyme6s3fww--dfhqc1b0cf0t@airbnb.com` gives us unique value we can use later while storing all of this.

And finally `SUMMARY:Not available` gives us a description we can show in the calender. I'm not aware how this changes if bookings have been made but it might show personal information so I won't be displaying it externally.

### Accessing the calendar from go

To load this in we can use the `net/http ` library from go. We can then simply call `http.Get` to load in the data. First we import the library.

```go
import (
    "log"
	"net/http"
)
```

Then in our main function we can load in the calendar. We can simply panic on error since this is just test code, but DON'T EVER DO THIS IN PRODUCTION! You have been warned.

```go
func main() {
    // Pull url
	response, err := http.Get("[INSERT_URL_HERE]")
    if err != nil {
    	log.Fatal(err)
    } else {
		defer response.Body.Close()
        // The data is now in the file line response.Body
    }
}
```

We now have the data but how shall one parse said data? Thou shall find help in `github.com/lestrrat/go-ical` (Apologies for the Shakespeare haters). This is an amazing community library that assists in the parsing and generating of iCal data. First we add this to the imports.

```go
import (
    "log"
	"net/http"
    "github.com/lestrrat/go-ical"
)
```

Then we can create a new parser and pass it the body. Again we can panic on error here but don't really do it.

```go
p := ical.NewParser()
// Parse file like body
c, err := p.Parse(response.Body)
if err != nil {
  log.Fatal(err)
}
```

The variable `c` now contains all the data from the calendar. We can iterate over all the entries with this code. Note that we simply continue on an error converting the entry to an event because other types of entries are allowed in iCal but we don't care about them.

```go
for e := range c.Entries() {
  // Try to convert entry into event
  ev, ok := e.(*ical.Event)
  // It's ok if it fails
  if !ok {
    continue
  }
}
```

Finally we can access the the previously discussed information about the event.

```go
// Get properties of each event
start, _ := ev.GetProperty("dtstart")
end, _ := ev.GetProperty("dtend")
uid, _ := ev.GetProperty("uid")
summary, _ := ev.GetProperty("summary")
// Print out values for testing
fmt.Print(start.Name(), " ", start.RawValue(), "\r\n")
fmt.Print(end.Name(), " ", end.RawValue(), "\r\n")
fmt.Print(uid.Name(), " ", uid.RawValue(), "\r\n")
fmt.Print(summary.Name(), " ", summary.RawValue(), "\r\n")
```

This prints out the following for our previously shown iCal data.

```
dtstart 20170116
dtend 20190117
uid i5miubnnbs7t--w5spgm8tdkiy@airbnb.com
summary Not available
```

As you can see we have successfully extracted the relevant information.

## Step 2: Setting up a database for storing the data

I chose to use a postgres database for storing all the calendar events due to it's support in Go and ease of deployment in Kuberetes. It also was a requirement that multiple instances could connect to allow multiple containers to run for a rolling update. I setup a server on my PC for testing. You can install the `postgresql` package with you choice of package manager, then you have to set a password for the `postgres` user. This can be done by `sudo`ing to the `postgres` system account and running psql. While where here well also create a database for our program.

```bash
sudo -u postgres psql postgres
postgres=# ALTER USER postgres WITH PASSWORD 'CHOOSE A PASSWORD';
postgres=# CREATE DATABASE airbnb_cal;
postgres=# \q
```

First we must connect to an postgres instance. Go has built in support for SQL databases so we need to include that and a database driver `github.com/lib/pq` in this case. So our import now looks like this.

```go
import (
   "log"
   "fmt"
   "net/http"
   "github.com/lestrrat/go-ical"
   "database/sql"
   // _ import beacause we only want it's side effects
   _ "github.com/lib/pq"
)
```

Then we create a `struct` to store the connection and that will have functions on it to allow communication to the database.

```go
type pgDb struct {
    dbConn *sql.DB
}
```

Finally we actually connect. The connect string only works here for instances running on the same host/container. We'll put this in a function to be called from `main`.

```go
func initDb() (*pgDb, error) {
  // Open connection to server
  if dbConn, err := sql.Open("postgres", "user=postgres password=PASSWORD FROM EARLIER host=127.0.0.1 dbname=airbnb_cal"); err != nil {
    return nil, err
  } else {
    // Create struct
    p := &pgDb{dbConn: dbConn}
    // Check the connection actually works
    if err := p.dbConn.Ping(); err != nil {
      return nil, err
    }
    return p, nil
  }
}
```

To initialize the database we need to add a function to `pgDb` called `createTablesIfNotExist` in this case. All it does is run one SQL query to setup the schema.

```go
func (p *pgDb) createTablesIfNotExist() error {
	createSql := `
       CREATE TABLE IF NOT EXISTS events (
       uid TEXT NOT NULL PRIMARY KEY,
       summary TEXT NOT NULL
       dtstart TIMESTAMP NOT NULL,
       dtend TIMESTAMP NOT NULL);`
    // Run query
	if rows, err := p.dbConn.Query(createSql); err != nil {
		return err
	} else {
		rows.Close()
	}
	return nil
}
```

Then in `main` we call `initDb` and `createTablesIfNotExist` on the returned database object.

```go
db, err := initDb()
if err != nil {
  log.Fatalf("Error initializing database: %v\n", err)
}

err = db.createTablesIfNotExist()
if err != nil {
  log.Fatalf("Error creating database tables: %v\n", err)
}
```

Our database is now setup to be used for storing

## Step 3: Store data

Now that we have access to a database and calendar events we'll create a `insertOrUpdateEvent` function on the database. This will check if the event with that data already exists and if not inserts otherwise updates if any values have changed. We can't use UID because Airbnb changes it on every request. We need to also import `time` here since this is where we convert from the iCal format to the database format.

```go
// Strcut can have member fucntions
func (p *pgDb) insertOrUpdateEvent(uid *ical.Property, start *ical.Property, end *ical.Property, summary *ical.Property) error {
   lookupSql := `
        SELECT uid FROM events
        WHERE dtstart = $1 AND dtend = $2
    `
    exists := true
    var oldSummary, oldUid string
    var oldStart, oldEnd time.Time
}
```

We now need to convert the iCal date to a go `time` format. First we'll define the format that go will use to parse the date. To do this we need to give go the specific date of **Mon Jan 2 2006** in the format to be parsed.

```go
const iCalDateFormat = "20060102"
```

We can then call `time.Parse(layout, input)` to get the date out. The following code can be added to the `insertOrUpdateEvent` function.

```go
newStart, err := time.Parse(iCalDateFormat, start.RawValue())
if err != nil {
  return err
}
newEnd, err := time.Parse(iCalDateFormat, end.RawValue())
if err != nil {
  return err
}
```

We can then query the database with out calculated values to find if an event already exists.

```go
r := p.dbConn.QueryRow(lookupSql, newStart, newEnd)
// This pulls the data from each row into variables
err = r.Scan(&oldUid, &oldSummary, &oldStart, &oldEnd)
if err == sql.ErrNoRows {
  exists = false
} else if err != nil {
  return err
}
```

Now if a record with that data doesn't already exist we'll need to insert one.

```go
if !exists {
  insertSql := `
        INSERT INTO events
        (uid, dtstart, dtend, summary)
        VALUES ($1, $2, $3, $4)
  `
  // Run the insert
  _, err := p.dbConn.Exec(insertSql, uid.RawValue(), newStart, newEnd, summary.RawValue())
  if err != nil {
    return err
  }
}
```

Finally if a record does exist we check if anything has changed and update otherwise we do nothing.

```go
// Check if anything changed
if newStart != oldStart || newEnd != oldEnd || summary.RawValue() != oldSummary {
  updateSql := `
        UPDATE events
        SET dtstart = $3, dtend = $4, summary = $5, uid = $2
        WHERE uid = $1
  `
  // Run the update
  _, err := p.dbConn.Exec(updateSql, oldUid, uid.RawValue(), newStart, newEnd, summary.RawValue())
  if err != nil {
    return err
  }
}
```

We can now use this function to insert calendar events. This replaces where we used to print out the event.

```go
start, _ := ev.GetProperty("dtstart")
end, _ := ev.GetProperty("dtend")
uid, _ := ev.GetProperty("uid")
summary, _ := ev.GetProperty("summary")
err := db.insertOrUpdateEvent(uid, start, end, summary)
if err != nil {
  log.Fatalf("Can't insert event: %v\n", err)
}
```

And if we look in the database we can see the event inserted.

![Screenshot_2018-01-17_07-12-11](/imgs/Screenshot_2018-01-17_07-12-11.png)

## Step 4: Export events to Airbnb

So we have half of the integration done. All events from Airbnb are in the database, we now need to get all events from the database into airbnb. The iCal library I chose also supports encoding data to iCal so all we need to do is run a HTTP server that pulls the data from the database and gives it to iCal to be written out to the client. So we'll import `net/http` and `github.com/gorilla/mux` as out router. We now need to setup our HTTP server.

```go
// Create router
router := mux.NewRouter().StrictSlash(true)
// Only accept GET at /calendar/ical.ics requests
router.Methods("GET").Path("/calendar/ical.ics").HandlerFunc(handleCalendar)
// Start the server and if it exits log the error
log.Fatal(http.ListenAndServe(":8080", router))
```

We now need to write this `handleCalendar` function. This takes a `ResponseWriter` to write back to the client and a `Request` with info on the request made. 

```go
func handleCalendar(w http.ResponseWriter, r *http.Request) {

}
```

Because this is a separate function to main we need to make the `db` variable global. Add this near the top of the file.

```go
var db *pgDb
```

And in the `main` function we need to change how we assign the `db` variable, because in go `:=` always sets a local variable and not the global variable. So find this line:

```go
db, err := initDb()
```

And change it to these lines:

```go
var err error
db, err = initDb()
```

We also need to add a function to the database struct to get all the events in the calendar. We cant just return the data so we return a row pointer.

```go
func (p *pgDb) getEvents() (*sql.Rows, error) {
  // Select all rows
  lookupSql := `
        SELECT uid, summary, dtstart, dtend FROM events
  `
  // Query rows
  r, err := p.dbConn.Query(lookupSql)
  if err != nil {
    return nil, err
  }
  // Return rows pointer
  return r, nil
}
```

Now in the `handleCalendar` function we can create a calendar and iterate over all the rows to be inserted.

```go
// Create calendar
c := ical.New()
// Get rows
r, err := db.getEvents()
if err != nil {
  // Send http error if we cant load events
  w.WriteHeader(http.InternalServerError)
  return
}
// Iterare over rows
for r.Next() {
  // Setup variables for each row
  var uid, summary string
  var start, end time.Time
  // Pull row data out
  err := rows.Scan(&uid, &summary, &start, &end)
  if err != nil {
    // Send error
    w.WriteHeader(http.InternalServerError)
    return
  }
}
if err := r.Err(); err != nil {
  // Send error
  w.WriteHeader(http.InternalServerError)
  return
}
```

In each iteration over the rows we can now create an event and insert all the data.

```go
// Create new event
e := ical.NewEvent()
// Convert times to text
var startTxt, endTxt string
startTxt = start.Format(iCalDateFormat)
endTxt = end.Format(iCalDateFormat)
// Add data
e.AddProperty("uid", uid)
e.AddProperty("summary", summary)
e.AddProperty("dtstart", startTxt)
e.AddProperty("dtend", endTxt)
// Ad event to calendar
c.AddEntry(e)
```

We can now encode the calendar and write it out to the client.

```go
ical.NewEncoder(w).Encode(c)
```

All of this together makes this handler function.

```go
func handleCalendar(w http.ResponseWriter, r *http.Request) {
  // Create calendar
  c := ical.New()
  // Get rows
  rows, err := db.getEvents()
  if err != nil {
    // Send http error if we cant load events
    w.WriteHeader(http.StatusInternalServerError)
    return
  }
  // Iterare over rows
  for rows.Next() {
    // Setup variables for each row
    var uid, summary string
    var start, end time.Time
    // Pull row data out
    err := rows.Scan(&uid, &summary, &start, &end)
    if err != nil {
      // Send error
      w.WriteHeader(http.StatusInternalServerError)
      return
    }
    // Create new event
    e := ical.NewEvent()
    // Convert times to text
    var startTxt, endTxt string
    startTxt = start.Format(iCalDateFormat)
    endTxt = end.Format(iCalDateFormat)
    // Add data
    e.AddProperty("uid", uid)
    e.AddProperty("summary", summary)
    e.AddProperty("dtstart", startTxt)
    e.AddProperty("dtend", endTxt)
    // Ad event to calendar
    c.AddEntry(e)
  }
  if err := rows.Err(); err != nil {
    // Send error
    w.WriteHeader(http.StatusInternalServerError)
    return
  }
  // Write out calendar
  ical.NewEncoder(w).Encode(c)
}
```

And when we run this the output I get in my browser is:

```
BEGIN:VCALENDAR
VERSION:2.0
PRODID:github.com/lestrrat/go-ical
BEGIN:VEVENT
DTEND:20190118
DTSTART:20170117
SUMMARY:Not available
UID:-4r3v77r036am--nqxyxezcrcob@airbnb.com
END:VEVENT
END:VCALENDAR
```

So it works! The website server can insert into this database and it will show up in the http server.

## Step 5: Cleanup

Now in it's current state while the program is functional it is definatley not complete. For example it fetches the Airbnb calender once at startup and then serves the http server. So let's fix some of those issues. First we'll split off the database functions into their own file `db.go`. The file looks like this:

```go
package main

import (
    "log"
	"time"
	"database/sql"
	_ "github.com/lib/pq"
	"github.com/lestrrat/go-ical"
)

type pgDb struct {
	dbConn *sql.DB
}

func initDb() (*pgDb, error) {
	if dbConn, err := sql.Open("postgres", "user=postgres password=PASSWORD host=127.0.0.1 dbname=airbnb_cal"); err != nil {
		return nil, err
	} else {
		p := &pgDb{dbConn: dbConn}
		if err := p.dbConn.Ping(); err != nil {
			return nil, err
		}
		return p, nil
	}
}

func (p *pgDb) createTablesIfNotExist() error {
	createSql := `
       CREATE TABLE IF NOT EXISTS events (
       uid TEXT NOT NULL PRIMARY KEY,
       summary TEXT NOT NULL,
       dtstart TIMESTAMP NOT NULL,
       dtend TIMESTAMP NOT NULL);
    `
	if rows, err := p.dbConn.Query(createSql); err != nil {
		return err
	} else {
		rows.Close()
	}
	return nil
}

func (p *pgDb) insertOrUpdateEvent(uid *ical.Property, start *ical.Property, end *ical.Property, summary *ical.Property) error {
	lookupSql := `
        SELECT uid, summary, dtstart, dtend FROM events
        WHERE dtstart = $1 AND dtend = $2
    `
	exists := true
	var oldSummary, oldUid string
	var oldStart, oldEnd time.Time
	newStart, err := time.Parse(iCalDateFormat, start.RawValue())
	if err != nil {
		return err
	}
	newEnd, err := time.Parse(iCalDateFormat, end.RawValue())
	if err != nil {
		return err
	}
	r := p.dbConn.QueryRow(lookupSql, newStart, newEnd)
	err = r.Scan(&oldUid, &oldSummary, &oldStart, &oldEnd)
	if err == sql.ErrNoRows {
		exists = false
	} else if err != nil {
		return err
	}
	if !exists {
		insertSql := `
			INSERT INTO events
			(uid, dtstart, dtend, summary)
			VALUES ($1, $2, $3, $4)
        `
		// Run the insert
		_, err := p.dbConn.Exec(insertSql, uid.RawValue(), newStart, newEnd, summary.RawValue())
		if err != nil {
			return err
		}
	} else {
		if newStart != oldStart || newEnd != oldEnd || summary.RawValue() != oldSummary {
			updateSql := `
				UPDATE events
				SET dtstart = $3, dtend = $4, summary = $5, uid = $2
				WHERE uid = $1
        	`
			// Run the update
			_, err := p.dbConn.Exec(updateSql, oldUid, uid.RawValue(), newStart, newEnd, summary.RawValue())
			if err != nil {
				return err
			}
		}
	}
	return nil
}

func (p *pgDb) getEvents() (*sql.Rows, error) {
	lookupSql := `
        SELECT uid, summary, dtstart, dtend FROM events
  `
	r, err := p.dbConn.Query(lookupSql)
	if err != nil {
		return nil, err
	}
	return r, nil
}
```

Then let's split off the serving of pages into `http.go`.

```go
package main

import (
	"time"
	"github.com/lestrrat/go-ical"
	"net/http"
	"github.com/gorilla/mux"
)

func handleCalendar(w http.ResponseWriter, r *http.Request) {
	c := ical.New()
	rows, err := db.getEvents()
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}
	for rows.Next() {
		var uid, summary string
		var start, end time.Time
		err := rows.Scan(&uid, &summary, &start, &end)
		if err != nil {
			w.WriteHeader(http.StatusInternalServerError)
			return
		}
		e := ical.NewEvent()
		var startTxt, endTxt string
		startTxt = start.Format(iCalDateFormat)
		endTxt = end.Format(iCalDateFormat)
		e.AddProperty("uid", uid)
		e.AddProperty("summary", summary)
		e.AddProperty("dtstart", startTxt)
		e.AddProperty("dtend", endTxt)
		c.AddEntry(e)
	}
	if err := rows.Err(); err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}
	ical.NewEncoder(w).Encode(c)
}

func serveHttp() error {
	router := mux.NewRouter().StrictSlash(true)
    router.Methods("GET").Path("/calendar/ical.ics").HandlerFunc(handleCalendar)
	return http.ListenAndServe(":8080", router)
}
```

And the getting of data goes into `load.go`

```go
package main

import (
	"net/http"
	"github.com/lestrrat/go-ical"
)

func updateEvents() error {
	response, err := http.Get("YOUR URL HERE")
	if err != nil {
		return err
	} else {
		defer response.Body.Close()
		p := ical.NewParser()
		c, err := p.Parse(response.Body)
		if err != nil {
			return err
		}

		for e := range c.Entries() {
			ev, ok := e.(*ical.Event)
			if !ok {
				continue
			}
			start, _ := ev.GetProperty("dtstart")
			end, _ := ev.GetProperty("dtend")
			uid, _ := ev.GetProperty("uid")
			summary, _ := ev.GetProperty("summary")
			err := db.insertOrUpdateEvent(uid, start, end, summary)
			if err != nil {
				return err
			}
		}
	}
	return nil
}
```

We now have everything nicely split up so we can start making this run properly. To get the airbnb calendar every now and then we can run a goroutine that waits for ticker events every 10 minutes or so. *But what is a goroutine?* I hear you cry. A goroutine is go's version of threading, but unlike some languages (*Cough* Python *Cough*) they can run at the same time. So this will just hang arround for a while until the ticker calls and then it will update the database.

First we define the ticker with an interval. This one will trigger every 5 seconds but of course this can and will be changed.

```go
updateTicker := time.NewTicker(time.Second * 5)
```

We can then define an anonymous function that runs in a goroutine to act on the ticker. The log will set us see whats happening.

```go
go func() {
  for t := range updateTicker.C {
    fmt.Printf("%v: Updating events", t)
    updateEvents()
  }
}()
```

We can then call the `serveHttp` which will run in the main thread. We'll also add some logging to show us whats going on. The `main.go` file now looks like this:

```go
package main

import (
	"log"
	"time"
)

const iCalDateFormat = "20060102"

var db *pgDb

func main() {
	log.Println("Connecting to database")
	var err error
	db, err = initDb()
	if err != nil {
		log.Fatalf("Error initializing database: %v\n", err)
	}

	err = db.createTablesIfNotExist()
	if err != nil {
		log.Fatalf("Error creating database tables: %v\n", err)
	}

	log.Println("Starting event update goroutine")
	updateTicker := time.NewTicker(time.Second * 5)
	go func() {
		for range updateTicker.C {
			log.Println("Updating events")
			updateEvents()
		}
	}()

	log.Println("Starting http server")
	serveHttp()
}
```

This gives us an output similar to:

```
2018/01/17 09:37:38 Connecting to database
2018/01/17 09:37:38 Setting up tables
2018/01/17 09:37:38 Starting event update goroutine
2018/01/17 09:37:38 Starting http server
2018/01/17 09:37:43 Updating events
2018/01/17 09:37:48 Updating events
2018/01/17 09:37:53 Updating events
```

Yay! It all works.

## Step 6: Take config from environment variables

So this all works fine now but there's one problem. We have to update the file and recompile to change config. The solution when being deployed in Docker (what I'll be doing) is to get these from environment variables. We can use `os.Getenv` for this purpose. It returns an empty string if the variable is not set. So the config variables we need are:

- Database host
- Database user
- Database password
- Database name
- Update interval (in seconds)

So let's take those in. We'll set some defaults as well.

```go
var dbHost, dbUser, dbPass, dbName, updateInterval string
var updateIntervalInt int
// Get variable
dbHost = os.Getenv("DB_HOST")
// Check if not set
if dbHost == "" {
  // Set default
  dbHost = "127.0.0.1"
}
dbUser = os.Getenv("DB_USER")
if dbUser = "" {
  dbUser = "postgres"
}
dbPass = os.Getenv("DB_PASS")
// Default is blank so we dont do anything here for dbPass
dbName = os.Getenv("DB_NAME")
if dbName = "" {
  dbName = "airbnb_cal"
}
updateInterval = os.Getenv("UPDATE_INTERVAL")
if updateInterval = "" {
  // 600 seconds = 10 minutes
  updateInterval = "600"
}
// Try to convert to int
if updateIntervalFloat, err := strconv.ParseFloat(updateInterval, 64); err != nil {
  // Need an integer
  log.Fatalf("Need an integer update interval. %v is not valid", updateInterval)
} else {
  // Remove decimal points
  updateIntervalInt = int(updateIntervalFloat)
}
```

We can now use these. Let's update the ticker first. It becomes this line.

```go
updateTicker := time.NewTicker(time.Second * time.Duration(updateIntervalInt))
```

We now  need to use the database values. Lets first define a database config struct. This goes in `db.go`

```go
type dbConfig struct {
	dbHost string
	dbName string
	dbUser string
	dbPass string
}
```

We can then accept a config in `initDb`. Our function definition becomes:

```go
func initDb(config *dbConfig) (*pgDb, error) {
```

We can now use said config in connecting to the database. We create a config string and pass it to the `Open` function.

```go
connectString := fmt.Sprintf("user=%s password=%s host=%s dbname=%s",
		config.dbUser, config.dbPass, config.dbHost, config.dbName)
if dbConn, err := sql.Open("postgres", connectString); err != nil {
```

Finally we create a config struct in `main.go` and pass it to the `dbInit` function.

```go
db, err = initDb(&dbConfig{
  dbHost: dbHost,
  dbUser: dbUser,
  dbPass: dbPass,
  dbName: dbName,
})
```

## Step 7: Dockerize!

Now thats all that we have to do. To finish let's put this is Docker as I said we would earlier Because this is a go file and we don’t want a 500MB image with all possible standard libraries we’ll have to statically link the executable. This is done by setting the `CGO_ENABLED` variable to 0. So our compile command would be (in the project root).

```
CGO_ENABLED=0 go build
```

This will take a while as it has to compile all the libraries we used. When it’s done we should have one `airbnb_calendar` file that can run anywhere. So now we need a docker container to run it all in. Our `Dockerfile` shall be:

```
FROM scratch

COPY airbnb_canelndar /

CMD ["/airbnb_canelndar"]
```

The `FROM scratch` starts us off with a blank container. We don’t need any libraries so this is perfect and makes the smallest of images. This can now be deployed to kubernetes etc, but that is beyond the scope of this post. You can use my [starter deployment template](https://misell.cymru/posts/starter-kubes-deploy/) for kubernetes if you want a starting point. 

Thanks for your time in reading this and I hope you learn't something!