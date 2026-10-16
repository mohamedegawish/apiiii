# Laravel Shared Hosting Deployment

## Current Check

`https://test.talentooo.com/up` currently returns the Hostinger 404 page. The subdomain is not pointing to Laravel's `api/public` directory yet.

## 1. Upload And Document Root

Use this layout:

```text
/home/USERNAME/lms-api/
  app/
  bootstrap/
  config/
  database/
  routes/
  storage/
  vendor/
  .env
  artisan
  public/
    index.php
```

Set the document root of `test.talentooo.com` to:

```text
/home/USERNAME/lms-api/public
```

Do not point the domain to the Laravel project root. Do not expose `.env` or `vendor` through the public document root.

## 2. Install Laravel

Run over SSH from the Laravel project directory:

```bash
cd /home/USERNAME/lms-api
which php
php -v
which composer
composer install --no-dev --prefer-dist --optimize-autoloader
cp .env.example .env
php artisan key:generate --force
php artisan migrate --force
php artisan storage:link
chmod -R 775 storage bootstrap/cache
php artisan optimize:clear
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

If the host has multiple PHP binaries, use the PHP 8.2 binary shown by the hosting panel, for example `/opt/cpanel/ea-php82/root/usr/bin/php`, for every `php artisan` command.

## 3. Production `.env`

Set these values with the real database and mail credentials:

```dotenv
APP_ENV=production
APP_DEBUG=false
APP_URL=https://test.talentooo.com
LOG_LEVEL=info

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=REAL_DATABASE
DB_USERNAME=REAL_DATABASE_USER
DB_PASSWORD=REAL_DATABASE_PASSWORD

SESSION_DRIVER=database
CACHE_STORE=database
QUEUE_CONNECTION=database
FILESYSTEM_DISK=local
```

Also configure Sanctum, mail, Firebase, storage, and Sentry values from the hosting environment. Never commit `.env`.

## 4. Cron Scheduler

Add this cPanel cron job with a frequency of every minute:

```cron
* * * * * cd /home/USERNAME/lms-api && /FULL/PATH/TO/php artisan schedule:run >> /home/USERNAME/lms-api/storage/logs/scheduler.log 2>&1
```

This runs the scheduled attendance summaries, reports, payment reminders, session generation, cleanup, backups, and queue health commands defined in `routes/console.php`.

## 5. Queue Worker On Shared Hosting

Because Shared Hosting normally cannot keep Supervisor running, add this second cron job:

```cron
* * * * * cd /home/USERNAME/lms-api && /FULL/PATH/TO/php artisan queue:work database --stop-when-empty --sleep=3 --tries=3 --timeout=120 >> /home/USERNAME/lms-api/storage/logs/queue-worker.log 2>&1
```

If `flock` is available, prevent overlapping workers:

```cron
* * * * * flock -n /tmp/talentooo-queue.lock -c 'cd /home/USERNAME/lms-api && /FULL/PATH/TO/php artisan queue:work database --stop-when-empty --sleep=3 --tries=3 --timeout=120' >> /home/USERNAME/lms-api/storage/logs/queue-worker.log 2>&1
```

## 6. Verify The API

These commands should be run after pointing the subdomain to `public`:

```bash
curl -i https://test.talentooo.com/up
curl -i https://test.talentooo.com/api/v1/auth/me
php artisan about
php artisan route:list --path=api/v1
php artisan schedule:list
php artisan queue:monitor database:default
```

Expected results:

- `/up` returns HTTP 200.
- `/api/v1/auth/me` returns HTTP 401 JSON without a token, not an HTML 404 page.
- `schedule:list` shows the application schedules.
- The queue command reports the database queue.

## 7. Build Both Applications

The application defaults now point to the production API, but use an explicit define for release builds:

```bash
cd admin_app
flutter build apk --release --dart-define=API_BASE_URL=https://test.talentooo.com/api/v1
flutter build web --release --dart-define=API_BASE_URL=https://test.talentooo.com/api/v1

cd ../student_app
flutter build apk --release --dart-define=API_BASE_URL=https://test.talentooo.com/api/v1
flutter build web --release --dart-define=API_BASE_URL=https://test.talentooo.com/api/v1
```

Upload each `build/web` to its own web document root if web builds are used. The Android APKs are under `build/app/outputs/flutter-apk/`.
