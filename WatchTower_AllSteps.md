
# Guidet øvelse: Hello World C# med Docker, Healthcheck, WatchTower og GitHub Actions

Denne øvelse fører dig trin-for-trin fra et simpelt C# Hello World-projekt til automatiseret container-opdatering med WatchTower og GitHub Actions. Alle trin er skrevet som sammenhængende tekst med kodeblokke, så du kan kopiere filen direkte til GitHub.

## Trin 1: Opret et nyt C# Hello World-projekt

Formålet er at etablere et minimalt projekt, som resten af forløbet bygger på.

```sh
dotnet new console -n HelloWorld
cd HelloWorld
dotnet run
```
Du skal se "Hello, World!" i terminalen. Hvis det ikke virker, dobbelttjek at du står i den rigtige mappe.

## Trin 2: Multistage Dockerfile

Vi containeriserer appen med en effektiv multistage Dockerfile, så der bygges i ét image og køres i et andet, mindre runtime-image.

**Opret filen `Dockerfile` i projektmappen** og indsæt:

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

# Healthcheck – for console-app bruger vi en enkel kommando
HEALTHCHECK --interval=30s --timeout=3s --retries=3 CMD [ "dotnet", "HelloWorld.dll" ]

ENTRYPOINT ["dotnet", "HelloWorld.dll"]
```

Byg og kør containeren lokalt:

```sh
docker build -t helloworld:latest .
docker run --rm helloworld:latest
```
Du skal igen se "Hello, World!" – nu fra din container.

## Trin 3: HEALTHCHECK

Formålet er at gøre det muligt for Docker og WatchTower at vurdere, om containeren er sund. I denne øvelse bruger vi en simpel kommando, der starter programmet og afslutter hurtigt, hvilket er tilstrækkeligt for en console-app. I webapps ville du normalt kalde et HTTP-endpoint med `curl`.

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 CMD [ "dotnet", "HelloWorld.dll" ]
```

## Trin 4: WatchTower med Docker Compose

Vi sætter WatchTower op til at overvåge og automatisk opdatere **kun** de services, der er mærket med den relevante label.

**Opret filen `docker-compose.yml` i driftmappen** (samme host som kører containere):

```yaml
version: "3.8"
services:
  helloworld:
    image: ghcr.io/<owner>/helloworld:latest
    restart: unless-stopped
    labels:
      com.centurylinklabs.watchtower.enable: "true"
    healthcheck:
      test: ["CMD", "dotnet", "HelloWorld.dll"]
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

Start stakken:

```sh
docker compose up -d
```

> **Bemærk**: Hvis dit GHCR-image er privat, skal du logge ind på hosten: `docker login ghcr.io` med en Personal Access Token, eller gøre pakken public til undervisning.

## Trin 5: GitHub Actions – automatisk build & push

Formålet er, at hvert `push` til `main` automatisk bygger et Docker-image og uploader det til GHCR. Det gør det muligt for WatchTower at opdage en ny digest.

Opret workflow-mappen og filen:

```sh
mkdir -p .github/workflows
```

**Opret `.github/workflows/docker-publish.yml`** med indholdet:

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
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ghcr.io/${{ github.repository_owner }}/helloworld:latest
```

Commit og push workflowet:

```sh
git add .
git commit -m "Add GH Actions build & push"
git branch -M main
# hvis ikke sat remote:
# git remote add origin <din-github-url>
git push -u origin main
```

Tjek under **Actions** i GitHub, at workflowet kører grønt. Dit image findes under **Packages** i repoet.

Typiske fejl og løsninger:

- *No permission to push image*: brug `${{ secrets.GITHUB_TOKEN }}` og tjek repo-/package-synlighed.
- *Cannot find Dockerfile*: sørg for korrekt sti i `file: ./Dockerfile`.
- *Workflow kører ikke*: pusher du til `main`? Ellers justér `on.push.branches`.

## Trin 6: Verificér WatchTower med en kodeændring (ingen Compose-ændring)

Pointen er, at du **kun** ændrer koden, pusher, og ser at WatchTower udskifter containeren automatisk.

Ret din `Program.cs` (fx beskeden):

```csharp
Console.WriteLine("Hello, WatchTower!");
```

Commit og push ændringen:

```sh
git add Program.cs
git commit -m "Change output to Hello, WatchTower!"
git push
```

Vent til GitHub Actions-workflowet er færdigt. WatchTower vil ved næste poll opdage ny digest i GHCR, pull’e image’et og genstarte `helloworld` med samme Compose-parametre.

Verificér på drift:

```sh
docker logs <helloworld-container>
```

Du bør nu se den nye tekst “Hello, WatchTower!”. Ingen Compose-ændringer, ingen manuelle genstarter.

---

## Ekstra udfordring (valgfri)

Gentag øvelsen med et projekt fra 2. semester: Tilføj passende HEALTHCHECK (fx HTTP endpoint i webapp), brug labels/opt-in i WatchTower, og vis jeres løsning i næste undervisning.

## Refleksionsspørgsmål

1. Hvilke konkrete fordele gav multistage (størrelse, sikkerhed, hastighed)?
2. Hvorfor er en deterministisk, hurtig healthcheck passende for en console-app?
3. Hvad gør WatchTower, når der er ny digest, og hvorfor er “samme Compose-parametre” vigtigt?
4. Hvilke dele af workflowet er kritiske for, at image ender i GHCR (login, tags, file, context)?
5. Hvor bruger du repo-URL (kildekode) vs. registry-URL (images), og hvorfor er forveksling et problem?
6. Hvilke fejl kan opstå (permissions, Dockerfile-sti, branch), og hvordan løser I dem?
7. Hvordan ville I ændre setup’et for et kritisk system (labels/opt-in, planlagte vinduer, pinning)?
8. Hvordan standardiserer I workflows i større teams (code owners, reviews, version-tags)?
9. Hvilke ændringer kræver et skifte til Docker Hub (login, tags, synlighed)?

