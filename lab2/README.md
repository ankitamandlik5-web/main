\# Lab 2 · Containerize it, then debug it



\- \*\*Image:\*\* `ghcr.io/ankitamandlik5-web/course-api:lab2`

\- \*\*Base image digest:\*\* `node:24-alpine@sha256:ebfe2f90462722a7a4de65e91990e97fe0d401c70e0e762c5b53302f905ec1c1`



\## Part 1 · Images and layers



| Image | Size | Distro | Default user |

|-------|-----:|--------|--------------|

| node:24 | | | |

| node:24-slim | | | |

| node:24-alpine | | | |



cow:bad = \_\_\_ MB · cow:good = \_\_\_ MB



| Build | RUN step CACHED? | Build time |

|-------|------------------|-----------:|

| b · app.txt changed | | |

| c · deps.txt changed | | |

| d · app.txt changed, wrong order | | |



\## Part 2 · The course API image



| Step | Image | Size |

|------|-------|-----:|

| Naive | course-api:naive | |

| + .dockerignore, npm ci --omit=dev, exec form | course-api:step1 | |

| Multi-stage, node:24-alpine, non-root | course-api:lab2 | 248 MB |

| Reduction against naive | | |



\### Non-root check



uid=1000(node) gid=1000(node) groups=1000(node),1000(node)





\### Health check



{"status":"ok"}





The container became:



Up ... (healthy)





\### Stop time



After the SIGTERM handler:



TotalSeconds: 0.2609 Exit code: 0





The API shuts down in less than 2 seconds.



\### Clean image check



The `/app` directory contains:



db.js node\_modules package.json server.js validate.js





There is no `.env`, `.git`, or Dockerfile in the image.



\### Dockerfile



The final image uses:



\- Multi-stage build

\- `node:24-alpine`

\- Pinned SHA256 digest

\- Production dependencies only

\- Non-root `node` user

\- Health check

\- Exec-form CMD

\- npm cache mount



\## Part 3 · Linux drills



\### 3.1 OS and kernel



Inside the Ubuntu container:



PRETTYNAME="Ubuntu 24.04.5 LTS" VERSIONID="24.04"





Kernel:



6.18.33.2-microsoft-standard-WSL2





`/etc/os-release` describes the Ubuntu userspace inside the container. `uname -r` shows the Linux kernel provided by WSL2.



\### 3.2 sed



`/lab/devopstools`:



chef tech ansible tech docker tech





`/lab/newtools.txt`:



chef tools ansible tools docker tools





\### 3.3 HTTP requests



20 requests were sent to the nginx container.



Result:



0





\### 3.4 File permissions



\-rwxr-x--- 1 root root 7 Oct 7 18:05 /lab/f





The `student` user receives:



cat: /lab/f: Permission denied





The file belongs to `root:root` and others have no read permission.



\### 3.5 Environment variables



Without `export`:



child sees:





With `export`:



child sees: staging





A normal shell variable is not automatically inherited by child processes. `export` makes it part of the environment.



\### 3.6 PID 1 and stopping



The lab container exited with:



137





Exit code 137 means the container was killed after it did not stop gracefully within Docker's timeout.



\### 3.7 Networking



From the host:



ComputerName : localhost RemoteAddress : ::1 RemotePort : 8081 TcpTestSucceeded : True





The nginx container is reachable through the published host port `8081`.



Inside the Docker network, containers use their container/service name such as `web`. From the host, the published port `8081` reaches nginx.



\## Part 4 · Broken containers



\### lab2-broken:1



\- \*\*Symptom:\*\* `Exited (127)` and `nodemon: not found`

\- \*\*Cause:\*\* The production installation excludes the `nodemon` devDependency, but the container starts the development script.

\- \*\*Fix:\*\* Use `CMD \["node", "server.js"]`.



\### lab2-broken:2



\- \*\*Symptom:\*\* `exec /entrypoint.sh: no such file or directory`

\- \*\*Cause:\*\* The entrypoint script has an invalid line ending/shebang.

\- \*\*Fix:\*\* Convert the script to Unix LF line endings and use a valid shebang.



\### lab2-broken:3



\- \*\*Symptom:\*\* The container runs but the published port cannot reach the API.

\- \*\*Cause:\*\* The application listens only on a loopback address inside the container.

\- \*\*Fix:\*\* Make the server listen on `0.0.0.0`.



\### lab2-broken:4



\- \*\*Symptom:\*\* Database connection fails with `ECONNREFUSED 127.0.0.1:5432`.

\- \*\*Cause:\*\* `DB\_HOST` points to localhost instead of the database container.

\- \*\*Fix:\*\* Set `DB\_HOST` to the database service/container name.



\### lab2-broken:5



\- \*\*Symptom:\*\* `EACCES: permission denied` when opening `/app/data/todos.log`.

\- \*\*Cause:\*\* The application runs as a non-root user but `/app/data` is owned by root.

\- \*\*Fix:\*\* Create/copy the directory with `node:node` ownership, for example using `COPY --chown=node:node`.



\### lab2-broken:6



\- \*\*Symptom:\*\* `docker stop` takes about 10 seconds and exits with code 137.

\- \*\*Cause:\*\* The process does not handle SIGTERM correctly.

\- \*\*Fix:\*\* Add a SIGTERM handler and gracefully close the server/database pool.



\### lab2-broken:7



\- \*\*Symptom:\*\* Container remains unhealthy although the API works.

\- \*\*Cause:\*\* The HEALTHCHECK is using the wrong endpoint/port or a command unavailable in the image.

\- \*\*Fix:\*\* Check the health configuration and use a valid endpoint such as `/healthz`.



\## Answers



\### 1. Why is cow:bad 50 MB bigger?



Docker images use layers. The `dd` command creates a 50 MB layer. Removing the file in a later layer does not remove the data from the earlier layer. The delete is recorded as a filesystem change, but the original layer remains.



\### 2. Why does COPY/RUN order affect rebuild time?



Docker caches individual layers. If a frequently changing file is copied before an expensive `RUN`, changing that file invalidates the cache for the expensive step. Dependencies should therefore be copied and installed before application source code.



\### 3. Why did docker stop take 10 seconds before SIGTERM handling?



Docker first sends SIGTERM to the main process. Without a handler, the Node process did not shut down gracefully, so Docker waited for its timeout and then sent SIGKILL. The container therefore exited with code 137.



\### 4. Name three things the naive image contains that course-api:lab2 does not.



Examples:



\- Development dependencies

\- The full Node base image instead of Alpine

\- Source/build files excluded by `.dockerignore`

\- Dockerfile and documentation files

\- `.env` when `.dockerignore` is missing

