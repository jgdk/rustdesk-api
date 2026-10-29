# hitransform-fork af rustdesk-api

Fork af [lejianwen/rustdesk-api](https://github.com/lejianwen/rustdesk-api) v2.7 (MIT). Giver login, adressebog og web-admin til vores open source RustDesk-server (`remote.hitransform.com`) uden RustDesk Server Pro. Upstream har været inaktiv siden 2025-09-28, så vi vedligeholder selv.

- **Branch `hitransform`** er vores hovedbranch. `master` holdes som ren kopi af upstream.
- **Image:** `ghcr.io/jgdk/rustdesk-api:hitransform` (og `:hitransform-<sha>`), bygget af `.github/workflows/hitransform.yml` med vet, tests, govulncheck og Trivy. Kører hver mandag.
- **Admin-UI** bygges fra `lejianwen/rustdesk-api-web` på commit `3998c2a` (se `Dockerfile.hitransform`).

## Sikkerhedsgennemsyn (2026-10-29)

Afhængigheder: govulncheck fandt 11 kendte sårbarheder før opdatering, 0 efter.

| # | Alvor | Fund | Status |
|---|---|---|---|
| 1 | Høj | `/api/login` uden brute-force-stop (`ban-threshold: 0`) | Config + nginx `limit_req` |
| 2 | Høj | Klient-IP kan forfalskes via `X-Forwarded-For` (gin stoler på alle proxyer) | Config `trust-proxy` + nginx |
| 3 | Mellem | Tokens = MD5(navn+tid) når JWT-nøgle mangler (`service/user.go:89`) | Config nu, kodefix senere |
| 4 | Mellem | Admin-logout tilbagekalder ikke token (`http/router/admin.go:23`) | Kodefix |
| 5 | Mellem | Første admin-adgangskode skrives i loggen (`cmd/apimain.go:369`) | Skift kode og ryd log; kodefix |
| 6 | Mellem | Åbne klient-endpoints (`/api/heartbeat`, `/api/sysinfo`, audit) uden login | nginx rate-limit |
| 7 | Mellem | Webklienten sender data til Google (Firebase, fonts) | `WEB_CLIENT=0` |
| 8 | Mellem | Upstream-compose publicerer 21114 offentligt, kører som root | Løst i vores image (uid 10001); compose skal binde til 127.0.0.1 |
| 9-14 | Lav | Delingslinks uden udløb, brugerliste synlig for alle, samling kan overdrages (`my/addressBookCollection.go:116`), klartekst-tokens i SQLite, OIDC-kobling på e-mail | Se rapport; mest relevant hvis kunder får konti |

## Før produktion (skal sættes)

- `RUSTDESK_API_JWT_KEY` = lang tilfældig hemmelighed
- `RUSTDESK_API_APP_BAN_THRESHOLD=5`
- `RUSTDESK_API_GIN_TRUST_PROXY` = nginx/docker-gateway-IP
- `RUSTDESK_API_APP_REGISTER=false`, `RUSTDESK_API_APP_SHOW_SWAGGER=0`, `RUSTDESK_API_APP_WEB_CLIENT=0`
- `RUSTDESK_API_GIN_MODE=release`
- `RUSTDESK_API_RUSTDESK_ID_SERVER=remote.hitransform.com`, `..._RELAY_SERVER=remote.hitransform.com`, `..._API_SERVER=https://<api-domæne>`, `..._KEY=<serverens offentlige nøgle>`
- Port `127.0.0.1:21114:21114`, kun HTTPS via CloudPanel, `limit_req` på login-stierne, `X-Forwarded-For $remote_addr`
- Skift admin-adgangskoden straks efter første start, og ryd loggen
- Backup af `/app/data` (indeholder adressebøger og tokens i klartekst)
