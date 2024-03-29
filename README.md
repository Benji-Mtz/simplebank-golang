# Notas de Backend Master Class [Golang + PostgreSQL + Kubernetes + gRPC]

## Descargando postgres

```sh
docker pull <image>:<tag>
docker pull postgres:12-alpine

# Un Container es una instancia de una imagen
```
## Instalando postgres en un contenedor

```sh
docker run --name <container_name> -p <host_ports:container_ports> -e <enviroment_variable> -d <image>:<tag>

docker run --name postgres12 -p 5432:5432 -e POSTGRES_USER=root -e POSTGRES_PASSWORD=secret -d postgres:12-alpine

docker ps
CONTAINER ID   IMAGE                COMMAND                  CREATED          STATUS          PORTS                                       NAMES
ae924fee9427   postgres:12-alpine   "docker-entrypoint.s…"   24 seconds ago   Up 23 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   postgres12

docker images
REPOSITORY   TAG         IMAGE ID       CREATED       SIZE
postgres     12-alpine   1ace9f47704c   3 weeks ago   211MB
```
## Correr comandos dentro de un contenedor
```sh
docker exec -it <container_name_or_id> <command> [args]
docker exec -it postgres12 psql -U root

root=# select now(); // 2022-09-07 05:09:50.83033+00
root=# \q // exit
```

## Ver logs del container
```sh
docker logs <container_name_or_id>
docker logs postgres12
```

## Configuración dbeaver
```
Host=localhost
Database=root //Igual que el username de default
Usuario=root
Contraseña=secret
```

## Migraciones con postgres y golang

```sh
go get -u -d github.com/golang-migrate/migrate/cmd/migrate

$ (root_directory) migrate create -extension sql -directorio db/migration -seq_num_migracion_autoincrement schema_name
$ (root_directory) migrate create -ext sql -dir db/migration -seq init_schema
```
```
(000001_init_schema.down.sql)
	DROP TABLE IF EXISTS entries;
	DROP TABLE IF EXISTS transfers;
	DROP TABLE IF EXISTS accounts;

(000001_init_schema.up.sql)
	CREATE TABLE "accounts" (
	  "id" SERIAL PRIMARY KEY,
	  "owner" varchar NOT NULL,
	  "balance" bigint NOT NULL,
	  "currency" varchar NOT NULL,
	  "created_at" timestamptz NOT NULL DEFAULT (now())
	);

	CREATE TABLE "entries" (
	  "id" bigserial PRIMARY KEY,
	  "account_id" bigint NOT NULL,
	  "amount" bigint NOT NULL,
	  "created_at" timestamptz NOT NULL DEFAULT (now())
	);

	CREATE TABLE "transfers" (
	  "id" bigserial PRIMARY KEY,
	  "from_account_id" bigint NOT NULL,
	  "to_account_id" bigint NOT NULL,
	  "amount" bigint NOT NULL,
	  "created_at" timestamptz NOT NULL DEFAULT (now())
	);

	CREATE INDEX ON "accounts" ("owner");
	CREATE INDEX ON "entries" ("account_id");
	CREATE INDEX ON "transfers" ("from_account_id");
	CREATE INDEX ON "transfers" ("to_account_id");
	CREATE INDEX ON "transfers" ("from_account_id", "to_account_id");
	COMMENT ON COLUMN "entries"."amount" IS 'can be negative or positive';
	COMMENT ON COLUMN "transfers"."amount" IS 'must be positive';
	ALTER TABLE "entries" ADD FOREIGN KEY ("account_id") REFERENCES "accounts" ("id");
	ALTER TABLE "transfers" ADD FOREIGN KEY ("from_account_id") REFERENCES "accounts" ("id");
	ALTER TABLE "transfers" ADD FOREIGN KEY ("to_account_id") REFERENCES "accounts" ("id");

# Esto es porque apartir de las migraciones con el comando UP se pueden regererar las bases de datos y con el comando DOWN se pueden reiniciar secuencialmente.
# O de otra forma se ejecutan secuencialmente los archivos up.sql o down.sql respectivamente.
```
## Alta y baja de contenedores

```sh
docker stop postgres12
docker ps //No sale
docker ps -a //Sale apagado
docker start postgres12 //Lo pronde de nuevo
```

## Verificando las migraciones

```sh
docker exec -it postgres12 /bin/sh

# pwd // /
# ls -l // directorios

docker exec -it postgres12 createdb --username=root --owner=root simple_bank
docker exec -it postgres12 psql -U root simple_bank

simple_bank=# \q

# Creamos un Makefile
make dropdb
make createdb

history | grep "docker run"

docker stop postgres12
docker ps -a
docker rm postgres12
docker ps -a

make postgres
docker ps
make createdb

migrate -help
migrate -path db/migration -database "postgresql://root:secret@localhost:5432/simple_bank?sslmode=disable" -verbose up


```
