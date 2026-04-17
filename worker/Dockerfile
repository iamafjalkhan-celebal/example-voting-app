# syntax=docker/dockerfile:1

# === Build stage ===
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS builder

WORKDIR /app

# Copy csproj and restore as distinct layers
COPY Worker.csproj ./
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet restore

# Copy everything else and build
COPY . .
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet publish -c Release -o /app/out

# === Final stage ===
FROM mcr.microsoft.com/dotnet/runtime:7.0

RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app

COPY --from=builder --chown=appuser:appgroup /app/out .

USER appuser

ENTRYPOINT ["dotnet", "Worker.dll"]
