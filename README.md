# 5-Voting-App-Docker — Docker Compose + PostgreSQL Bind Mount

> **University DevOps Practical Quiz** — Docker ki official [example-voting-app](https://github.com/dockersamples/example-voting-app) ko Docker Compose ke zariye 5 services ke saath chalana, aur PostgreSQL ka data ek **bind mount** (`./postgres-data`) mein host machine pe persist karna.

---

## Table of Contents

1. [Introduction & Objectives](#1-introduction--objectives)
2. [Technologies Used](#2-technologies-used)
3. [Architecture & Workflow](#3-architecture--workflow)
4. [Repository Clone Karna](#4-repository-clone-karna)
5. [Project Structure](#5-project-structure)
6. [Docker Compose Implementation](#6-docker-compose-implementation)
7. [Bind Mount aur `-v` Flag](#7-bind-mount-aur--v-flag)
8. [Complete Configuration Explanation](#8-complete-configuration-explanation)
9. [Run Karna — Step by Step Commands](#9-run-karna--step-by-step-commands)
10. [Application Testing (Multiple Votes)](#10-application-testing-multiple-votes)
11. [Database Persistence Verification](#11-database-persistence-verification)
12. [Volume Mapping Verification (`docker inspect`)](#12-volume-mapping-verification-docker-inspect)
13. [Network Verification](#13-network-verification)
14. [Troubleshooting](#14-troubleshooting)
15. [Viva Questions & Answers](#15-viva-questions--answers)
16. [Screenshots / Evidence Checklist](#16-screenshots--evidence-checklist)
17. [GitHub Submission](#17-github-submission)
18. [Final Submission Checklist](#18-final-submission-checklist)

---

## 1. Introduction & Objectives

Ye project ek chhoti **distributed application** hai jisme 5 alag containers mil kar kaam karte hain. User browser mein **Cats vs Dogs** mein se ek option ko vote karta hai, aur doosre page pe live result dikhta hai.

**Objectives:**

- Official GitHub repository ko Git se clone karna.
- 5 services (Vote, Result, Worker, Redis, PostgreSQL) ko **Docker Compose** se ek command mein chalana.
- Docker **networks** (`front-tier`, `back-tier`) se containers ki aapas mein communication.
- PostgreSQL data ko **bind mount** `./postgres-data:/var/lib/postgresql/data` se host pe store karna.
- Ye prove karna ke `docker compose down` ke baad bhi data **persist** rehta hai.
- `docker inspect` se volume mapping verify karna.

---

## 2. Technologies Used

| Technology | Role | Version used in testing |
|---|---|---|
| Docker Engine | Containers chalana | 29.9.2 |
| Docker Compose (v2 plugin) | Multi-container orchestration | v5.6.1 |
| Python (Flask) | **Vote** web app | `vote/Dockerfile` |
| Node.js (Express + Socket.io) | **Result** web app | `result/Dockerfile` |
| .NET (C#) | **Worker** — Redis se vote utha kar DB mein likhta hai | `worker/Dockerfile` |
| Redis | In-memory queue (votes temporarily yahan aate hain) | `redis:alpine` |
| PostgreSQL | Permanent database | `postgres:15-alpine` |
| Git / GitHub | Version control & submission | — |

---

## 3. Architecture & Workflow

![Architecture diagram](architecture.excalidraw.png)

```
                    ┌──────────── front-tier network ─────────────┐
   Browser          │                                              │        Browser
 localhost:8080 ──► │  ┌──────────┐                  ┌──────────┐  │ ◄── localhost:8081
                    │  │   VOTE   │                  │  RESULT  │  │
                    │  │ (Python) │                  │ (Node.js)│  │
                    └──┼──────────┼──────────────────┼──────────┼──┘
                    ┌──┼──────────┼─── back-tier ────┼──────────┼──┐
                    │  └────┬─────┘                  └────▲─────┘  │
                    │       │ (1) LPUSH vote               │ (4) SELECT  │
                    │  ┌────▼─────┐   (2) BLPOP   ┌───────┴────┐   │
                    │  │  REDIS   │ ◄──────────── │ POSTGRES db│   │
                    │  │  queue   │      ┌──────► │            │   │
                    │  └──────────┘      │        └─────┬──────┘   │
                    │            ┌───────┴──┐           │          │
                    │            │  WORKER  │ (3) INSERT/UPDATE    │
                    │            │  (.NET)  │                      │
                    │            └──────────┘                      │
                    └──────────────────────────────────────────────┘
                                                          │ bind mount
                                                          ▼
                                     Host: ./postgres-data  (persistent)
```

**Workflow (simple words mein):**

1. User `http://localhost:8080` pe vote karta hai → **Vote** app vote ko **Redis** ki list (`votes`) mein push karta hai.
2. **Worker** lagatar Redis se vote nikalta hai.
3. Worker vote ko **PostgreSQL** ki `votes` table mein save karta hai (har voter ka sirf 1 vote — agar wahi voter dobara vote kare to UPDATE hota hai).
4. **Result** app PostgreSQL se count padhta hai aur `http://localhost:8081` pe live dikhata hai.
5. PostgreSQL ki files host ke `./postgres-data` folder mein save hoti hain — container delete ho jaye tab bhi data safe.

---

## 4. Repository Clone Karna

```bash
# Official repo clone karo
git clone https://github.com/dockersamples/example-voting-app.git votingapp_Quiz_2

# Project folder mein jao
cd votingapp_Quiz_2

# Confirm karo ke files aa gayi hain
ls
```

**Expected output (`ls`):**

```
architecture.excalidraw.png  docker-compose.images.yml  docker-compose.yml  docker-stack.yml
healthchecks  k8s-specifications  LICENSE  MAINTAINERS  README.md  result  seed-data  vote  worker
```

> Is submission repo (`5-voting-app-docker`) ko clone karna ho to:
> `git clone https://github.com/RakibAkram5/5-voting-app-docker.git`

---

## 5. Project Structure

```
votingapp_Quiz_2/
├── docker-compose.yml        # ⭐ Main file — 5 services + seed, networks, bind mount
├── docker-compose.images.yml # Official alternative: pre-built images use karta hai
├── docker-stack.yml          # Docker Swarm ke liye (is quiz mein use nahi)
├── k8s-specifications/       # Kubernetes YAMLs (is quiz mein use nahi)
├── healthchecks/
│   ├── postgres.sh           # DB healthy hai ya nahi (SELECT 1)
│   └── redis.sh              # Redis healthy hai ya nahi (PING)
├── vote/                     # Python Flask app + Dockerfile
├── result/                   # Node.js app + Dockerfile
├── worker/                   # .NET worker + Dockerfile
├── seed-data/                # Optional: test votes generate karne ke liye
├── postgres-data/            # ⭐ Bind mount folder (git mein ignore — upload NAHI hota)
├── .gitignore
└── README.md
```

---

## 6. Docker Compose Implementation

Official `docker-compose.yml` ko **as-is** rakha gaya hai. Sirf **ek** zaroori change kiya gaya hai:

```diff
     volumes:
-      - "db-data:/var/lib/postgresql/data"
+      - "./postgres-data:/var/lib/postgresql/data"
       - "./healthchecks:/healthchecks"
 ...
-volumes:
-  db-data:
```

Yaani **named volume `db-data`** ko **bind mount `./postgres-data`** se replace kiya, aur ab named volume ki zaroorat nahi to top-level `volumes:` block hata diya. Baqi sab (ports, healthchecks, networks, `seed` profile) official jaisa hai.

**Final `db` service:**

```yaml
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: "postgres"
      POSTGRES_PASSWORD: "postgres"
    volumes:
      # Bind mount (same as: docker run -v ./postgres-data:/var/lib/postgresql/data)
      # PostgreSQL files are stored on the host in ./postgres-data
      - "./postgres-data:/var/lib/postgresql/data"
      - "./healthchecks:/healthchecks"
    healthcheck:
      test: /healthchecks/postgres.sh
      interval: "5s"
    networks:
      - back-tier
```

---

## 7. Bind Mount aur `-v` Flag

### `-v` flag kya hai?

`docker run` mein `-v HOST_PATH:CONTAINER_PATH` likhne se host ka ek folder container ke andar "dikhne" lagta hai. Dono jagah **same files** hoti hain — copy nahi, balke wahi folder.

```bash
# Compose ke bina, sirf docker run se same kaam:
docker run -d --name db \
  -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres \
  -v "$(pwd)/postgres-data:/var/lib/postgresql/data" \
  postgres:15-alpine
```

Compose file mein `volumes:` ke neeche likha `"./postgres-data:/var/lib/postgresql/data"` bilkul **isi `-v` flag ka equivalent** hai.

### Bind mount kaise kaam karta hai?

- Left side (`./postgres-data`) = **host** ka path (compose file ke folder ke relative).
- Right side (`/var/lib/postgresql/data`) = **container** ke andar path, jahan PostgreSQL apni database files likhta hai.
- PostgreSQL jo bhi likhta hai, wo asal mein host ke `postgres-data/` folder mein likha jata hai.
- Container delete ho jaye (`docker compose down`), folder host pe rehta hai. Naya container same folder mount karta hai → purana data wapas.

### Bind Mount vs Named Volume

| Point | Bind Mount (`./postgres-data:/...`) | Named Volume (`db-data:/...`) |
|---|---|---|
| Data kahan store hota hai | Aapke **chosen host folder** mein | Docker ke managed area mein (`/var/lib/docker/volumes/...`) |
| Kaun manage karta hai | Aap (user) | Docker |
| `docker volume ls` mein dikhta hai? | ❌ Nahi | ✅ Haan |
| Host se direct dekhna/backup | Aasaan — normal folder | `docker volume` commands ya root access chahiye |
| Portability | Host ke path pe depend karta hai | Har host pe same chalta hai |
| Permissions | Host ke file permissions apply hote hain (issue ho sakte hain) | Docker handle karta hai |
| `docker inspect` mein `Type` | `bind` | `volume` |
| Best for | Development, source code sharing, local backups | Production databases |

> **Note:** Is project mein `postgres-data/` ka owner UID `70` (Alpine image ka `postgres` user) hota hai aur permission `700`. Is liye normal user `ls postgres-data` kare to `Permission denied` aa sakta hai — ye **normal** hai, data safe hai.

---

## 8. Complete Configuration Explanation

| Service | Build / Image | Ports (host:container) | Networks | Depends on | Healthcheck | Notes |
|---|---|---|---|---|---|---|
| **vote** | `./vote` (target `dev`) | `8080:80` | front-tier, back-tier | redis (healthy) | `curl -f http://localhost` | `./vote` code bind-mounted (live reload) |
| **result** | `./result` | `8081:80`, `127.0.0.1:9229:9229` | front-tier, back-tier | db (healthy) | — (official mein nahi) | `nodemon` se chalta hai; 9229 = Node debugger, sirf localhost |
| **worker** | `./worker` | — (internal) | back-tier | redis + db (healthy) | — (official mein nahi) | Redis → Postgres |
| **redis** | `redis:alpine` | — (internal) | back-tier | — | `/healthchecks/redis.sh` | Queue |
| **db** | `postgres:15-alpine` | — (internal) | back-tier | — | `/healthchecks/postgres.sh` | **Bind mount `./postgres-data`** |
| **seed** | `./seed-data` | — | front-tier | vote (healthy) | — | Sirf `--profile seed` se chalta hai |

**Key concepts:**

- **`depends_on` + `condition: service_healthy`** — worker tab tak start nahi hota jab tak Redis aur DB *healthy* na ho jayein. Isse "connection refused" errors kam hote hain.
- **Networks** — `front-tier` pe sirf user-facing apps (vote, result). `back-tier` pe sab backend. Redis aur DB ka koi port host pe expose nahi — ye **security** ke liye achha hai.
- **Service name = DNS name** — Compose network pe container ek doosre ko naam se dhoondte hain: vote code mein `redis` host, worker/result mein `db` host.
- **`POSTGRES_PASSWORD: "postgres"`** — ye official demo ka default hai, sirf local learning ke liye. Real project mein password `.env` file mein rakho (jo `.gitignore` mein hai) aur kabhi public repo mein na daalo.

---

## 9. Run Karna — Step by Step Commands

### Step 1 — Docker check

```bash
docker --version
docker compose version
```
**Purpose:** Confirm karna ke Docker aur Compose installed hain.
**Expected:** `Docker version 2x.x.x ...` aur `Docker Compose version v2.x+`

### Step 2 — Compose file validate karo

```bash
docker compose config -q && echo CONFIG_OK
```
**Purpose:** YAML mein syntax error to nahi. **Expected:** `CONFIG_OK`

### Step 3 — Build aur start

```bash
docker compose up -d --build
```
**Purpose:** `--build` = vote/result/worker ki images banao; `-d` = background (detached) mein chalao.
**Expected (end of output):**
```
 Container votingapp_quiz_2-db-1 Healthy
 Container votingapp_quiz_2-worker-1 Started
 Container votingapp_quiz_2-result-1 Started
```

> Pehli dafa `postgres-data/` folder na ho to Docker khud bana deta hai aur PostgreSQL usme naya database initialize karta hai.

### Step 4 — Status check

```bash
docker compose ps
```
**Expected (actual output from testing):**
```
SERVICE   STATUS
db        Up 39 seconds (healthy)
redis     Up 38 seconds (healthy)
result    Up 33 seconds
vote      Up 32 seconds (healthy)
worker    Up 31 seconds
```
`result` aur `worker` ka official repo mein healthcheck define nahi, is liye un pe sirf `Up` dikhta hai — ye normal hai. Unki health logs se confirm karo (Step 5).

### Step 5 — Logs

```bash
docker compose logs worker --tail 20
docker compose logs result --tail 20
```
**Expected worker log:** `Connected to db`, `Found redis at 172.x.x.x`, aur vote ke baad `Processing vote for 'a' by '...'`.

### Step 6 — Browser

- Vote: <http://localhost:8080>
- Result: <http://localhost:8081>

---

## 10. Application Testing (Multiple Votes)

**Important:** App har browser ko ek `voter_id` cookie deta hai. **Ek browser = ek vote.** Dobara click karne se naya vote nahi banta, balke vote **change** hota hai.

### Option A — Browser se

1. Normal window → `localhost:8080` → **Cats** pe vote.
2. Incognito / doosra browser → **Dogs** pe vote.
3. Ek aur incognito → **Cats**.
4. `localhost:8081` pe result dekho — percentages update hone chahiye.

### Option B — Terminal se (har vote alag `voter_id` ke saath)

```bash
# 3 votes "a" (Cats), 2 votes "b" (Dogs) — har ek alag voter
for v in a a a b b; do
  id=$(cat /proc/sys/kernel/random/uuid)
  curl -s -o /dev/null -w "vote=$v HTTP %{http_code}\n" -b "voter_id=$id" -d "vote=$v" http://localhost:8080/
done
```

### Database mein count check karo

```bash
docker compose exec db psql -U postgres -c "SELECT vote, COUNT(*) FROM votes GROUP BY vote ORDER BY vote;"
```

**Actual test result (is project pe chala kar):**

| Stage | `a` (Cats) | `b` (Dogs) |
|---|---|---|
| Before testing (purane votes) | 0 | 2 |
| +3 Cats, +2 Dogs (5 alag voters) | 3 | 4 |
| Ek Dogs voter ne vote Cats mein change kiya | **4** | **3** |

Worker log ne har vote confirm kiya:
```
worker-1  | Processing vote for 'a' by 'f0c4fe6f-3aac-4fac-ac35-4db0b29fdf9e'
worker-1  | Processing vote for 'b' by '53a01497-865b-4258-8919-8616ca0aec88'
```

### Option C — Official seed (bohot saare votes)

```bash
docker compose --profile seed up seed
```
Ye official `seed-data` container chala kar `ab` tool se bohot saare votes bhejta hai (load test).

---

## 11. Database Persistence Verification

```bash
# 1. Votes note karo
docker compose exec db psql -U postgres -c "SELECT vote, COUNT(*) FROM votes GROUP BY vote;"

# 2. Saare containers + networks DELETE karo (bina -v ke!)
docker compose down

# 3. Confirm: koi container nahi bacha
docker compose ps -a

# 4. Confirm: data folder host pe ab bhi hai
sudo ls postgres-data
#   (sudo nahi to:)  docker run --rm -v "$(pwd)/postgres-data:/d:ro" alpine ls /d

# 5. Dobara start
docker compose up -d

# 6. Same query — votes same hone chahiye
docker compose exec db psql -U postgres -c "SELECT vote, COUNT(*) FROM votes GROUP BY vote;"
```

**Actual result:**

```
# down ke baad docker compose ps -a:
NAME      IMAGE     COMMAND   SERVICE   CREATED   STATUS    PORTS        <- empty

# postgres-data ka content (host pe):
PG_VERSION  base  global  pg_commit_ts  pg_dynshmem ...

# up ke baad query:
 vote | count
------+-------
 a    |     4
 b    |     3
```

✅ Containers delete hone ke baad bhi votes **4 / 3** wapas aaye → **persistence verified**.

> ⚠️ `docker compose down -v` named volumes delete karta hai. Bind mount folder ko ye delete nahi karta, lekin aadat ke taur pe database wale project mein `-v` use mat karo, aur **kabhi `postgres-data/` ko `rm -rf` mat karo** — sara data chala jayega.

---

## 12. Volume Mapping Verification (`docker inspect`)

```bash
docker inspect votingapp_quiz_2-db-1 \
  --format '{{range .Mounts}}{{.Type}} | {{.Source}} -> {{.Destination}} | RW={{.RW}}{{println}}{{end}}'
```

**Actual output:**

```
bind | /home/muhammad-rakib-akram/votingapp_Quiz_2/healthchecks -> /healthchecks | RW=true
bind | /home/muhammad-rakib-akram/votingapp_Quiz_2/postgres-data -> /var/lib/postgresql/data | RW=true
```

Full JSON:

```bash
docker inspect votingapp_quiz_2-db-1 --format '{{json .Mounts}}' | python3 -m json.tool
```
```json
{
    "Type": "bind",
    "Source": "/home/muhammad-rakib-akram/votingapp_Quiz_2/postgres-data",
    "Destination": "/var/lib/postgresql/data",
    "Mode": "rw",
    "RW": true,
    "Propagation": "rprivate"
}
```

**Kya dekhna hai:** `"Type": "bind"` (named volume hota to `"volume"`), aur `Source` aapka local `postgres-data` folder.

```bash
docker volume ls | grep votingapp   # kuch nahi aana chahiye — named volume use nahi ho raha
```

> Container ka naam aapke folder ke naam pe depend karta hai (`<folder>-db-1`). `docker compose ps` se apna exact naam dekh lo.

---

## 13. Network Verification

```bash
docker network ls --filter name=votingapp
docker network inspect votingapp_quiz_2_back-tier \
  --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{println}}{{end}}'
```

**Actual output:**

```
NAME                          DRIVER    SCOPE
votingapp_quiz_2_back-tier    bridge    local
votingapp_quiz_2_front-tier   bridge    local

front-tier:  vote, result
back-tier:   vote, worker, redis, db, result
```

DNS (service name se connect) test:

```bash
docker compose exec vote python -c "import socket; print(socket.gethostbyname('redis'))"
docker compose exec result getent hosts db
```
**Expected:** `172.22.0.x` style IP — matlab container naam se doosre container ko dhoond leta hai.

---

## 14. Troubleshooting

> Golden rule: **pehle logs padho** — `docker compose logs <service>`. Aur **kabhi `postgres-data` delete karke "fix" mat karo.**

| Problem / Error | Wajah | Solution |
|---|---|---|
| `Bind for 0.0.0.0:8080 failed: port is already allocated` | Koi aur process 8080/8081 use kar raha hai | `sudo ss -ltnp \| grep -E '8080\|8081'` se dhoondo, us process ko band karo, ya compose mein `"8090:80"` jaisa doosra host port do |
| `permission denied while trying to connect to the Docker daemon socket` | User `docker` group mein nahi | `sudo usermod -aG docker $USER` phir logout/login (ya `newgrp docker`) |
| `Cannot connect to the Docker daemon` | Docker service band hai | `sudo systemctl start docker` |
| `dependency failed to start: container ... is unhealthy` | DB/Redis healthy nahi hua | `docker compose logs db` padho; `docker inspect --format '{{json .State.Health}}' votingapp_quiz_2-db-1` |
| DB log: `initdb: directory "/var/lib/postgresql/data" exists but is not empty` | `postgres-data` mein koi random file hai lekin valid DB nahi | Folder ka content check karo; data move/backup karo — blind delete nahi |
| DB log: `database files are incompatible with server` | Folder purane/doosre Postgres version se bana hai | Image version wapas match karo (`postgres:15-alpine`), ya `pg_dump` se migrate karo |
| `ls: cannot open directory 'postgres-data': Permission denied` | Folder ka owner UID 70 (container ka postgres user) | Normal hai. `sudo ls postgres-data` use karo; **`chmod 777` mat karo** — Postgres strict permissions maangta hai |
| `git add .` pe `postgres-data` permission error | Git folder padhne ki koshish kar raha | `.gitignore` mein `postgres-data/` hona chahiye (hai) |
| Worker log: `Waiting for db` / `Waiting for redis` loop | Network ya DB issue | `docker compose ps` — db/redis healthy hain? Worker aur db dono `back-tier` pe hain? |
| Result page "0 votes" / update nahi | Worker band hai ya votes same browser se | `docker compose logs worker`; alag browser/incognito se vote karo |
| Vote dobara karne pe count nahi badhta | Same `voter_id` cookie → sirf UPDATE | Ye feature hai, bug nahi. Naya voter = naya browser/incognito |
| `yaml: line X: did not find expected key` | YAML indentation (tabs/spaces) | Sirf spaces use karo; `docker compose config` se validate karo |
| `network ... not found` / purane containers ka conflict | Purane run ke leftovers | `docker compose down` phir `docker compose up -d` |
| Code change ke baad image purani | Rebuild nahi hua | `docker compose up -d --build` |
| `no space left on device` | Disk full (images/build cache) | `docker system df`; `docker image prune` / `docker builder prune` (ye **bind mount data delete nahi karte**) |

---

## 15. Viva Questions & Answers

**Q1. Docker kya hai?**
Docker ek platform hai jo application aur uski saari dependencies ko ek **container** mein pack karta hai, taake wo har machine pe same tareeqe se chale.

**Q2. Image aur Container mein kya farq hai?**
Image ek read-only template/blueprint hai; container us image ka running instance hai. Ek image se kai containers ban sakte hain.

**Q3. Docker Compose kya hai aur kyun use kiya?**
Compose ek YAML file (`docker-compose.yml`) se multiple containers, networks aur volumes ko define karke ek command (`docker compose up`) se chalata hai. 5 services ko alag alag `docker run` se chalana mushkil aur error-prone hota.

**Q4. Is app ki 5 services kaunsi hain aur har ek ka kaam?**
Vote (Python, vote leta hai), Redis (queue), Worker (.NET, Redis → Postgres), PostgreSQL db (permanent storage), Result (Node.js, results dikhata hai).

**Q5. Redis yahan kyun hai, vote seedha DB mein kyun nahi jata?**
Redis ek fast in-memory **queue** hai. Ye vote app ko DB se decouple karta hai — traffic zyada ho to bhi vote app fast rehta hai, aur worker apni speed se DB mein likhta hai.

**Q6. Bind mount kya hai?**
Host ke kisi specific folder ko container ke andar ek path pe mount karna. Yahan `./postgres-data` → `/var/lib/postgresql/data`.

**Q7. Bind mount aur named volume mein farq?**
Bind mount ka location user choose karta hai (host path); named volume Docker manage karta hai (`/var/lib/docker/volumes`) aur `docker volume ls` mein dikhta hai. Inspect mein `Type: bind` vs `Type: volume`.

**Q8. `-v` flag ka syntax kya hai?**
`-v <host_path_or_volume_name>:<container_path>[:ro]`. Path `/` ya `./` se shuru ho to bind mount, sirf naam ho to named volume.

**Q9. `/var/lib/postgresql/data` kya hai?**
Official PostgreSQL image ka default data directory (`PGDATA`) — saari tables, indexes, WAL files yahan hoti hain.

**Q10. Bind mount na ho to kya hoga?**
Data container ki writable layer mein rehta; `docker compose down` (container remove) pe sara data delete ho jata.

**Q11. `docker compose down` aur `docker compose stop` mein farq?**
`stop` sirf containers rokta hai (wapas `start` ho sakte hain). `down` containers aur networks **remove** kar deta hai. `down -v` named volumes bhi delete karta hai.

**Q12. Persistence kaise prove ki?**
Votes count kiye (4/3) → `docker compose down` → `ps -a` empty → `up -d` → same query → 4/3 wapas.

**Q13. Containers aapas mein kaise baat karte hain?**
Compose har project ke liye bridge networks banata hai aur built-in DNS deta hai. Container doosre container ko **service name** se reach karta hai (e.g. host `redis`, host `db`).

**Q14. `front-tier` aur `back-tier` networks kyun alag hain?**
Network segmentation / security. Database aur Redis sirf `back-tier` pe hain aur host pe koi port expose nahi, is liye bahar se direct access nahi ho sakta.

**Q15. `depends_on` with `condition: service_healthy` kya karta hai?**
Ek service ko tab tak start nahi karta jab tak dependency ka healthcheck pass na ho. Sirf `depends_on` start order deta hai, health guarantee nahi.

**Q16. Healthcheck kya hai? Is project mein kaise hai?**
Container ke andar periodically chalne wali command jo batati hai service sahi kaam kar rahi hai. DB ke liye `healthchecks/postgres.sh` (`SELECT 1`), Redis ke liye `redis.sh` (`PING`), Vote ke liye `curl -f http://localhost`.

**Q17. Port mapping `8080:80` ka matlab?**
Host ka port 8080 → container ka port 80. Browser `localhost:8080` kholta hai, traffic container ke 80 pe jata hai.

**Q18. Ek browser se do dafa vote karne pe count kyun nahi badhta?**
Vote app `voter_id` cookie set karta hai; worker DB mein `id` (voter_id) unique rakhta hai, is liye same voter ka dobara vote UPDATE hota hai, naya row nahi.

**Q19. `postgres-data` GitHub pe kyun upload nahi kiya?**
Ye live database files hain — size bara, binary, machine-specific, aur isme sensitive data ho sakta hai. Is liye `.gitignore` mein hai.

**Q20. `docker inspect` se kya verify kiya?**
DB container ke `.Mounts` mein `Type: bind`, `Source: .../postgres-data`, `Destination: /var/lib/postgresql/data`, `RW: true`.

---

## 16. Screenshots / Evidence Checklist

`screenshots/` folder bana kar ye screenshots save karo (filename suggestions):

- [ ] `01-git-clone.png` — `git clone` aur `ls` output
- [ ] `02-compose-file.png` — `docker-compose.yml` ka `db` service (bind mount line highlight)
- [ ] `03-compose-up.png` — `docker compose up -d --build` ka end (`Healthy` / `Started`)
- [ ] `04-compose-ps.png` — `docker compose ps` (5 services Up, db/redis/vote healthy)
- [ ] `05-vote-page.png` — Browser `localhost:8080` (vote cast hone ke baad tick mark)
- [ ] `06-result-page.png` — Browser `localhost:8081` (percentages)
- [ ] `07-worker-logs.png` — `Processing vote for ...` lines
- [ ] `08-db-count-before.png` — `SELECT vote, COUNT(*)` before down
- [ ] `09-compose-down.png` — `docker compose down` + empty `docker compose ps -a`
- [ ] `10-postgres-data-host.png` — `sudo ls postgres-data` (host pe files)
- [ ] `11-db-count-after.png` — up ke baad same count
- [ ] `12-docker-inspect.png` — `docker inspect` Mounts (`Type: bind`)
- [ ] `13-networks.png` — `docker network inspect` front-tier / back-tier
- [ ] `14-github-repo.png` — GitHub pe repo page (README render, `postgres-data` nahi dikh raha)

---

## 17. GitHub Submission

```bash
git status                 # kya change hua
git check-ignore -v postgres-data   # confirm: .gitignore:1:postgres-data/
git remote -v              # origin = apni repo
git branch                 # * main
git add .
git commit -m "Voting app with PostgreSQL bind mount and documentation"
git push -u origin main
```

**Push se pehle security check:**

```bash
git ls-files | grep -Ei 'postgres-data|\.env$|token|secret'   # kuch nahi aana chahiye
```

> GitHub password se push nahi hota. **Personal Access Token (PAT)** use karo (GitHub → Settings → Developer settings → Tokens) ya `gh auth login`. Token kabhi kisi file mein commit mat karo.

---

## 18. Final Submission Checklist

- [x] Official repo clone kiya (`dockersamples/example-voting-app`)
- [x] Project structure analyze kiya
- [x] `docker-compose.yml` mein sirf zaroori change (named volume → bind mount)
- [x] 5 services: vote, result, worker, redis, db — sab running
- [x] `front-tier` / `back-tier` networks configured & verified
- [x] Bind mount `./postgres-data:/var/lib/postgresql/data`
- [x] Multiple votes test kiye (3 + 2 + 1 change) — counts sahi
- [x] `docker compose down` → `up` ke baad data persist
- [x] `docker inspect` se `Type: bind` verify
- [x] `.gitignore` mein `postgres-data/`, `.env`, `*.log`
- [x] README with architecture, commands, troubleshooting, viva
- [ ] Screenshots `screenshots/` folder mein add karo
- [ ] GitHub pe push karo aur repo link submit karo

---

### Credits

Original application: [dockersamples/example-voting-app](https://github.com/dockersamples/example-voting-app) (Apache 2.0 License — see `LICENSE`).
Is repo mein sirf PostgreSQL storage ko bind mount pe shift kiya gaya hai aur documentation add ki gayi hai.
