---
layout: post
title:  "Go Testing + Docker"
date:   2015-09-14
categories: go tests testing docker
---

Databases, authentication management, metric aggregators; there's a service or API for everything.

There's also a place called dependency hell.

Dependency hell. Where it's impossible to run a test on your laptop. Where bugs are found by running a main, clicking around, and seeing what breaks. Where you cross your fingers and pray that new versions don't break everything. Where we use mocks and stubs to combat a growing technology stack. 

But why mock when you can have the real thing?

This post is about using Docker in test infrastructure. To run the code you'll need:

1. Go
2. Docker running on your host machine (not boot2docker - sorry OS X and Windows users)

## An example using Postgres 

I work at a company that has a Go project that uses Postgres. How do you test database code? Maybe you can convince all of your employees to install Postgres with the exact same configuration. Maybe you could use SQLite to reduce dev setup times (never do this). 

Of course with the awesomeness of Linux containers you just need to use Docker!

The workflow we're looking for: for each test, start a fresh instance of Postgres in a Docker container, then run the tests using it.

{% highlight go %}
dockerContainer, err := NewPostgresDB()
if err != nil {
	// handle error
}
defer destroy(dockerContainer)

// run test
{% endhighlight %}

For our code, the Postgres struct will look like this. There's only two bits of information that it'll hold, the port the container is running on and the ID of the container.

{% highlight go %}
type PostgresDB struct {
	// Of form 'host:port'
	Host string

	// The container id assigned by docker
	cid string
}
{% endhighlight %}


## Postgres' Docker images

Postgres has an [official image](https://hub.docker.com/_/postgres/) through Docker Hub. Most databases do.

Starting a container is easy using Docker and will look something like the following.

<pre><code>docker run -d -p $HOSTPORT:5432 -e POSTGRES_PASSWORD=$PASS postgres:$VERSION
</code></pre>

The `-p` flag requests an exposed port and, because Postgres runs on port `5432`, we need to map `5432` to the host machine. `-d` indicates "detached mode," and `-e` is used to specify environment variables.

The actual image allows for a bit of configuration through the env. This will be another struct.

{% highlight go %}
type PostgresConfig struct {
	Password string
	Username string // defaults to "postgres"
	Database string // defaults to "username"
	Version  string // defaults to "latest"
}
{% endhighlight %}

To create an instance of Postgres, we'll determine Docker's command line arguments given a config. Then run the Docker command through Go's `os/exec` library. 

{% highlight go %}
func NewPostgresDB(c PostgresConfig) (*PostgresDB, error) {
	img := "postgres:latest"
	if c.Version == "" {
		img = "postgres:" + c.Version
	}

	// docker's run command has the nasty habbit of pulling images if you don't have them.
	// Warn user they need to pull the image, don't automatically pull for them.
	if exec.Command("docker", "inspect", img).Run() != nil {
		return nil, fmt.Errorf("db requires docker image %s, please pull or specify a different version", img)
	}

	// Running on port 0 instructs the operating system to pick an available port.
	dockerArgs := []string{"run", "-d", "-p", "127.0.0.1:0:5432"}
	envvars := map[string]string{
		"POSTGRES_PASSWORD": c.Password,
		"POSTGRES_USER":     c.Username,
		"POSTGRES_DB":       c.Database,
	}
	for key, val := range envvars {
		if val != "" {
			dockerArgs = append(dockerArgs, "-e", key+"="+val)
		}
	}
	dockerArgs = append(dockerArgs, img)

	// Start the docker container.
	out, err := exec.Command("docker", dockerArgs...).CombinedOutput()
	if err != nil {
		return nil, fmt.Errorf("docker run: %v: %s", err, out)
	}
	cid := strings.TrimSpace(string(out))
	db := &PostgresDB{cid: cid}

	db.Host, err = portMapping(cid, "5432/tcp")
	if err != nil {
		db.Close()
		return nil, err
	}
	return db, nil
}
{% endhighlight %}

The host address specified above is `127.0.0.1:0`. Port 0 is a way of telling the operating system to pick any available port. For testing it's nice to not hard code an explicit port in case the person running the tests has a conflicting service (like an actually Postgres database).

`docker inspect` allows us figure out what port the operating system chose. This returns a giant JSON object with, among other things, network settings for the running container.

{% highlight go %}
func portMapping(cid, containerPort string) (hostAddr string, err error) {
	out, err := exec.Command("docker", "inspect", cid).CombinedOutput()
	if err != nil {
		return "", fmt.Errorf("docker inspect: %v: %s", err, out)
	}

	// anonymous struct for unmarshalling JSON into
	var inspectResp []struct {
		NetworkSettings struct {
			Ports map[string][]struct {
				HostIp   string
				HostPort string
			}
		}
	}
	if err := json.Unmarshal(out, &inspectResp); err != nil {
		return "", fmt.Errorf("decoding docker inspect result failed: %v: %s", err, out)
	}
	if len(inspectResp) != 1 {
		return "", fmt.Errorf("expected one inspect result, got %d", len(inspectResp))
	}
	ports := inspectResp[0].NetworkSettings.Ports[containerPort]
	if len(ports) != 1 {
		return "", fmt.Errorf("expected one port mapping, got %d", len(ports))
	}
	return ports[0].HostIp + ":" + ports[0].HostPort, nil
}
{% endhighlight %}

Finally, closing the database should result in the entire container being removed. Again, we'll just use `os/exec` to call Docker.

{% highlight go %}
// Close removes the container running the postgres database.
func (db *PostgresDB) Close() error {
	out, err := exec.Command("docker", "rm", "-f", db.cid).CombinedOutput()
	if err != nil {
		return fmt.Errorf("docker rm: %v: %s", err, out)
	}
	return nil
}
{% endhighlight %}

## Writing a test helper

Go tests always have a signature like this:

{% highlight go %}
func TestXyz(t *testing.T)
{% endhighlight %}

We want to write tests that take a database connection as well. Something that looks like this:

{% highlight go %}
type DBTest func(t *testing.T, conn *sql.DB)
{% endhighlight %}

So let's write a function to run a `DBTest`. This steps will be

1. Create a container
2. Connect to the container
3. Run a test with the connection 
4. Clean up the container

Importing the [`github.com/lib/pq`](https://github.com/lib/pq) driver is enough to connect to the container. While the code we used before will create and clean up a new instance of Postgres for each test.

{% highlight go %}
import (
	// other imports

	_ "github.com/lib/pq" // "empty import" to register driver with database/sql
)

type DBTest func(t *testing.T, conn *sql.DB)

func RunDBTest(t *testing.T, dbVersion string, test DBTest) {
	c := PostgresConfig{"pass", "eric", "postgrestest", dbVersion}

	// create a postgres container
	db, err := NewPostgresDB(c)
	if err != nil {
		t.Fatal(err)
	}
	defer db.Close() // destroy the postgres container after the test

	// create a connection URL
	// http://www.postgresql.org/docs/current/static/libpq-connect.html#LIBPQ-CONNSTRING
	connURL := &url.URL{
		Scheme:   "postgres",
		User:     url.UserPassword(c.Username, c.Password),
		Host:     db.Host,
		Path:     "/" + c.Database,
		RawQuery: "sslmode=disable",
	}

	// connect to database
	conn, err := sql.Open("postgres", connURL.String())
	if err != nil {
		t.Error(err)
		return
	}
	defer conn.Close()

	// ping the database until it comes up
	timeout := time.Now().Add(time.Second * 20)
	for time.Now().Before(timeout) {
		if err = conn.Ping(); err == nil {
			// yay! we've connected to the database, time to run the test
			test(t, conn)
			return
		}
		time.Sleep(100 * time.Millisecond)
	}
	t.Errorf("failed to connect to database: %v", err)
}
{% endhighlight %}

## Time to write the tests

Finally we can just write tests that adhere to `DBTest`'s function signature and run them with `RunDBTest`. Let's test creating a table and inserting a record.

{% highlight go %}
package dbtest

import (
	"database/sql"
	"testing"

	_ "github.com/lib/pq"
)

func createTable(conn *sql.DB) error {
	statement := `
		CREATE TABLE Worker (
			WorkerId SERIAL PRIMARY KEY,
			Host     TEXT NOT NULL,
			UsingTLS BOOLEAN NOT NULL
		);`
	_, err := conn.Exec(statement)
	return err
}

// Test creating a table
func testCreateTable(t *testing.T, conn *sql.DB) {
	if err := createTable(conn); err != nil {
		t.Error(err)
	}
}

// Test creating a table then inserting a row
func testInsertWorker(t *testing.T, conn *sql.DB) {
	if err := createTable(conn); err != nil {
		t.Error(err)
		return
	}
	insertWorker := `INSERT INTO Worker (Host, UsingTLS) VALUES ($1, $2);`
	if _, err := conn.Exec(insertWorker, "10.0.0.3", true); err != nil {
		t.Error(err)
	}
}

// Test multiple versions of Postgres
const (
	version94 = "9.4"
	version95 = "9.5"
)

// The actual Go tests just immediately call RunDBTest
func TestCreateTable94(t *testing.T)  { RunDBTest(t, version94, testCreateTable) }
func TestCreateTable95(t *testing.T)  { RunDBTest(t, version95, testCreateTable) }
func TestInsertWorker94(t *testing.T) { RunDBTest(t, version94, testInsertWorker) }
func TestInsertWorker95(t *testing.T) { RunDBTest(t, version95, testInsertWorker) }
{% endhighlight %}

This setup enables a bunch of cool things like tests against different versions of Postgres. Once we (or importantly, one of our coworkers) have pulled the correct Postgres images, we just have to run `go test`. The only dependencies are Go and Docker.

<pre><code>$ go test -v
=== RUN   TestCreateTable94
--- PASS: TestCreateTable94 (6.21s)
=== RUN   TestCreateTable95
--- PASS: TestCreateTable95 (6.09s)
=== RUN   TestInsertWorker94
--- PASS: TestInsertWorker94 (5.97s)
=== RUN   TestInsertWorker95
--- PASS: TestInsertWorker95 (5.83s)
PASS
ok  	_/p/go/dbtest	24.106s
</code></pre>

It's not blindingly fast, but most of the overhead is from Postgres' [initialization](https://github.com/docker-library/postgres/blob/87b8be7e9b324ff2bcd6545d05895fac8f012dac/9.5/docker-entrypoint.sh), not Docker. If you know your configuration ahead of time, it's easy to make a custom Docker image and speed things up.

Importantly, we're testing against full installations of Postgres 9.4 and 9.5. No mocks, no sqlite.

## Things other than databases

This strategy is absolutely _not_ limited to databases, and gets even cooler when you expand past just hitting an exposed port.

With a little bit of imagination you can use Docker to test your Nginx config files, your RabbitMQ client, your WordPress plugins. For each test you'll get a fresh installation of the thing your testing against, and all a coworker needs to do is download a Docker image.

So mount your files, use [`--net=host`](https://github.com/bradfitz/http2/blob/f8202bc903bda493ebba4aa54922d78430c2c42f/http2_test.go#L96-L102), and who knows, you might even have a little fun writing tests.
