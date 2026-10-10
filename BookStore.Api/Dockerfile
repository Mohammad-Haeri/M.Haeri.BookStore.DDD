# See https://aka.ms/customizecontainer to learn how to customize your debug container and how Visual Studio uses this Dockerfile to build your images for faster debugging.

# This stage is used when running from VS in fast mode (Default for Debug configuration)
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
USER $APP_UID
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

# Restore dependencies before copying application source.
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src

COPY ["Directory.Build.props", "./"]
COPY ["BookStore.Api/M.Haeri.BookStore.Api.csproj", "BookStore.Api/"]
COPY ["BookStore.Application/M.Haeri.BookStore.Application.csproj", "BookStore.Application/"]
COPY ["BookStore.Domain/M.Haeri.BookStore.Domain.csproj", "BookStore.Domain/"]
COPY ["BookStore.Infrastructure/M.Haeri.BookStore.Infrastructure.csproj", "BookStore.Infrastructure/"]

RUN dotnet restore "BookStore.Api/M.Haeri.BookStore.Api.csproj" \
    -p:Configuration="$BUILD_CONFIGURATION" \
    -p:UseAppHost=false

COPY . .
WORKDIR /src/BookStore.Api

# Compile and publish without restoring dependencies again.
FROM build AS publish
RUN dotnet publish "M.Haeri.BookStore.Api.csproj" \
    -c "$BUILD_CONFIGURATION" \
    -o /app/publish \
    --no-restore \
    -p:UseAppHost=false

# Run the published application without the SDK.
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "M.Haeri.BookStore.Api.dll"]
