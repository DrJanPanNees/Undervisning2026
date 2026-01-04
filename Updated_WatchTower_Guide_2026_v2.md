
# Guidet øvelse (opdateret 2026): C# Hello World → Docker → GHCR → Watchtower (med GitHub Actions)

Denne version er rettet, så studerende undgår de fejl, vi oplevede:
- **Brugernavn skal være `lowercase`** i alle registry-tags (GHCR).
- **GitHub Actions kræver PAT med `workflow`-scope** første gang du pusher workflows.
- **Multi-arkitektur (amd64 + arm64)** bygges via QEMU/Buildx, så Apple Silicon kan køre images.
- **Watchtower kræver korrekt label**: `com.centurylinklabs.watchtower.enable: "true"`.
- **GHCR-pakker er private som standard** – gør pakken public til undervisning, eller log ind i Docker.
- **Compose `version:`-felt er forældet** – udelades for at undgå advarsler.

---
## Forudsætninger (gør dette først)
1. **.NET SDK 8**, **Docker Desktop** og **Git** installeret.
2. **GitHub-konto** og et **privat eller offentligt repo** (fx `helloworld`).
3. **Personal Access Token (PAT, classic)** til **første push med workflows**:
   - GitHub → Settings → *Developer settings* → *Personal access tokens* → *Tokens (classic)* → *Generate*.
   - Scopes: `repo`, **`workflow`** (vigtigt!).
   - Kopiér token straks.

> **Hvorfor?** GitHub afviser push af filer i `.github/workflows/` uden `workflow`-scope (fejl: *refusing to allow a Personal Access Token ... without workflow scope*).

---
## Trin 1 – Hello World
```sh
mkdir HelloWorld && cd HelloWorld
 dotnet new console
 dotnet run
```
Du skal se `Hello, World!`.

> **Tip**: Hold programmet kørende eller tilføj timer/sleep i undervisningsscenarier, men til denne øvelse bruger vi Watchtower + restart-policy.

---
## Trin 2 – Dockerfile (multistage)
Opret `Dockerfile` i projektroden:
```dockerfile
# Stage 1: build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /out

# Stage 2: runtime
FROM mcr.microsoft.com/dotnet/runtime:8.0 AS runtime
WORKDIR /app
COPY --from=build /out ./
# Simpel healthcheck for demo (console-app)
HEALTHCHECK --interval=30s --timeout=3s --retries=3 CMD ["dotnet","HelloWorld.dll"]
ENTRYPOINT ["dotnet","HelloWorld.dll"]
```
Test lokalt:
```sh
docker build -t helloworld:local .
docker run --rm helloworld:local
```

---
## Trin 3 – GitHub Actions → Byg & push til GHCR (amd64+arm64)
Opret mappen og workflow-filen:
```sh
mkdir -p .github/workflows
```
`.github/workflows/docker-publish.yml` (brug **lowercase** brugernavn i tags – her erstattes `<owner_lc>` af dit eget GitHub-brugernavn i små bogstaver):
```yaml
name: Build and Push Docker image
on:
  push:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up QEMU (multi-arch)
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push (amd64+arm64)
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          platforms: linux/amd64,linux/arm64
          tags: ghcr.io/<owner_lc>/helloworld:latest
```

**Vigtigt ved første `git push`:** Brug din PAT med `workflow`-scope, når Git beder om Password. Alternativt opdater origin-URL midlertidigt:
```sh
git init
 git add .
 git commit -m "init: console app + docker + gha"
 git branch -M main
 git remote add origin https://github.com/<OwnerCase>/helloworld.git

# Første push – Git vil bede om bruger+password
 git push -u origin main
```

> **Efter success**: Gå til repoets **Actions**-fane. Vent til workflowet er grønt. Gå derefter til **Packages** (linket ved repoets top). Hvis nødvendigt: **Package settings → Change visibility → Make public** (for undervisning).

**Kontrol af arkitekturer i pakken**:
- På packagesiden skal OS/Arch vise **Linux/amd64** og **Linux/arm64** (ikke `unknown/unknown`).

---
## Trin 4 – docker-compose + Watchtower
Opret `docker-compose.yml` (uden `version:`):
```yaml
services:
  helloworld:
    image: ghcr.io/<owner_lc>/helloworld:latest
    # Valgfrit på Apple Silicon for tydelighed
    # platform: linux/arm64
    restart: unless-stopped
    labels:
      com.centurylinklabs.watchtower.enable: "true"
    healthcheck:
      test: ["CMD","dotnet","HelloWorld.dll"]
      interval: 30s
      timeout: 3s
      retries: 3

  watchtower:
    image: containrrr/watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      WATCHTOWER_POLL_INTERVAL: 1800
      WATCHTOWER_CLEANUP: "true"
    command: --label-enable
```
> **Erstat `<owner_lc>` med dit GitHub-brugernavn i **små bogstaver**. Fejl som *invalid reference format ... must be lowercase* skyldes store bogstaver.

Start stacken:
```sh
docker compose up -d
```
Hvis pakken er privat, skal du logge ind på GHCR på hosten først:
```sh
docker login ghcr.io
# Username: <owner_lc>
# Password: PAT (kan være repo-scoped; til pull kan fine-grained bruges)
```

---
## Trin 5 – Bekræft pipeline → ændr kode
Lav en synlig kodeændring i `Program.cs`:
```csharp
Console.WriteLine("Hello, WatchTower!");
```
Commit & push:
```sh
git add Program.cs
 git commit -m "feat: visible change for watchtower"
 git push
```
Tjek **Actions** til den er grøn. På serveren vil Watchtower ved næste poll hente ny digest og genstarte `helloworld` med samme Compose-parametre.

**Logs**:
```sh
docker logs $(docker ps | grep watchtower | awk '{print $1}')
# Du bør se: Scanned=1 (ikke 0)
```

---
## Typiske fejl og hurtige løsninger
- **PAT mangler `workflow`** → Opret ny PAT (classic) med `workflow`; push igen.
- **Repo-tag med store bogstaver** → Brug `ghcr.io/<owner_lc>/...` i *små bogstaver* overalt (workflow + compose).
- **Kun amd64 bygget** → Sørg for `setup-qemu-action` + `platforms: linux/amd64,linux/arm64` i workflow.
- **OS/Arch viser `unknown/unknown`** → Slet defekt package-version (Package settings), kør ny build fra Actions.
- **Watchtower finder ingen containere (Scanned=0)** → Mangler label eller service kører ikke. Brug Compose og label `com.centurylinklabs.watchtower.enable: "true"`.
- **Compose-warning om `version:`** → Fjern `version:`-feltet.
- **GHCR 404 eller pull denied** → Pakke findes ikke endnu eller er privat. Vent til workflow er grønt, eller gør pakken public / log ind.

---
## Appendix – Minimal checkliste (undervisning)
1. `<owner_lc>` konsekvent i **små bogstaver** i tags og compose.
2. Actions workflow: QEMU + Buildx + `platforms` sæt.
3. Første push med PAT (`workflow`-scope) hvis promptet.
4. Pakke gjort **public** (undervisning), ellers `docker login ghcr.io` på hosten.
5. Compose uden `version:`, med Watchtower-**label** og `restart: unless-stopped`.
6. Logs viser `Scanned=1` og opdatering ved ny digest.

---

## Ekstra udfordring (valgfri)

Gentag øvelsen med et projekt fra 2. semester: Tilføj passende HEALTHCHECK (fx HTTP endpoint i webapp), brug labels/opt-in i WatchTower, og vis jeres løsning i næste undervisning.



