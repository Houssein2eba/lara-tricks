# Laravel Sail with Docker Desktop and PowerShell

This project uses Laravel Sail's Docker configuration. Sail's `vendor/bin/sail`
launcher is a Bash script, so use Docker Compose directly from PowerShell on
Windows. Docker Desktop may use its own WSL 2 backend; a separate Ubuntu WSL
distro is not required.

## One-time setup

1. Install and start Docker Desktop.
2. From PowerShell, check that the Docker engine responds:

   ```powershell
   docker info
   docker compose version
   ```

   `docker info` should include a **Server** section. `docker --version` alone
   only confirms that the Docker CLI is installed.

## Run this project from PowerShell

Open PowerShell in the project root (the directory containing `compose.yaml`).
Start the application and database:

```powershell
docker compose up -d --build
docker compose ps
```

The Compose file defaults `WWWGROUP` and `WWWUSER` to `1000` when they are not
set in `.env`, preventing an empty group ID from breaking the image build.

Run Composer, Artisan, and frontend commands inside the app container. This
avoids needing PHP or Node.js installed on Windows:

```powershell
docker compose exec laravel.test composer install
docker compose exec laravel.test php artisan migrate
docker compose exec laravel.test npm install
docker compose exec laravel.test npm run build
```

The app image includes Node.js and npm. `docker compose exec` requires the
`laravel.test` container to be running first.

Stop and remove this project's containers and network when finished:

```powershell
docker compose down
```

This keeps named database volumes. Do not add `--volumes` unless you intend to
delete the database data too.

## Add Sail to a future Laravel project

Create/open the project from PowerShell, with PHP and Composer available on
Windows, then from its root:

```powershell
composer require laravel/sail --dev --no-scripts
php artisan sail:install
docker compose up -d --build
docker compose exec laravel.test composer install
docker compose exec laravel.test npm install
docker compose exec laravel.test npm run build
```

Choose the services the project needs during `sail:install`. The `--no-scripts`
option avoids running Composer hooks on Windows before the container is
available. Run Composer and npm commands with `docker compose exec` afterward,
so they use the PHP, Composer, Node.js, and npm installed in the container.

## Avoid the setup problems from this project

- Do not run `vendor/bin/sail` from PowerShell; it requires Bash. Use the
  `docker compose` commands above instead.
- If `docker info` cannot reach the server, start Docker Desktop and wait for
  its engine to finish starting before retrying.
- Node.js inside the app container is not automatically available on Windows.
  Run `npm` through `docker compose exec laravel.test npm ...`.
- If a Composer hook tries to run npm on Windows, run Composer in the container
  (for example, `docker compose exec laravel.test composer update`).
- Run Compose commands from the project root so it finds `compose.yaml` and
  `.env`.

## Reduce memory use

- Stop this project's containers when you are finished. This preserves named
  database volumes:

  ```powershell
  docker compose down
  ```

- To see which containers are using memory:

  ```powershell
  docker stats --no-stream
  ```

- Docker Desktop can use the WSL 2 backend even when you do not have an Ubuntu
  distro. To cap memory available to that backend, create or edit
  `%UserProfile%\.wslconfig` in Windows. Choose a limit that leaves enough RAM
  for Windows and other apps; for example:

  ```ini
  [wsl2]
  memory=4GB
  swap=2GB
  ```

  Apply changes by running `wsl --shutdown` from PowerShell and restarting
  Docker Desktop. This affects the Docker Desktop WSL 2 backend and any other
  WSL 2 distros, so avoid setting the limit lower than your projects need.
