# Docker Project: Deploy a Flask API with PostgreSQL

## 1. Objective

In this practical, we will deploy a Python Flask API and a PostgreSQL database using Docker containers.

We will learn how to:

- Create a custom Docker network.
- Create a persistent Docker volume.
- Run PostgreSQL in a container.
- Build a Python backend Docker image.
- Connect Flask to PostgreSQL using environment variables.
- Create database tables.
- Test API endpoints.
- Troubleshoot common Docker and database errors.

## 2. Prerequisites

Install the following tools:

- Docker Desktop
- Python application source code
- PostgreSQL-compatible Flask application
- PowerShell on Windows

Verify Docker is working:

```powershell
docker --version
docker info
```

Check whether Docker is running before proceeding.

## 3. Project Directory Structure

Create the following project structure:

```text
C:\Docker-Project\
│
├── backend\
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
│
└── .env
```

The `backend` directory contains the Flask application and its Dockerfile. The `.env` file stores database connection settings.

## 4. Create a Docker Network

A Docker network allows the backend container to communicate with PostgreSQL using its container name.

```powershell
docker network create mynetwork
```

Verify the network:

```powershell
docker network ls
docker network inspect mynetwork
```

Both the PostgreSQL and Flask containers will use this network.

> Run the network creation command only once. If `mynetwork` already exists, reuse it.

## 5. Create a Persistent Docker Volume

A Docker volume stores PostgreSQL data independently of the container lifecycle.

```powershell
docker volume create pgdata
```

Verify the volume:

```powershell
docker volume ls
docker volume inspect pgdata
```

We will mount this volume at `/var/lib/postgresql/data` inside the PostgreSQL container.

This allows database data to survive container removal and recreation.

## 6. Start the PostgreSQL Container

Run the following command in PowerShell:

```powershell
docker run -d `
  --name my_postgres `
  --network mynetwork `
  -e POSTGRES_USER=admin `
  -e POSTGRES_PASSWORD=123456 `
  -e POSTGRES_DB=mydatabase `
  -v pgdata:/var/lib/postgresql/data `
  -p 5432:5432 `
  --restart unless-stopped `
  postgres:13
```

### Explanation of the command

| Option | Purpose |
|---|---|
| `-d` | Runs the container in the background |
| `--name my_postgres` | Assigns a container name |
| `--network mynetwork` | Connects the container to the Docker network |
| `POSTGRES_USER` | Sets the initial database user |
| `POSTGRES_PASSWORD` | Sets the initial database password |
| `POSTGRES_DB` | Creates the initial database |
| `-v pgdata:/var/lib/postgresql/data` | Persists PostgreSQL data |
| `-p 5432:5432` | Publishes PostgreSQL's port |
| `--restart unless-stopped` | Restarts the container automatically unless stopped |
| `postgres:13` | Specifies the PostgreSQL image |

The example password is for a local learning environment only. Use a strong password or a secret-management solution for real deployments.

### Verify PostgreSQL

```powershell
docker ps
docker logs my_postgres
```

Check whether PostgreSQL is accepting connections:

```powershell
docker exec my_postgres pg_isready -U admin -d mydatabase
```

Expected output:

```text
/var/run/postgresql:5432 - accepting connections
```

## 7. Configure the Flask Backend

Create a `.env` file in `C:\Docker-Project`:

```dotenv
DB_HOST=my_postgres
DB_PORT=5432
DB_NAME=mydatabase
DB_USERNAME=admin
DB_PASSWORD=123456
DB_SSLMODE=disable

CORS_ORIGINS=http://localhost:3000
```

These values must match the PostgreSQL container configuration.

### Important database connection detail

The Flask application must use:

```text
Host: my_postgres
Port: 5432
Database: mydatabase
Username: admin
Password: 123456
```

The hostname `my_postgres` works because both containers share the `mynetwork` Docker network.

For this local Docker setup, `DB_SSLMODE=disable` is appropriate when PostgreSQL is not configured for TLS. Do not use this setting for a remote database that requires encrypted connections.

Add `.env` to `.gitignore` so credentials are not committed to source control.

## 8. Build the Python Backend Image

Navigate to the project directory:

```powershell
cd C:\Docker-Project
```

Build the backend image:

```powershell
docker build -t my-webapp:fctp-dev .\backend\
```

Explanation:

- `docker build` builds a Docker image.
- `-t my-webapp:fctp-dev` assigns the image name and tag.
- `.\backend\` specifies the build context containing the Dockerfile, application code and requirements.

Verify the image:

```powershell
docker images
```

The image `my-webapp:fctp-dev` should appear in the list.

## 9. Run the Flask Backend Container

Start the Python container:

```powershell
docker run -d `
  --name my-python-container `
  --network mynetwork `
  --env-file .env `
  -p 5000:5000 `
  --restart unless-stopped `
  my-webapp:fctp-dev
```

### Explanation

| Option | Purpose |
|---|---|
| `--name my-python-container` | Names the backend container |
| `--network mynetwork` | Connects Flask to PostgreSQL |
| `--env-file .env` | Loads database configuration |
| `-p 5000:5000` | Exposes the API on host port 5000 |
| `my-webapp:fctp-dev` | Specifies the backend image |

The backend Dockerfile must start the application using Gunicorn, for example:

```dockerfile
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", "app:app"]
```

The `app:app` value means the Python module is `app.py` and the Flask application object is named `app`.

## 10. Initialize the Database Tables

The Flask application requires the `students` and `items` tables. If they have not been created, the API will return a PostgreSQL error stating that the relation does not exist.

Connect to PostgreSQL:

```powershell
docker exec -it my_postgres psql -U admin -d mydatabase
```

Execute the following SQL:

```sql
CREATE TABLE IF NOT EXISTS students (
    student_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    birth_date DATE,
    email VARCHAR(100),
    enrolled_date DATE
);

CREATE TABLE IF NOT EXISTS items (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT
);
```

Verify the tables:

```sql
\dt
```

Exit the PostgreSQL shell:

```sql
\q
```

For repeated deployments, use a database migration or initialization process rather than relying on manual SQL execution.

## 11. Verify All Containers

Run:

```powershell
docker ps
```

You should see these containers:

```text
my-frontend-container
my-python-container
my_postgres
```

Their expected host port mappings are:

| Container | Host port |
|---|---:|
| Frontend | 3000 |
| Flask backend | 5000 |
| PostgreSQL | 5432 |

Container status should show `Up`.

Check the backend environment variables:

```powershell
docker exec my-python-container printenv DB_HOST
docker exec my-python-container printenv DB_PORT
docker exec my-python-container printenv DB_NAME
docker exec my-python-container printenv DB_USERNAME
```

Expected values:

```text
my_postgres
5432
mydatabase
admin
```

Check database readiness:

```powershell
docker exec my_postgres pg_isready -U admin -d mydatabase
```

## 12. Test the Flask API

Open PowerShell and run:

```powershell
curl.exe -i http://localhost:5000/
```

Test application health:

```powershell
curl.exe -i http://localhost:5000/health
```

Test database readiness:

```powershell
curl.exe -i http://localhost:5000/ready
```

Retrieve students:

```powershell
curl.exe -i http://localhost:5000/students
```

Retrieve items:

```powershell
curl.exe -i http://localhost:5000/items
```

If the tables are empty, the collection endpoints should return an empty JSON array:

```json
[]
```

## 13. Create a Student Record

Run this PowerShell command:

```powershell
curl.exe -i `
  -X POST `
  http://localhost:5000/students `
  -H "Content-Type: application/json" `
  -d '{\"first_name\":\"Rahul\",\"last_name\":\"Sharma\",\"birth_date\":\"2000-01-15\",\"email\":\"rahul@example.com\",\"enrolled_date\":\"2026-10-27\"}'
```

The API should return HTTP `201 Created` if the insertion succeeds.

Retrieve the records again:

```powershell
curl.exe -i http://localhost:5000/students
```

## 14. Create an Item

Run:

```powershell
curl.exe -i `
  -X POST `
  http://localhost:5000/items `
  -H "Content-Type: application/json" `
  -d '{\"name\":\"Docker Book\",\"description\":\"Learning Docker fundamentals\"}'
```

Retrieve all items:

```powershell
curl.exe -i http://localhost:5000/items
```

These tests verify that the Flask API can receive HTTP requests, connect to PostgreSQL, execute SQL statements and return JSON responses.

## 15. Troubleshooting Common Errors

### Error 1: PostgreSQL password authentication failed

Example:

```text
FATAL: password authentication failed for user "admin"
```

Check the backend's configured database username and password:

```powershell
docker exec my-python-container printenv DB_USERNAME
docker exec my-python-container printenv DB_PASSWORD
```

The configured password must match the actual PostgreSQL user's password. Changing `POSTGRES_PASSWORD` does not reset an existing database user's password when the data directory has already been initialized.

### Error 2: Relation does not exist

Example:

```text
relation "students" does not exist
```

Create the required tables using the SQL in Section 10.

### Error 3: PostgreSQL connection refused

Check the container:

```powershell
docker ps
docker logs my_postgres
```

Verify that both containers share the same network:

```powershell
docker inspect my_postgres --format '{{json .NetworkSettings.Networks}}'
docker inspect my-python-container --format '{{json .NetworkSettings.Networks}}'
```

### Error 4: CORS error

Check the Flask CORS configuration. The frontend origin must be permitted, for example:

```text
http://localhost:3000
```

The Flask application must actually use the `CORS_ORIGINS` environment variable if you configure it in `.env`. Setting the variable alone does not change the application's CORS policy.

### Error 5: Backend container fails to start

Check its logs:

```powershell
docker logs --tail 100 my-python-container
```

Inspect its status:

```powershell
docker ps -a
```

## 16. Useful Docker Commands

View running containers:

```powershell
docker ps
```

View all containers:

```powershell
docker ps -a
```

View images:

```powershell
docker images
```

View backend logs:

```powershell
docker logs -f my-python-container
```

View PostgreSQL logs:

```powershell
docker logs -f my_postgres
```

Restart the backend:

```powershell
docker restart my-python-container
```

Restart PostgreSQL:

```powershell
docker restart my_postgres
```

Stop the backend:

```powershell
docker stop my-python-container
```

Stop PostgreSQL:

```powershell
docker stop my_postgres
```

Remove the backend container:

```powershell
docker rm -f my-python-container
```

Remove the PostgreSQL container:

```powershell
docker rm -f my_postgres
```

Removing the PostgreSQL container does not remove the named `pgdata` volume.

**Warning:** Do not delete the `pgdata` volume unless you intentionally want to remove the persisted database data.

## 17. Final Architecture

```text
Browser
   |
   | HTTP :3000
   v
Frontend Container
   |
   | HTTP API :5000
   v
Flask + Gunicorn Container
   |
   | PostgreSQL TCP :5432
   v
PostgreSQL Container
   |
   v
Docker Volume: pgdata
```

The frontend communicates with Flask through the published API port. Flask communicates with PostgreSQL over the internal Docker network using `my_postgres` as the hostname. PostgreSQL stores its data in the persistent `pgdata` volume.

## Conclusion

You have now covered the complete workflow for building and running a containerized Flask application with PostgreSQL.

The key concepts are:

1. **Docker images** package application code and dependencies.
2. **Containers** run the packaged applications.
3. **Docker networks** enable container-to-container communication.
4. **Environment variables** supply runtime configuration.
5. **Docker volumes** preserve database data.
6. **Gunicorn** serves the Flask application.
7. **HTTP endpoints and SQL queries** verify the end-to-end application workflow.

For production environments, use supported software versions, secure secrets management, database migrations, restricted network exposure and appropriately configured TLS.
