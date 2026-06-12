FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

WORKDIR /src

COPY *.csproj ./

RUN dotnet restore

COPY . .

RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0

WORKDIR /app

RUN useradd -r -s /bin/false appuser

COPY --from=build --chown=appuser:appuser /app/publish .

USER appuser

ENTRYPOINT ["dotnet", "MyApp.dll"]