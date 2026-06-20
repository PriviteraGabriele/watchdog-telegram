# CLAUDE.md — watchdog-telegram

Leggi questo file integralmente prima di fare qualsiasi cosa.
Queste istruzioni descrivono esattamente cosa fare, in quale ordine, e gli standard da rispettare.

---

## Contesto

Questo è un fork di `appleboy/drone-telegram` (Go, MIT).
Il plugin manda notifiche Telegram da pipeline Drone/Woodpecker CI.

**Nuovo nome:** `watchdog-telegram`
**Docker Hub:** `thecorps/watchdog-telegram`
**GitHub:** `thecorps/watchdog-telegram` (fork di `appleboy/drone-telegram`)

---

## Struttura del codebase originale

Solo due file Go rilevanti:

### `main.go`
- Entry point CLI con `github.com/urfave/cli`
- Definisce tutti i flag (nome → env var)
- Crea il `Plugin` struct e chiama `plugin.Exec()`
- **Problema attuale:** i flag mappano solo a env var `DRONE_*` — mancano le `CI_*` di Woodpecker

### `plugin.go`
- Struct: `GitHub`, `Repo`, `Commit`, `Build`, `Config`, `Plugin`
- `Plugin.Exec()` — costruisce il bot, seleziona il messaggio, invia a ogni destinatario
- `Plugin.Send()` — wrapper `bot.Send()` con debug logging
- `Plugin.Message()` — template di messaggio di default
- Libreria Telegram: `github.com/go-telegram-bot-api/telegram-bot-api` v4.6.4
- Formato: `Markdown` (default attuale) o `HTML` (già supportato)
- `disable_notification` già presente nel codebase

### `docker/Dockerfile`
- Alpine multi-arch
- Copia il binario da `release/${TARGETOS}/${TARGETARCH}/drone-telegram` → da rinominare
- Entry point: `/bin/drone-telegram` → da rinominare

---

## Cosa implementare (in questo ordine)

### Step 1 — Rinominare modulo e binario

**`go.mod`** — cambia la prima riga:
```
module github.com/appleboy/drone-telegram
```
→
```
module github.com/thecorps/watchdog-telegram
```

**`Makefile`** — riga 1:
```
EXECUTABLE := drone-telegram
```
→
```
EXECUTABLE := watchdog-telegram
```

**`docker/Dockerfile`** — aggiorna il COPY e l'ENTRYPOINT:
```dockerfile
COPY release/${TARGETOS}/${TARGETARCH}/watchdog-telegram /bin/
ENTRYPOINT ["/bin/watchdog-telegram"]
```

Aggiorna anche le LABEL:
```dockerfile
LABEL maintainer="Gabriele Privitera <prigab.zucchetti@gmail.com>" \
  org.label-schema.name="Watchdog Telegram Plugin" \
  org.label-schema.vendor="thecorps"
LABEL org.opencontainers.image.source=https://github.com/thecorps/watchdog-telegram
LABEL org.opencontainers.image.description="Enhanced Drone/Woodpecker CI Telegram notifications"
```

**`main.go`** — aggiorna nome e usage:
```go
app.Name = "watchdog-telegram plugin"
app.Usage = "watchdog-telegram plugin"
```

---

### Step 2 — Aggiungere env var Woodpecker CI in `main.go`

Woodpecker usa env var `CI_*` invece di `DRONE_*`. Aggiungi gli alias a tutti i flag rilevanti.
La sintassi è già usata nel codice: `EnvVar: "DRONE_X,GITHUB_X"` — aggiungi `CI_X` nella stessa stringa.

| Flag | EnvVar attuale | Aggiungi |
|------|----------------|----------|
| `repo` | `DRONE_REPO,GITHUB_REPOSITORY` | `,CI_REPO` |
| `repo.namespace` | `DRONE_REPO_OWNER,DRONE_REPO_NAMESPACE,GITHUB_ACTOR` | `,CI_REPO_OWNER` |
| `repo.name` | `DRONE_REPO_NAME` | `,CI_REPO_NAME` |
| `commit.sha` | `DRONE_COMMIT_SHA,GITHUB_SHA` | `,CI_COMMIT_SHA` |
| `commit.ref` | `DRONE_COMMIT_REF,GITHUB_REF` | `,CI_COMMIT_REF` |
| `commit.branch` | `DRONE_COMMIT_BRANCH` | `,CI_COMMIT_BRANCH` |
| `commit.link` | `DRONE_COMMIT_LINK` | `,CI_COMMIT_URL` |
| `commit.author` | `DRONE_COMMIT_AUTHOR` | `,CI_COMMIT_AUTHOR` |
| `commit.author.email` | `DRONE_COMMIT_AUTHOR_EMAIL` | `,CI_COMMIT_AUTHOR_EMAIL` |
| `commit.author.avatar` | `DRONE_COMMIT_AUTHOR_AVATAR` | `,CI_COMMIT_AUTHOR_AVATAR` |
| `commit.message` | `DRONE_COMMIT_MESSAGE` | `,CI_COMMIT_MESSAGE` |
| `build.event` | `DRONE_BUILD_EVENT` | `,CI_PIPELINE_EVENT` |
| `build.number` | `DRONE_BUILD_NUMBER` | `,CI_PIPELINE_NUMBER` |
| `build.status` | `DRONE_BUILD_STATUS` | `,CI_PIPELINE_STATUS` |
| `build.link` | `DRONE_BUILD_LINK` | `,CI_PIPELINE_URL` |
| `build.tag` | `DRONE_TAG` | `,CI_COMMIT_TAG` |
| `build.started` | `DRONE_STAGE_STARTED` | `,CI_PIPELINE_STARTED` |
| `build.finished` | `DRONE_BUILD_FINISHED` | `,CI_PIPELINE_FINISHED` |

---

### Step 3 — Aggiungere nuovi flag in `main.go`

Aggiungi questi flag nel blocco `app.Flags`, dopo i flag esistenti:

```go
cli.StringFlag{
    Name:   "prev.build.status",
    Usage:  "previous build status",
    EnvVar: "CI_PREV_PIPELINE_STATUS,DRONE_PREV_BUILD_STATUS",
},
cli.StringFlag{
    Name:   "message.success",
    Usage:  "telegram message template for successful builds",
    EnvVar: "PLUGIN_MESSAGE_SUCCESS",
},
cli.StringFlag{
    Name:   "message.failure",
    Usage:  "telegram message template for failed builds",
    EnvVar: "PLUGIN_MESSAGE_FAILURE",
},
cli.StringFlag{
    Name:   "message.fixed",
    Usage:  "telegram message template when build recovers from failure",
    EnvVar: "PLUGIN_MESSAGE_FIXED",
},
cli.BoolFlag{
    Name:   "disable.notification.on.success",
    Usage:  "send success notifications silently (no sound)",
    EnvVar: "PLUGIN_DISABLE_NOTIFICATION_ON_SUCCESS",
},
cli.StringFlag{
    Name:   "buttons",
    Usage:  "inline keyboard buttons as JSON array: [{\"text\":\"...\",\"url\":\"...\"}]",
    EnvVar: "PLUGIN_BUTTONS",
},
```

Aggiungi alla funzione `run()` nel blocco `Config{}`:
```go
PrevBuildStatus:      c.String("prev.build.status"),
MessageSuccess:       c.String("message.success"),
MessageFailure:       c.String("message.failure"),
MessageFixed:         c.String("message.fixed"),
DisableNotifOnSuccess: c.Bool("disable.notification.on.success"),
Buttons:              c.String("buttons"),
```

---

### Step 4 — Modificare `plugin.go`

#### 4a — Aggiungere struct e campi

Aggiungi `Button` struct (prima di `Location`):
```go
Button struct {
    Text string `json:"text"`
    URL  string `json:"url"`
}
```

Aggiungi campi a `Config`:
```go
PrevBuildStatus      string
MessageSuccess       string
MessageFailure       string
MessageFixed         string
DisableNotifOnSuccess bool
Buttons             string
```

Aggiungi `Duration string` a `Build`:
```go
Duration string
```

#### 4b — Aggiungere funzioni helper

**`parseButtons`** — deserializza il JSON dei pulsanti:
```go
func parseButtons(s string) ([]Button, error) {
    if s == "" {
        return nil, nil
    }
    var buttons []Button
    if err := json.Unmarshal([]byte(s), &buttons); err != nil {
        return nil, fmt.Errorf("invalid buttons JSON: %w", err)
    }
    return buttons, nil
}
```

**`buildInlineKeyboard`** — crea la tastiera inline Telegram:
```go
func buildInlineKeyboard(buttons []Button) tgbotapi.InlineKeyboardMarkup {
    row := make([]tgbotapi.InlineKeyboardButton, 0, len(buttons))
    for _, b := range buttons {
        btn := tgbotapi.NewInlineKeyboardButtonURL(b.Text, b.URL)
        row = append(row, btn)
    }
    return tgbotapi.NewInlineKeyboardMarkup(row)
}
```

**`formatDuration`** — calcola durata pipeline:
```go
func formatDuration(started, finished int64) string {
    if started == 0 || finished == 0 {
        return ""
    }
    secs := finished - started
    if secs < 60 {
        return fmt.Sprintf("%ds", secs)
    }
    return fmt.Sprintf("%dm %ds", secs/60, secs%60)
}
```

**`escapeHTMLFields`** — escape HTML per i campi dinamici:
```go
func escapeHTMLFields(fields ...*string) {
    for _, f := range fields {
        *f = html.EscapeString(*f)
    }
}
```

#### 4c — Modificare `Exec()`

All'inizio di `Exec()`, dopo i controlli iniziali e prima del blocco `message`, aggiungi:

```go
// Calcola durata
p.Build.Duration = formatDuration(p.Build.Started, p.Build.Finished)

// Determina se la pipeline si è "ripresa" da un fallimento
isFixed := strings.ToLower(p.Config.PrevBuildStatus) == "failure" &&
    strings.ToLower(p.Build.Status) == "success"
```

Sostituisci il blocco di selezione messaggio:
```go
// Attuale:
var message []string
switch {
case len(p.Config.MessageFile) > 0:
    ...
case len(p.Config.Message) > 0:
    message = []string{p.Config.Message}
default:
    ...
    message = p.Message()
}
```

Con:
```go
var message []string
switch {
case len(p.Config.MessageFile) > 0:
    message, err = loadTextFromFile(p.Config.MessageFile)
    if err != nil {
        return fmt.Errorf("error loading message file '%s': %w", p.Config.MessageFile, err)
    }
case isFixed && len(p.Config.MessageFixed) > 0:
    message = []string{p.Config.MessageFixed}
case strings.ToLower(p.Build.Status) == "failure" && len(p.Config.MessageFailure) > 0:
    message = []string{p.Config.MessageFailure}
case strings.ToLower(p.Build.Status) == "success" && len(p.Config.MessageSuccess) > 0:
    message = []string{p.Config.MessageSuccess}
case len(p.Config.Message) > 0:
    message = []string{p.Config.Message}
default:
    p.Config.Format = formatHTML
    message = p.Message()
}
```

Dopo il blocco format/escape esistente, aggiungi l'escape HTML:
```go
if p.Config.Format == formatMarkdown {
    message = escapeMarkdown(message)
    escapeMarkdownFields(
        &p.Commit.Message, &p.Commit.Branch, &p.Commit.Link,
        &p.Commit.Author, &p.Commit.Email,
        &p.Build.Tag, &p.Build.Link, &p.Build.PR,
        &p.Repo.Namespace, &p.Repo.Name,
    )
} else if p.Config.Format == formatHTML {
    escapeHTMLFields(
        &p.Commit.Message, &p.Commit.Branch,
        &p.Commit.Author, &p.Commit.Email,
        &p.Repo.FullName, &p.Repo.Name, &p.Repo.Namespace,
    )
}
```

Modifica il rendering template — rimuovi `html.UnescapeString` per il formato HTML:
```go
// Prima (uguale per tutti i formati):
renderedMessages = append(renderedMessages, html.UnescapeString(txt))

// Dopo:
if p.Config.Format == formatHTML {
    renderedMessages = append(renderedMessages, txt)
} else {
    renderedMessages = append(renderedMessages, html.UnescapeString(txt))
}
```

Parsea i pulsanti prima del loop sui destinatari:
```go
buttons, err := parseButtons(p.Config.Buttons)
if err != nil {
    return err
}
```

Nel loop `for _, user := range ids`, modifica la creazione del messaggio testo:
```go
for _, txt := range renderedMessages {
    msg := tgbotapi.NewMessage(user, txt)
    msg.ParseMode = p.Config.Format
    msg.DisableWebPagePreview = p.Config.DisableWebPagePreview

    // Silent su successo (non quando è "fixed")
    disableNotif := p.Config.DisableNotification
    if p.Config.DisableNotifOnSuccess && strings.ToLower(p.Build.Status) == "success" && !isFixed {
        disableNotif = true
    }
    msg.DisableNotification = disableNotif

    // Tastiera inline
    if len(buttons) > 0 {
        keyboard := buildInlineKeyboard(buttons)
        msg.ReplyMarkup = keyboard
    }

    if err := p.Send(bot, msg); err != nil {
        return err
    }
}
```

#### 4d — Aggiornare `Message()` default

Aggiorna il template default per usare HTML e includere pipeline number + commit message:
```go
func (p *Plugin) Message() []string {
    icon := icons[strings.ToLower(p.Build.Status)]

    if p.Config.GitHub {
        return []string{fmt.Sprintf("%s/%s triggered by %s (%s)",
            p.Repo.FullName,
            p.GitHub.Workflow,
            p.Repo.Namespace,
            p.GitHub.EventName,
        )}
    }

    return []string{
        fmt.Sprintf("%s <b>Build #%d</b> of <code>%s</code> %s.\n\n📝 <b>%s</b> on <code>%s</code>:\n<pre>%s</pre>\n\n🌐 %s",
            icon,
            p.Build.Number,
            p.Repo.FullName,
            p.Build.Status,
            p.Commit.Author,
            p.Commit.Branch,
            p.Commit.Message,
            p.Build.Link,
        ),
    }
}
```

---

### Step 5 — Modificare il formato default

In `main.go`, nel flag `format`, cambia il `Value` default:
```go
cli.StringFlag{
    Name:   "format",
    Value:  formatHTML,   // era formatMarkdown
    Usage:  "telegram message format (Markdown or HTML)",
    EnvVar: "PLUGIN_FORMAT,FORMAT,INPUT_FORMAT",
},
```

---

## Standard da rispettare

- Go idiomatico — nessuna astrazione non necessaria
- Nessun commento a meno che il PERCHÉ non sia non-ovvio
- Tutte le nuove funzioni vanno in `plugin.go` — non creare nuovi file
- Nuovi flag in `main.go` seguono esattamente il pattern esistente
- Nuovi campi Config/Build inseriti in ordine alfabetico nel loro gruppo

---

## Build e pubblicazione su Docker Hub

### Prerequisiti
- Go installato localmente (`go version` ≥ 1.21)
- Docker con buildx abilitato
- Login su Docker Hub: `docker login`

### Build binari (necessario prima del build Docker)

Il Dockerfile copia il binario pre-compilato da `release/${TARGETOS}/${TARGETARCH}/`:

```bash
# linux/amd64
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
  -ldflags="-s -w" \
  -o release/linux/amd64/watchdog-telegram .

# linux/arm64 (per Raspberry Pi / server ARM)
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build \
  -ldflags="-s -w" \
  -o release/linux/arm64/watchdog-telegram .
```

### Build e push immagine multi-arch

```bash
# Prima volta: crea un builder multi-arch
docker buildx create --name multiarch --use

# Build e push in un comando
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -f docker/Dockerfile \
  -t thecorps/watchdog-telegram:latest \
  -t thecorps/watchdog-telegram:1.0.0 \
  --push \
  .
```

### Versioning (SemVer)
- `v1.0.0` — prima release stabile con tutte le feature
- `v1.x.x` — patch/minor per fix e miglioramenti futuri
- Tagga sempre sia `:latest` che `:x.y.z`

---

## Utilizzo in `.woodpecker.yml`

Sostituisci `appleboy/drone-telegram` con `thecorps/watchdog-telegram`.
Cambia `when.status` da `[failure]` a `[success, failure]` — il plugin gestisce il silenzio internamente.

```yaml
- name: notify-telegram
  image: thecorps/watchdog-telegram
  depends_on: [backend-test, backend-phpstan, backend-deptrac, frontend-typecheck]
  settings:
    token:
      from_secret: TELEGRAM_TOKEN
    to:
      from_secret: TELEGRAM_CHAT_ID
    format: HTML
    disable_notification_on_success: true
    message_failure: |
      🔴 <b>Pipeline Failed</b>

      <b>Repo:</b> <code>${CI_REPO}</code>
      <b>Branch:</b> <code>${CI_COMMIT_BRANCH}</code>
      <b>Build:</b> #${CI_PIPELINE_NUMBER}
      <b>Commit:</b> ${CI_COMMIT_MESSAGE}
      <b>Author:</b> ${CI_COMMIT_AUTHOR}
    message_success: |
      ✅ Pipeline OK · #${CI_PIPELINE_NUMBER} · ${CI_COMMIT_MESSAGE}
    message_fixed: |
      🟢 <b>Pipeline Fixed!</b>

      <b>Build:</b> #${CI_PIPELINE_NUMBER}
      <b>Commit:</b> ${CI_COMMIT_MESSAGE}
    buttons: '[{"text":"🔍 View Pipeline","url":"${CI_PIPELINE_URL}"}]'
  when:
    - event: push
      branch: main
      status: [success, failure]
```

**Note:**
- `disable_notification_on_success: true` — i messaggi di successo arrivano senza suono
- `message_fixed` — ricevi notifica con suono quando la pipeline torna verde dopo un fallimento
- `status: [success, failure]` — lo step gira sempre; è il plugin a scegliere il template e il silenzio
- I `${CI_*}` vengono sostituiti da Woodpecker prima di passarli al plugin — non richiedono template syntax

---

## Test

```bash
go test ./...
```

I test esistenti in `plugin_test.go` usano env var `DRONE_*` — verifica che passino ancora dopo le modifiche.
