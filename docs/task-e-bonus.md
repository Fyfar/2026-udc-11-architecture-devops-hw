# Task E (bonus) — обрано Шлях 1: стек по-справжньому (Docker)

Перевірено 2026-10-27 на macOS (x86_64) тричі: (1) engine 20.10.12 + Docker Compose v2.29.7;
(2) після оновлення Docker Desktop до 4.97.0 — engine 29.9.2 + Docker Compose v5.6.1, командою
пайплайну `up -d --wait --wait-timeout 90` (exit 0). `deploy/Dockerfile` і `deploy/compose.yaml` —
ті самі, що в Task D. Вивід нижче — з першого прогону. Третій прогін (2026-10-27, після зміни
`SMS_RATE_PER_SEC` з 20 на 8 за post-mortem, дія 6) — так само `up --wait` exit 0, `api` healthy,
`worker` Up; у worker-і `tr "\0" "\n" < /proc/1/environ | grep "^SMS_RATE"` → `SMS_RATE_PER_SEC=8`
(та сама команда, що в runbook, Д9).

## Шлях 1 — стек по-справжньому (Docker)

- Команди, якими підняли стек (змінні — у команді, не в закоміченому `.env`):

  ```bash
  # 1. Образ. У compose навмисно немає `build:` — на VM образ підтягується за digest
  #    (POSHTOMAT_IMAGE), тож `up --build` з прикладу тут нічого не збирає; збираємо окремо.
  docker build -t poshtomat:local -f deploy/Dockerfile app

  # 2. secrets.env (env_file з required: true) — поза репо, у тимчасовому каталозі, який
  #    відтворює /opt/poshtomat на VM. --project-directory робить його базою для ./secrets.env.
  VM=$(mktemp -d)
  printf 'DATABASE_URL=CHANGE_ME\nSMS_TOKEN=CHANGE_ME\n' > "$VM/secrets.env"
  chmod 600 "$VM/secrets.env"

  # 3. Стек: POSHTOMAT_IMAGE — у команді, не в .env
  POSHTOMAT_IMAGE=poshtomat:local docker compose -f deploy/compose.yaml --project-directory "$VM" config -q
  POSHTOMAT_IMAGE=poshtomat:local docker compose -f deploy/compose.yaml --project-directory "$VM" up -d
  POSHTOMAT_IMAGE=poshtomat:local docker compose -f deploy/compose.yaml --project-directory "$VM" ps
  ```

  Окремої бази не потрібно: сховище в `app/src/db.js` — у пам'яті, `DATABASE_URL` лише
  задовольняє обов'язковий `secrets.env`.

- Вивід `docker compose -f deploy/compose.yaml ps` — `api` `healthy`; `worker` `Up`:

  ```text
  NAME          IMAGE             COMMAND                  SERVICE   CREATED          STATUS                    PORTS
  vm-api-1      poshtomat:local   "docker-entrypoint.s…"   api       34 seconds ago   Up 33 seconds (healthy)   0.0.0.0:8080->8080/tcp
  vm-worker-1   poshtomat:local   "docker-entrypoint.s…"   worker    34 seconds ago   Up 33 seconds             8080/tcp
  ```

  `worker` — `Up` без `(healthy)`, бо в compose для нього `healthcheck: disable: true`:
  worker не слухає порт, і `HEALTHCHECK` з образу (запит до `/health` на 8080) для нього
  завжди падав би, тож Docker вічно показував би `unhealthy`.

- Запит до `GET /health` і відповідь:

  ```text
  $ curl -si http://127.0.0.1:8080/health
  HTTP/1.1 200 OK
  content-type: application/json; charset=utf-8

  {"status":"ok","version":"1.7.3"}
  ```

  Логи старту обох сервісів:

  ```text
  api-1     | {"ts":"2026-10-27T10:05:30.614+02:00","level":"info","msg":"service started","version":"1.7.3"}
  worker-1  | {"ts":"2026-10-27T10:05:30.596+02:00","level":"info","msg":"worker started","version":"1.7.3"}
  ```

- Додатково перевірили, що посилення з Task D працює під час виконання, а не лише в YAML:

  ```text
  $ docker compose ... exec api id
  uid=1000(node) gid=1000(node) groups=1000(node)          # USER node, не root
  $ docker compose ... exec api sh -c 'touch /app/x; touch /tmp/ok && echo tmp-writable'
  touch: cannot touch '/app/x': Read-only file system      # read_only: true
  tmp-writable                                             # tmpfs /tmp працює
  $ docker compose ... down                                 # 5 с — менше за stop_grace_period 20s,
                                                           # тобто SIGTERM оброблено, без SIGKILL
  ```

- Що пішло не так і як полагодили:
  1. **Compose 2.2.3 не прочитав `compose.yaml`:** `services.api.env_file.0 must be a string`.
     Довга форма `env_file: [{path, required}]` з'явилася лише в Docker Compose 2.24; Docker
     Desktop тут постачав 2.2.3. Полагодили оновленням Compose, а не файлу: поставили
     офіційний бінарник v2.29.7 у `~/.docker/cli-plugins/docker-compose` (sha256 звірено з
     `.sha256` з релізу; плагін користувача має пріоритет над вбудованим у Docker Desktop).
     **Висновок для прод:** на VM потрібен Docker Compose ≥ 2.24 — інакше `docker compose up`
     у `pipeline/deploy.yml` впаде на тому самому розборі, вже після заміни файлів. Тому в
     крок «Roll out» додано перевірку `docker compose version --short` проти 2.24.0
     (`sort -C -V`, до `docker pull` і будь-яких змін на диску), а в `docs/runbook.md` —
     вимогу до версії. Перевірку прогнали окремо: 5.6.1 → проходить, підставлена v2.2.3 →
     `::error::Docker Compose v2.2.3 on VM, need >= 2.24`, exit 1.
  2. **`up --build` з прикладу нічого не збирає:** у compose немає `build:` (на VM образ лише
     підтягується за digest), тому образ зібрали окремо `docker build` і передали його тег
     через `POSHTOMAT_IMAGE` у команді.
  3. **`secrets.env` обов'язковий і не був у `.gitignore`** (там були лише `.env` і `.env.*`).
     Додали `secrets.env` у `.gitignore` (`git check-ignore -v deploy/secrets.env` → правило
     спрацьовує). Для прогону файл із `CHANGE_ME` однаково створено поза репо, а compose
     спрямовано на нього через `--project-directory`. Побічний ефект — ім'я проєкту `vm`
     (від назви каталогу), звідси `vm-api-1`; на VM це буде `poshtomat-…`.
  4. **Після оновлення Docker Desktop `up` впав з `pull access denied for poshtomat`:**
     оновлення engine 20 → 29 очистило локальні образи, і compose, не знайшовши
     `poshtomat:local`, пішов шукати його в Docker Hub. Це не помилка compose: образ перезібрали
     тією самою `docker build`, і стек піднявся.
