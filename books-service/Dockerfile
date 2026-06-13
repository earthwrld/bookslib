FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS base
WORKDIR /src
COPY ["BooksLib.csproj", "./"]
COPY ["BooksLib.Tests/BooksLib.Tests.csproj", "BooksLib.Tests/"]
RUN dotnet restore "BooksLib.csproj"
RUN dotnet restore "BooksLib.Tests/BooksLib.Tests.csproj"
COPY . .

FROM base AS build
WORKDIR /src
RUN dotnet publish "BooksLib.csproj" -c Release -o /app/publish --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS production
WORKDIR /app
COPY --from=build /app/publish .
USER $APP_UID
EXPOSE 8080
ENTRYPOINT ["dotnet", "BooksLib.dll"]