# GreenLight

A RESTful API for retrieving and managing information about movies. Companion API for the book [Let's Go Further](https://lets-go-further.alexedwards.net/) by [Alex Edwards](https://github.com/alexedwards).

## Quickstart

Assuming [Setup](#setup) and [Initialization](#initialization) have been completed:

1. Run the API server:

  ```sh
  go run ./cmd/api
  ```

2. Connect to PostgreSQL:

  ```sh
  docker exec -it greenlight-postgres psql -U greenlight -d greenlight
  ```

Navigate to http://localhost:5000

Test healthcheck:

```sh
curl -i http://localhost:5000/v1/healthcheck
```

## PostgreSQL Integration

Instead of directly installing PostgreSQL, I've opted to use a Docker container for the PostgreSQL database. This approach simplifies the setup process and ensures consistency across different development environments.

> For comprehensive details on working with the PostgreSQL container and PostgreSQL in general, reference [PostgreSQL Container Documentation](./_docs/postgres-container.md).

The [container configuration](./docker-compose.yml) is setup to persist data in a volume in a `.data` directory at the repository root. It also exposes read-only access to the [`sql`](./sql) directory on `/opt/sql` so that the SQL infrastructure can be initialized directly form the container.

### Setup

Make sure the following env var is setup (i.e. - `~/.bash_profile`):

```sh
export GREENLIGHT_DB_DSN="postgres://greenlight:pa55word@localhost/greenlight?sslmode=disable"
```

Setup the [`migrate`](https://github.com/golang-migrate/migrate) library in `~/go/bin`:

```sh
# setup tmp directory
mkdir -p /tmp/migrate
cd /tmp/migrate

# download the binary package
curl -L https://github.com/golang-migrate/migrate/releases/download/[version]/migrate.linux-amd64.tar.gz | tar xvz

# move to ~/go/bin
mv migrate ~/go/bin

# return to project
cd [path-to]/greenlight

# cleanup
rm -rf /tmp/migrate
```

### Initialization

Start the container in detached mode:

```sh
docker compose up -d
```

Verify the container is running:

```sh
docker ps
```

Connect via Docker Exec:

```sh
docker exec -it greenlight-postgres psql -U postgres
```

Initialize SQL infrastructure:

```sh
\i /opt/sql/db-init.sql
```

Exit MySQL and apply the database migrations:

```sh
migrate -path=./migrations -database=$GREENLIGHT_DB_DSN up
```

Test the infrastructure:

```sh
docker exec -it greenlight-postgres psql -U greenlight -d greenlight
```

```
greenlight=> SELECT current_user;
 current_user
--------------
 greenlight
(1 row)

greenlight=> \dt
                List of relations
 Schema |       Name        | Type  |   Owner
--------+-------------------+-------+------------
 public | movies            | table | greenlight
 public | schema_migrations | table | greenlight
(2 rows)

greenlight=> SELECT * FROM schema_migrations;
 version | dirty
---------+-------
       2 | f
(1 row)

greenlight=> \d movies
                                        Table "public.movies"
   Column   |            Type             | Collation | Nullable |              Default
------------+-----------------------------+-----------+----------+------------------------------------
 id         | bigint                      |           | not null | nextval('movies_id_seq'::regclass)
 created_at | timestamp(0) with time zone |           | not null | now()
 title      | text                        |           | not null |
 year       | integer                     |           | not null |
 runtime    | integer                     |           | not null |
 genres     | text[]                      |           | not null |
 version    | integer                     |           | not null | 1
Indexes:
    "movies_pkey" PRIMARY KEY, btree (id)
Check constraints:
    "genres_length_check" CHECK (array_length(genres, 1) >= 1 AND array_length(genres, 1) <= 5)
    "movies_runtime_check" CHECK (runtime >= 0)
    "movies_year_check" CHECK (year >= 1888 AND year::double precision <= date_part('year'::text, now()))
```

### Migrations

**Generate a Migration**

```sh
migrate create -seq -ext=.sql -dir=./migrations [migration-name]
```

You can then modify the SQL files directly on your machine to configure the migrations.

**Execute Migrations**

```sh
migrate -path=./migrations -database=$GREENLIGHT_DB_DSN up
```

**Check Current Migration Version**

```sh
migrate -path=./migrations -database=$GREENLIGHT_DB_DSN version
2
```

**Migrate to Specific Version**

```sh
migrate -path=./migrations -database=$GREENLIGHT_DB_DSN goto 1
```

**Migrate Down**

> [!NOTE]
> Specifying `down` without providing a version will roll back ALL migrations.

```sh
migrate -path=./migrations -database=$GREENLIGHT_DB_DSN down 1
```

### Shutdown

Stop the container:

```sh
docker compose stop
```

Stop and remove the container:

```sh
docker compose down
```

Stop and remove the container and delete the volume:

```sh
docker compose down -v
sudo rm -rf .data/
```

## References

Helpful links discovered throughout this book.

> [!NOTE]
> This is a continuation of references already captured in the [snippetbox repo](https://github.com/JaimeStill/snippetbox?tab=readme-ov-file#references).

- [httprouter](https://pkg.go.dev/github.com/julienschmidt/httprouter)
- [Go Well-known Struct Tags](https://go.dev/wiki/Well-known-struct-tags)
- [json:api](https://jsonapi.org/)
- [jsend](https://github.com/omniti-labs/jsend)
- [Go Method Pointer vs. Value Receivers](https://medium.com/globant/go-method-receiver-pointer-vs-value-ffc5ab7acdb)
- [PostgreSQL](https://www.postgresql.org/)
- [PostgreSQL Extensions](https://www.postgresql.org/docs/current/contrib.html)
- [Tune PostgreSQL for Memory](https://www.enterprisedb.com/postgres-tutorials/how-tune-postgresql-memory)
- [pq Go PostgreSQL Driver](https://github.com/lib/pq)
- [golang-migrate](https://github.com/golang-migrate/migrate)
- [PostgreSQL Data Types](https://www.postgresql.org/docs/current/datatype.html)
- [Decoupling Database Migrations from Server Startup](https://pythonspeed.com/articles/schema-migrations-server-startup/)
