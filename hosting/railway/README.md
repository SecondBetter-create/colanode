# Colanode op Railway

Deze pagina bevat de korte route om Colanode op Railway te starten. De eenvoudigste configuratie gebruikt PostgreSQL met pgvector, Valkey, de server en de webapp. MinIO is optioneel.

## Snelkoppelingen

- [Railway-dashboard](https://railway.com/dashboard)
- [Railway-variabelen](https://docs.railway.com/guides/variables)
- [Railway private networking](https://docs.railway.com/guides/private-networking)
- [Railway volumes](https://docs.railway.com/guides/volumes)
- [Docker Compose-configuratie in deze repository](../docker/docker-compose.yaml)

## Wachtwoorden veilig maken

Er staan bewust geen Railway-wachtwoorden of standaard inloggegevens in deze repository. Gebruik voor zowel PostgreSQL als Valkey een eigen willekeurig wachtwoord. Dit PowerShell-commando maakt een URL-veilige hex-wachtwoordwaarde en kopieert die naar het klembord:

```powershell
$bytes = [byte[]]::new(32)
[System.Security.Cryptography.RandomNumberGenerator]::Fill($bytes)
$password = [Convert]::ToHexString($bytes).ToLowerInvariant()
Set-Clipboard $password
Write-Host "Een willekeurig wachtwoord staat op het klembord."
```

Voer het commando apart uit voor PostgreSQL en Valkey. Plak de eerste waarde bij `POSTGRES_PASSWORD` op de Postgres-service en de tweede bij `REDIS_PASSWORD` op Valkey. Railway bewaart deze als servicevariabelen; commit of deel de waarden niet.

## Services instellen

Maak alle services aan binnen hetzelfde Railway-project. Gebruik exact deze servicenames, omdat de variabelen hieronder ernaar verwijzen.

### 1. `postgres`

- Image: `pgvector/pgvector:pg17`
- Variabelen:

  ```text
  POSTGRES_DB=colanode_db
  POSTGRES_USER=colanode_user
  POSTGRES_PASSWORD=<plak het gegenereerde PostgreSQL-wachtwoord>
  ```

- Voeg een volume toe op `/var/lib/postgresql/data`.
- Maak geen public domain aan.

### 2. `valkey`

- Image: `valkey/valkey:8.1`
- Variabele:

  ```text
  REDIS_PASSWORD=<plak het gegenereerde Valkey-wachtwoord>
  ```

- Start command:

  ```sh
  sh -c 'exec valkey-server --requirepass "$REDIS_PASSWORD"'
  ```

- Maak geen public domain aan. Een volume op `/data` is aanbevolen als je cachedata wilt behouden.

### 3. `server`

- Source: deze GitHub-repository.
- Root directory: `/` (de repository-root).
- Dockerfile: `apps/server/Dockerfile`.
- Voeg een volume toe op `/data`.
- Maak een public domain aan nadat de eerste deploy is geslaagd.
- Zet deze variabelen:

  ```text
  NODE_ENV=production
  CONFIG=/app/apps/server/config.railway.json
  POSTGRES_URL=postgres://colanode_user:${{postgres.POSTGRES_PASSWORD}}@${{postgres.RAILWAY_PRIVATE_DOMAIN}}:5432/colanode_db
  REDIS_URL=redis://:${{valkey.REDIS_PASSWORD}}@${{valkey.RAILWAY_PRIVATE_DOMAIN}}:6379/0
  WEB_DOMAIN=${{web.RAILWAY_PUBLIC_DOMAIN}}
  WEB_ORIGIN=https://${{web.RAILWAY_PUBLIC_DOMAIN}}
  ```

Railway vult `PORT` zelf in; stel die hier niet handmatig in. De server luistert op die runtime-poort.

### 4. `web`

- Source: deze GitHub-repository.
- Root directory: `/` (de repository-root).
- Dockerfile: `apps/web/Dockerfile`.
- Maak een public domain aan.
- Extra variabelen zijn niet nodig voor de standaardconfiguratie.

De variabelereferenties zoals `${{postgres.POSTGRES_PASSWORD}}` en `${{web.RAILWAY_PUBLIC_DOMAIN}}` worden door Railway ingevuld. Zet de services in hetzelfde Railway-project en behoud de namen hierboven.

## Na de deploy

1. Controleer dat `postgres` en `valkey` intern bereikbaar zijn en geen public domain hebben.
2. Wacht tot zowel `server` als `web` succesvol zijn gedeployed.
3. Open de publieke URL van `server` en noteer het domein. De serverconfiguratie is bereikbaar op `https://<server-domein>/config`.
4. Open de publieke URL van `web` en voeg in Colanode de server toe met die configuratie-URL.

Er zijn geen vooraf ingestelde Colanode-accountgegevens. Maak je account aan via de aanmeld-/registratieflow van de app. De Railway-databasegebruiker hierboven is alleen voor de serverdatabase, niet om in Colanode in te loggen.

## Optioneel: MinIO

Voeg dit alleen toe als je objectopslag wilt gebruiken in plaats van de standaard bestandopslag.

- Image: `minio/minio:RELEASE.2025-04-08T15-41-24Z`
- Start command:

  ```sh
  minio server /data --address ":9000" --console-address ":9001"
  ```

- Stel een gebruikersnaam in en maak met het PowerShell-commando hierboven een willekeurig wachtwoord:

  ```text
  MINIO_ROOT_USER=colanode_minio
  MINIO_ROOT_PASSWORD=<gegenereerd MinIO-wachtwoord>
  ```

- Voeg een volume toe op `/data`; maak geen public domain aan voor de MinIO API.
- Maak in MinIO een bucket met de naam `colanode`.
- Vervang op `server` `CONFIG` door `/app/apps/server/config.railway.s3.json` en voeg toe:

  ```text
  S3_ENDPOINT=http://${{minio.RAILWAY_PRIVATE_DOMAIN}}:9000
  S3_ACCESS_KEY=${{minio.MINIO_ROOT_USER}}
  S3_SECRET_KEY=${{minio.MINIO_ROOT_PASSWORD}}
  S3_BUCKET=colanode
  S3_REGION=us-east-1
  ```
