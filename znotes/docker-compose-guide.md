# Docker Compose — Complete Guide

যেকোনো project এ docker-compose.yml লেখার reference।

---

## Mental Model

**একটাই প্রশ্ন দিয়ে শুরু করো: "এই project চালাতে কি কি service লাগবে?"**

প্রতিটা service এর জন্য ৪টা প্রশ্ন:
```
১. image (ready) নাকি build (নিজের Dockerfile)?
২. কোন port এ চলে?
৩. কোন env variable লাগে? কার আগে চালু হবে?
৪. Data persist করতে হবে? (database হলে হ্যাঁ)
```

---

## Basic Structure

```yaml
services:

  service-name:
    image: nginx:alpine        # ready image ব্যবহার করলে
    # অথবা
    build:
      context: ./client        # Dockerfile কোন folder এ
      dockerfile: Dockerfile

    container_name: my-app     # optional, না দিলে auto নাম
    restart: unless-stopped    # crash হলে restart করো

    ports:
      - "HOST:CONTAINER"       # "80:80", "5000:5000"

    env_file:
      - .env                   # পুরো .env → container এ inject
    environment:
      - PORT=5000              # সরাসরি বা ${VAR} দিয়ে .env থেকে নাও

    depends_on:
      other-service:
        condition: service_healthy   # healthy হলে তারপর চালু হও

    volumes:
      - volume-name:/path/in/container

    networks:
      - app-network

    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:5000/api/health"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 15s

volumes:
  volume-name:          # named volume declare করো

networks:
  app-network:
    driver: bridge
```

---

## .env — দুইটা উপায়

```yaml
# উপায় ১ — পুরো file inject করো
env_file:
  - .env
# container এ process.env.PORT, process.env.MONGODB_URI সব পাবে

# উপায় ২ — compose file এর ভেতরে variable ব্যবহার করো
ports:
  - "${SERVER_PORT}:5000"   # .env এর SERVER_PORT নিচ্ছে
```

**.env → gitignore করো।**  
**.env.example → git এ রাখো** (template হিসেবে, value ছাড়া)।

---

## Volume — কেন লাগে, কিভাবে কাজ করে

```yaml
volumes:
  - mongo_data:/data/db
```

- Container এর `/data/db` folder এ MongoDB data রাখে
- `docker exec -it mongodb sh` দিয়ে ঢুকলে `ls /data/db` করলে দেখবে
- Container delete করলেও data থাকে (host এ `mongo_data` নামে save)
- কোন path দিতে হবে? → DockerHub এ সেই image এর page এ লেখা থাকে

---

## Network — কেন লাগে

```
Custom network না দিলেও কাজ করে:
  Docker Compose সব service কে <project>_default network এ রাখে
  service name দিয়ে একে অপরকে reach করা যায়

Custom network কেন দেয়?
  → নাম control করতে (পরিষ্কার)
  → কিছু service isolate করতে (DB শুধু server দেখবে, client না)
  → Multiple compose file connect করতে
```

**কেন সবসময় bridge?**
```
bridge  → host এ virtual network, local development এর জন্য
overlay → একাধিক machine (Docker Swarm, production cluster)
host    → container সরাসরি host network use করে (কম secure)
```

---

## MERN Stack Template

```yaml
services:

  mongodb:
    image: mongo:7
    container_name: mongodb
    restart: unless-stopped
    volumes:
      - mongo_data:/data/db
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  server:
    build:
      context: ./server
      dockerfile: Dockerfile
    container_name: server
    restart: unless-stopped
    ports:
      - "5000:5000"
    environment:
      - PORT=5000
      - MONGODB_URI=mongodb://mongodb:27017/mydb
    depends_on:
      mongodb:
        condition: service_healthy
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:5000/api/health"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 15s

  client:
    build:
      context: ./client
      dockerfile: Dockerfile
    container_name: client
    restart: unless-stopped
    ports:
      - "80:80"
    depends_on:
      server:
        condition: service_healthy
    networks:
      - app-network

volumes:
  mongo_data:

networks:
  app-network:
    driver: bridge
```

> MongoDB Atlas use করলে `mongodb` service বাদ দাও, `volumes` block বাদ দাও, `MONGODB_URI` তে Atlas URI দাও।

---

## Common Commands

```bash
# Build করে start করো
docker compose up --build

# Background এ run করো
docker compose up --build -d

# Logs দেখো
docker compose logs
docker compose logs server       # নির্দিষ্ট service এর

# Container এর ভেতরে ঢোকো
docker exec -it server sh
docker exec -it mongodb sh

# চলছে কিনা দেখো
docker compose ps

# বন্ধ করো
docker compose down

# Data সহ বন্ধ করো (volume delete)
docker compose down -v
```

---

## Key Rules

| Rule | কারণ |
|---|---|
| `depends_on: condition: service_healthy` দাও | শুধু "started" না, সত্যিই ready হলে পরের service উঠবে |
| Database এ সবসময় volume দাও | Container restart এ data হারাবে না |
| `restart: unless-stopped` দাও | Crash এ auto recover |
| `.env` gitignore করো | Secrets public হবে না |
| MONGODB_URI তে service name দাও | `mongodb://mongodb:27017` — localhost না |
