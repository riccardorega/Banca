# ==========================================
# Build Stage
# ==========================================
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# Copy solution and project files for layer caching
COPY Conti.sln ./
COPY Conti.Shared/Conti.Shared.csproj Conti.Shared/
COPY Conti.Repository/Conti.Repository.csproj Conti.Repository/
COPY Conti.Business/Conti.Business.csproj Conti.Business/
COPY Conti.ClientHttp/Conti.ClientHttp.csproj Conti.ClientHttp/
COPY Conti.Api/Conti.Api.csproj Conti.Api/

# Restore NuGet packages
RUN dotnet restore Conti.sln

# Copy all source code
COPY . .

# Build and publish Conti.Api
WORKDIR /src/Conti.Api
RUN dotnet publish -c Release -o /app/publish /p:UseAppHost=false

# ==========================================
# Runtime Stage
# ==========================================
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS runtime
WORKDIR /app

# Create non-root user for security
RUN adduser --disabled-password --gecos "" appuser && chown -R appuser /app
USER appuser

# Copy published application from build stage
COPY --from=build --chown=appuser:appuser /app/publish .

# Expose ports
EXPOSE 8080
EXPOSE 8081

ENV ASPNETCORE_URLS=http://+:8080

ENTRYPOINT ["dotnet", "Conti.Api.dll"]
