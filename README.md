# Library API - IN3101 Lab 03

Student index: **234033P**. This is the Lab 02 Spring Boot Book API packaged with Docker and MySQL for Lab 03.

## Start the system

Prerequisites: Docker Desktop running with Linux containers, Java 21 or newer, and internet access for the first build.

From this project directory in PowerShell:

```powershell
# On a fresh clone, create the local password file first:
Copy-Item .env.example .env
# Edit .env and set DB_ROOT_PASSWORD before starting.
.\mvnw.cmd clean package -DskipTests
docker compose up --build -d
docker compose ps
docker compose logs api
```

The local `.env` prepared for this lab uses the lab sheet's password, `root123`. It is ignored by Git. Do not overwrite an existing `.env` when repeating the commands. The JAR must be built before Docker builds the image; this follows the single-stage Dockerfile required by the sheet. `-DskipTests` skips test execution because the existing Spring context test requires a running database.

Open:

- API: http://localhost:8080/api/books
- Swagger UI: http://localhost:8080/swagger-ui.html
- OpenAPI JSON: http://localhost:8080/v3/api-docs
- MySQL Workbench: host `127.0.0.1`, port `3307`, user `root`, password from `.env`, database `library_db`.

## What each file does

| File | Purpose |
| --- | --- |
| `Dockerfile` | Starts with `eclipse-temurin:21-jre`, selects `/app`, copies the built JAR as `app.jar`, documents port 8080, and runs `java -jar app.jar`. Maven and the JDK are not needed inside the runtime image. |
| `.dockerignore` | Excludes source, Git metadata, editor files, documentation, local tools, and `.env` from the Docker build context. Keeps `target/*.jar` available for copying. |
| `compose.yaml` | Defines the `db` and `api` services, their ports, database health check, environment variables, network dependency, and persistent volume. |
| `.env` | Holds the local database password. This file is not committed. |
| `.env.example` | Gives other students the required variable name without committing the local password. |
| `src/main/resources/application.properties` | Provides shareable application settings. Local defaults target `localhost:3306`; Compose overrides the datasource settings using Spring environment variables. |
| `docs/Library-API-Lab03.postman_collection.json` | Importable Postman requests for CRUD, validation, and your three submission books. |
| `scripts/verify-lab.py` | Runs the lab's Docker and API checks and writes actual output to `docs/evidence/`. Requires Python 3 and the built JAR. |

The original `application.properties` was excluded from Git and contained a laptop-specific password. It is now included with an empty default password. For running directly on your laptop, set `DB_PASSWORD` to your local MySQL password; Compose supplies `SPRING_DATASOURCE_PASSWORD` automatically. No Java or dependency changes were needed.

## Networking and startup explained

Inside the API container, `localhost` refers to that API container. It cannot reach a database on your laptop or in another container through its own localhost address. This is why Part C deliberately fails when the image is run alone.

Compose creates a network where the service name `db` resolves to the database container. The API uses `jdbc:mysql://db:3306/library_db`. Spring Boot maps `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, and `SPRING_DATASOURCE_PASSWORD` to the corresponding application properties, overriding the file settings.

Laptop clients use `localhost:3307`, which Docker forwards to MySQL's container port 3306. Container-to-container traffic uses `db:3306` directly. The API mapping is `8080:8080`. `EXPOSE 8080` documents the image's port; the Compose `ports` entry publishes it on your laptop.

The database health check probes MySQL over TCP every five seconds. `depends_on: condition: service_healthy` delays API startup until that check passes. `$$MYSQL_ROOT_PASSWORD` defers variable expansion to the database container. See [Docker's startup-order documentation](https://docs.docker.com/compose/how-tos/startup-order/).

## Persistence explained

The named volume `library-data` mounts at `/var/lib/mysql`, where MySQL stores database files. Compose normally calls the physical volume `library-api_library-data`. The standalone Part B exercise uses a separate physical volume called `library-data`.

```powershell
docker compose down
docker volume ls
docker compose up -d
```

`down` removes the containers and network while retaining the database volume. Newly created containers therefore recover the same books and IDs. `docker compose down -v` deletes the Compose volume and its data; it is not part of the persistence verification. See [Docker's volume documentation](https://docs.docker.com/engine/storage/volumes/).

## Demonstrate the API in Postman

Import `docs/Library-API-Lab03.postman_collection.json`. Its `baseUrl` is `http://localhost:8080`. Run the requests in collection order; the create response saves `bookId` for later requests. The collection retains the three required books and uses a separate temporary book for update/delete testing.

The three retained titles are `234033P - Clean Code`, `Effective Java`, and `Design Patterns`. Creating them again will create additional rows; the API does not enforce ISBN uniqueness.

In MySQL Workbench, create a connection using port 3307 and run:

```sql
SELECT * FROM library_db.book;
```

For the restart demonstration, send GET `/api/books`, note the books and IDs, run `docker compose down` followed by `docker compose up -d`, wait for startup, and send GET again. The results should match.

## Automated verification and evidence

```powershell
python scripts/verify-lab.py
```

Run this only with ports 8080 and 3307 available, no existing `library-mysql` or `api-test` container, and the Compose stack stopped. The script creates and removes its standalone test containers, recreates this project's Compose containers without deleting volumes, and leaves the final stack running. It adds three sample books on each run and retains the Part B volume.

The checks cover hello-world, Part B volume survival across container replacement, image build, the expected standalone connection failure, Compose readiness, CRUD HTTP statuses, invalid input, Swagger, direct SQL rows, and identical data/IDs after Compose recreation. Output is saved under `docs/evidence/`; these machine-generated files are ignored by Git. See `docs/LAB03-RESULTS.md` for the recorded outcome.

## Submission

The lab asks for the same GitHub repository used for Lab 02: https://github.com/Bbuddhika/EAD-Lab02.

Review and commit `Dockerfile`, `.dockerignore`, `compose.yaml`, `.env.example`, the updated `.gitignore`, shareable application settings, and documentation/scripts. Verify `.env` is ignored with `git check-ignore .env`. Push the commit to your existing repository and paste that repository link into the Lab 03 LMS submission box. The PDF specifies **11 November 2026** as the deadline. GitHub publication and LMS submission are separate from the local implementation.
