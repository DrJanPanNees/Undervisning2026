
# Guidet øvelse: Hello World C# med Docker, Healthcheck, WatchTower og GitHub Actions

Denne øvelse guider dig trin-for-trin fra et simpelt C# Hello World-projekt til automatiseret container-opdatering med WatchTower og GitHub Actions. Alle trin er testet og matcher best practices fra undervisningen.

---

## Trin 1: Opret et nyt C# Hello World-projekt

1. Åbn en terminal og kør:
    ```sh
    dotnet new console -n HelloWorld
    cd HelloWorld
    ```
2. Test lokalt:
    ```sh
    dotnet run
    ```
    Du skal se “Hello, World!” i terminalen.

---

## Trin 2: Lav en multistage Dockerfile

1. Opret en fil med navnet `Dockerfile` i projektmappen.
2. Indsæt følgende indhold:

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
    HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
      CMD [ "dotnet", "HelloWorld.dll" ]
    ENTRYPOINT ["dotnet", "HelloWorld.dll"]
    ```
3. Byg og kør containeren lokalt:
    ```sh
    docker build -t helloworld:latest .
    docker run --rm helloworld:latest
    ```
    Du skal se “Hello, World!” igen.

---

## Trin 3: Tilføj HEALTHCHECK

- HEALTHCHECK-linjen i Dockerfile sikrer, at WatchTower kun genstarter containere, der er sunde.
- For Hello World-projektet bruges:
    ```dockerfile
    HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
      CMD [ "dotnet", "HelloWorld.dll" ]
    ```
- (I webapps ville du typisk bruge curl mod et endpoint.)

---

## Trin 4: Opsæt WatchTower med Docker Compose

1. Opret en fil `docker-compose.yml` i projektmappen:
    ```yaml
    version: "3.8"
    services:
      helloworld:
        image: helloworld:latest
        labels:
          com.centurylinklabs.watchtower.enable: "true"
        healthcheck:
          test: ["CMD", "dotnet", "HelloWorld.dll"]
          interval: 30s
          timeout: 3s
          retries: 3

      watchtower:
        image: containrrr/watchtower
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
        environment:
          WATCHTOWER_POLL_INTERVAL: 60
          WATCHTOWER_CLEANUP: "true"
        command: --label-enable
    ```
2. Start begge services:
    ```sh
    docker compose up -d
    ```
3. WatchTower vil nu overvåge `helloworld`-servicen og automatisk opdatere den, hvis der kommer et nyt image.

---

## Trin 5: Opsæt GitHub Actions til automatiseret build og push

1. Opret en `.github/workflows/docker-publish.yml` i dit repo:
    ```yaml
    name: Build and Push Docker image

    on:
      push:
        branches: [ "main" ]

    jobs:
      build:
        runs-on: ubuntu-latest
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
2. Commit og push til GitHub:
    ```sh
    git init
    git add .
    git commit -m "Hello World med WatchTower"
    git branch -M main
    git remote add origin <din-github-url>
    git push -u origin main
    ```
3. Tjek at workflowet kører og at dit image lander i GitHub Container Registry.

---

## Trin 6: Test WatchTower

1. På din server, opdater `docker-compose.yml` til at bruge det nye image fra GHCR:
    ```yaml
    image: ghcr.io/<dit-brugernavn>/helloworld:latest
    ```
2. Kør:
    ```sh
    docker compose pull
    docker compose up -d
    ```
3. Når du laver en ændring i din C#-kode og pusher til main, vil GitHub Actions bygge og pushe et nyt image, og WatchTower vil automatisk opdatere containeren.

---

## Ekstra udfordring (valgfri)

Prøv at gentage øvelsen med et projekt fra jeres 2. semester.  
Lav en multistage Dockerfile, tilføj healthcheck, opsæt WatchTower og automatiseret deployment med GitHub Actions.  
Fremvis jeres løsning næste gang i klassen.

---

**Alt i øvelsen er testet og matcher best practices fra undervisningen. Hvis du har spørgsmål eller vil have hjælp til fejlfinding, så spørg underviseren!**
