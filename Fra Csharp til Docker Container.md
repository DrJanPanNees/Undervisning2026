# Øvelse: Simpelt C# Konsolscript i Docker

Denne øvelse guider dig igennem at bygge og køre et enkelt C#-konsolprogram i en Docker-container. Øvelsen bruger multi‑stage builds og genererer projektstrukturen direkte i containeren.

## Mål
- Køre et C#-script i en container uden at have .NET installeret lokalt
- Forstå forskellen på SDK- og Runtime-images
- Bygge deterministiske multi-stage Docker images
- Håndtere argumenter og miljøvariable i et .NET‑program

## 1) Mappestruktur
```
ScriptAppDocker/
  ├─ Dockerfile
  └─ Program.cs
```

## 2) Program.cs
```csharp
using System.Globalization;

static (string name, int count) ParseArgs(string[] args)
{
    string name = "Student";
    int count = 3;

    for (int i = 0; i < args.Length; i++)
    {
        if (args[i] == "--name" && i + 1 < args.Length)
            name = args[i + 1];
        if (args[i] == "--count" && i + 1 < args.Length && int.TryParse(args[i + 1], out var c))
            count = c;
    }
    return (name, count);
}

var (nameArg, countArg) = ParseArgs(args);
var appEnv = Environment.GetEnvironmentVariable("APP_ENV") ?? "local";

var numbers = Enumerable.Range(1, Math.Max(1, countArg)).ToArray();
var sum = numbers.Sum();
var avg = numbers.Average();

Console.WriteLine("=== ScriptApp Report ===");
Console.WriteLine($"Name      : {nameArg}");
Console.WriteLine($"Count     : {numbers.Length}");
Console.WriteLine($"Sum       : {sum}");
Console.WriteLine($"Average   : {avg.ToString("0.00", CultureInfo.InvariantCulture)}");
Console.WriteLine($"APP_ENV   : {appEnv}");
Console.WriteLine("Numbers   : " + string.Join(",", numbers));
Console.WriteLine("========================");
```

## 3) Dockerfile
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /app

COPY Program.cs .

RUN dotnet new console -n ScriptApp --output ./out     && mv Program.cs ./out/Program.cs     && cd ./out     && dotnet build -c Release

FROM mcr.microsoft.com/dotnet/runtime:6.0 AS runtime
WORKDIR /app

COPY --from=build /app/out/bin/Release/net6.0/ ./

ENTRYPOINT ["dotnet", "ScriptApp.dll"]
```

## 4) Build og kør
```
docker build -t ucl/scriptapp:1.0 .
docker run --rm ucl/scriptapp:1.0
docker run --rm ucl/scriptapp:1.0 --name "Aurelia" --count 5
docker run --rm -e APP_ENV=production ucl/scriptapp:1.0
```
