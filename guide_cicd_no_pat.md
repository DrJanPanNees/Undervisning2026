# Guidet øvelse: Fra C# til CI/CD med Docker, Healthcheck og Watchtower

Denne guide er optimeret til at fjerne de typiske adgangsproblemer (PAT) ved at bruge GitHubs indbyggede rettigheder.

---

## 🔑 Forstå de to forskellige adgangskrav

1.  **Din PC → GitHub:** Her bruger du **SSH** (hvis du får `publickey`-fejl) eller **HTTPS**. Dette er kun for at `git push` din kode.
2.  **GitHub Actions → GHCR (Registry):** Her bruges **ingen PAT**. Vi bruger det indbyggede `GITHUB_TOKEN` via `permissions` i din YAML-fil.

---

## DEL 1: Udviklingsmaskinen (Fabrikken)

### 1. Opret Web-app
Vi bruger en Web-app, da det gør det let at teste Healthchecks.
```sh
dotnet new web -n HelloWorld
cd HelloWorld
```

I `Program.cs`, tilføj et simpelt endpoint og en healthcheck:
```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello from CI/CD Pipeline!");
app.MapGet("/healthz", () => Results.Ok("Healthy")); 

app.Run();
```

### 2. Dockerfile
Opret en `Dockerfile` i projektmappen:
```dockerfile
# Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /out

# Runtime
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /out ./
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080

# Healthcheck
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -fsS http://localhost:8080/healthz || exit 1

ENTRYPOINT ["dotnet", "HelloWorld.dll"]
```

### 3. GitHub Actions (Løsningen på PAT-problemet)
Opret `.github/workflows/docker-publish.yml` i dit repo. Ved at tilføje `permissions` nedenfor, giver du GitHub lov til at skrive til dit registry uden en PAT.

```yaml
name: Build and Push Docker image
on:
  push:
    branches: [ "main", "master" ]

permissions:
  contents: read
  packages: write  # Dette gør manuel PAT overflødig!

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v5

      - name: Set lowercase owner
        run: echo "OWNER_LC=${GITHUB_REPOSITORY_OWNER@L}" >> $GITHUB_ENV

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build & Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ env.OWNER_LC }}/helloworld:latest
```
### 4. Opret jeres Github repo på www.github.com
1. Opret et repo på github.com det giver dig en sti der ser noget ala: https://github.com/DrJanPanNees/HelloWatchTower2.git
2. Du skal kende dit Brugerhavn
3. Du skal oprette en kode til github: Settings -> SSH keys -> New SSH key (gem denne nøgle et sikkert sted)
```
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/DrJanPanNees/HelloWatchTower2.git
git push -u origin main

Indtast brugernavn med små, password er det underlige i fik med SSH key'en.
---
```
## DEL 2: Produktionsmaskinen (Udstillingsvinduet)

### 4. Gør pakken Public
GitHub pakker er private som standard. For at din server kan hente dem uden login, skal de være offentlige:
1. Gå til dit repo på GitHub -> **Packages**.
2. Vælg din pakke (`helloworld`).
3. Gå til **Package Settings**.
4. Scroll ned til "Danger Zone" og vælg **Change visibility** -> **Public**.

### 5. Docker Compose
På din server (eller lokale test-maskine), kør denne `docker-compose.yml`:

```yaml
services:
  app:
    image: ghcr.io/<dit-brugernavn-i-små-bogstaver>/helloworld:latest
    ports:
      - "8080:8080"
    labels:
      com.centurylinklabs.watchtower.enable: "true"

  watchtower:
    image: containrrr/watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      WATCHTOWER_POLL_INTERVAL: 60
      WATCHTOWER_CLEANUP: "true"
    command: --label-enable
```

---

## 🛠 Fejlfinding
* **Permission denied (publickey):** Din computer mangler at få sin SSH-nøgle tilføjet til GitHub Settings.
* **Workflow fejler ved push:** Tjek at din YAML har `packages: write` under permissions.
* **Watchtower henter ikke nyt:** Tjek at du har sat din Package til "Public" inde på GitHub.
