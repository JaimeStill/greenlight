# GreenLight

A RESTful API for retrieving and managing information about movies. Companion API for the book [Let's Go Further](https://lets-go-further.alexedwards.net/) by [Alex Edwards](https://github.com/alexedwards).

## Quickstart

Assuming [Initialization](#initialization) has been completed:

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

Exit MySQL and test the infrastructure:

```sh
docker exec -it greenlight-postgres psql -U greenlight -d greenlight
```

```sql
SELECT current_user;
```

```
greenlight=> SELECT current_user;
 current_user
--------------
 greenlight
(1 row)
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
