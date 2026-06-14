FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build

WORKDIR /src

COPY *.csproj ./

RUN dotnet restore

COPY . .

RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:7.0

WORKDIR /app

RUN useradd -r -s /bin/false appuser

COPY --from=build --chown=appuser:appuser /app/publish .

USER appuser

HEALTHCHECK --interval=15s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

ENTRYPOINT ["dotnet", "Worker.dll"]
