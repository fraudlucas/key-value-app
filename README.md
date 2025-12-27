# Key-Value REST API ([Docker course project](https://www.udemy.com/course/complete-docker-kubernetes/))

This repository contains a small key-value REST API implemented with Node.js, Express and MongoDB. It was built as part of a Docker course to demonstrate packaging a simple service in a container and running it together with a MongoDB instance. This version is from before introducing Docker Compose — the project shows how to run the database and backend using individual Docker containers and helper scripts.

**Objective:** Provide a minimal, clear example of a key-value store REST API to practice Docker concepts (images, containers, volumes, networks), and to prepare for moving to a Docker Compose setup.

Project layout
- **backend/**: Node.js backend service
  - `src/server.js`: Express app that connects to MongoDB and mounts routes
  - `src/routes/store.js`: Key-value CRUD routes (`POST /store`, `GET /store/:key`, `PUT /store/:key`, `DELETE /store/:key`)
  - `src/routes/health.js`: Health-check route
  - `src/models/keyValue.js`: Mongoose model for storing `key` and `value`
  - `Dockerfile.dev`: Dockerfile used for development (bind-mounts `src` into container when running)
  - `package.json`: dependency and start scripts
- **db-config/**: Database initialization scripts
  - `mongo-init.js`: Creates the database user and database on first startup
- `setup.sh`, `start-db.sh`, `start-backend.sh`, `cleanup.sh`: helper scripts to create volumes/networks, start the DB container, start the backend container, and clean up resources

How the app works
- The backend exposes a simple REST API to store and retrieve string key-value pairs.
- Data is saved in MongoDB using Mongoose and the `KeyValue` model, which enforces unique keys.
- The backend expects several environment variables for connecting to MongoDB and for the server port (see Environment variables below).

API Endpoints (examples)
- Create a key-value pair

  POST /store
  Body: { "key": "myKey", "value": "some value" }

- Retrieve a value

  GET /store/myKey

- Update a value

  PUT /store/myKey
  Body: { "value": "new value" }

- Delete a value

  DELETE /store/myKey

Health check
- GET /health

Environment variables
- `KEY_VALUE_DB`: database name created/used by the app
- `KEY_VALUE_USER`: username created by `mongo-init.js`
- `KEY_VALUE_PASSWORD`: password for the created user
- `MONGODB_HOSTNAME`: hostname or service name where MongoDB is reachable (e.g., `localhost` or `mongodb` when on the same Docker network)
- `PORT`: port where the backend listens (container port is typically `3000`)

Running locally with the provided scripts (no docker-compose)

1. Prepare volumes and network (creates a Docker volume and network used by the scripts):

```bash
./setup.sh
```

2. Start the MongoDB container (the script mounts `db-config/mongo-init.js` so the DB user and DB are created on initialization):

```bash
./start-db.sh
```

3. Start the backend service (builds the image from `backend/Dockerfile.dev`, mounts the `backend/src` folder to allow live edits during development):

```bash
./start-backend.sh
```

4. Test the API with `curl` or any HTTP client (example):

```bash
# create
curl -X POST -H "Content-Type: application/json" -d '{"key":"foo","value":"bar"}' http://localhost:3000/store

# read
curl http://localhost:3000/store/foo

# update
curl -X PUT -H "Content-Type: application/json" -d '{"value":"baz"}' http://localhost:3000/store/foo

# delete
curl -X DELETE http://localhost:3000/store/foo
```

Notes about the Docker setup
- `backend/Dockerfile.dev` uses `node:22-alpine`, runs `npm ci` and defaults to `npm run dev` (nodemon). The `start-backend.sh` script mounts `./backend/src` into `/app/src` for fast development iterations.
- The `start-db.sh` script uses the official `mongodb/mongodb-community-server` image and mounts `db-config/mongo-init.js` into `/docker-entrypoint-initdb.d/` to create the application DB and user on first startup.

Explanation of mounts and network used
- **Bind mount (development code hot-reload):** The backend container run in `start-backend.sh` includes a bind mount from the host `./backend/src` to the container path `/app/src` using `-v ./backend/src:/app/src`. This is a bind mount, not a Docker-managed volume. It means files you edit on your host are immediately available inside the container. Because the container runs `nodemon` (via `npm run dev`), the server auto-restarts on changes — great for iterative development. Note: bind mounts reflect the host filesystem permissions and are ideal for local development, but not recommended for production image builds.

- **Docker volume (persistent DB storage):** The MongoDB container mounts a Docker volume into the container's data directory with `-v $VOLUME_NAME:/data/db`. This is a Docker-managed volume. Volumes are stored by Docker and are independent of the container lifecycle; when the container stops or is removed, the volume remains. That allows MongoDB data to persist across container restarts and upgrades. The `setup.sh` script creates the named volume referenced by `$VOLUME_NAME`.

- **File mount for DB initialization:** To create the application database and user on first run, `start-db.sh` mounts the host file `db-config/mongo-init.js` into the container initialization directory using `-v ./db-config/mongo-init.js:/docker-entrypoint-initdb.d/mongo-init.js:ro`. The `:ro` suffix mounts it read-only. Files placed into `/docker-entrypoint-initdb.d/` are executed by the MongoDB image on first initialization — a convenient way to set up users and initial state.

- **User-defined Docker network:** The scripts create and use a user-defined Docker network (created by `setup.sh`) and pass `--network $NETWORK_NAME` to `docker run`. A user-defined bridge network provides automatic DNS-based service discovery between containers attached to the same network (containers can resolve each other by name). In this project `start-backend.sh` sets `MONGODB_HOSTNAME="mongodb"` and expects the MongoDB container to be accessible by that hostname on the same network. Using a user-defined network isolates the containers from other Docker containers and enables reliable inter-container communication without exposing internal ports to the host.

Why there's no docker-compose yet
- This project stage intentionally manages containers individually to teach how images, networks and volumes work. Docker Compose will be introduced later to simplify multi-container orchestration and provide an easier developer experience.

Next steps / suggestions
- Add a `docker-compose.yml` to orchestrate the backend and MongoDB together (simpler startup, environment management, and logs)
- Add tests for the API routes (supertest + jest or similar)
- Add input validation and request schema (e.g., `express-validator`)
- Add persistent configuration for local development (example `.env` file and `.env.*` templates are already referenced by the scripts)

Notes
- This project is intentionally minimal to focus on demonstrating Docker concepts and service wiring.
- The env files are being versioned intentionally for study purposes.
