# Type-Safe Configuration

## Dependencies

- `"com.github.pureconfig" %% "pureconfig-core"` — reads HOCON/properties files
  into Scala case classes

---

## Configuration as case classes

Each module defines its own configuration as a case class with `derives
ConfigReader`:

```scala
case class HttpConfig(host: String, port: Int) derives ConfigReader

case class DBConfig(username: String, password: Sensitive, url: String,
                    migrateOnStart: Boolean, driver: String) derives ConfigReader

case class UserConfig(defaultApiKeyValid: Duration) derives ConfigReader

case class PasswordResetConfig(resetLinkPattern: String, codeValid: Duration) derives ConfigReader
```

`derives ConfigReader` auto-generates a reader with `camelCase` → `kebab-case`
key mapping (e.g., `migrateOnStart` reads from `migrate-on-start`). Custom
types need a `given ConfigReader` instance.

## Nested configuration

The top-level `Config` composes all module configs:

```scala
case class Config(db: DBConfig, api: HttpConfig, email: EmailConfig,
                  passwordReset: PasswordResetConfig, user: UserConfig)
    derives ConfigReader
```

This mirrors the HOCON structure in `application.conf`:

```hocon
api {
  host = "0.0.0.0"
  port = 8080
}

db {
  username = "postgres"
  password = "bootzooka"
  url = "jdbc:postgresql://"${db.host}":"${db.port}"/"${db.name}
  migrate-on-start = true
}

user {
  default-api-key-valid = 1 day
}
```

## Environment variable overrides

Each setting can be overridden via an environment variable using HOCON's
substitution syntax:

```hocon
api {
  host = "0.0.0.0"
  host = ${?API_HOST}

  port = 8080
  port = ${?API_PORT}
}
```

## Sensitive values

Passwords and API keys use a `Sensitive` wrapper that overrides `toString` to
prevent accidental logging:

```scala
case class Sensitive(value: String):
  override def toString: String = "***"

object Sensitive:
  given ConfigReader[Sensitive] = pureconfig.ConfigReader[String].map(Sensitive(_))
```

## Loading and logging

Configuration is loaded once at startup:

```scala
object Config:
  def read: Config = ConfigSource.default.loadOrThrow[Config]
```

The config is logged at startup for debugging, with `Sensitive` values masked:

```scala
def log(config: Config): Unit =
  logger.info(s"""
    |Application configuration:
    |DB:    ${config.db}
    |API:   ${config.api}
    |Email: ${config.email}
    |""".stripMargin)
```

The production code also logs build metadata here — see [Version
API](320-version-api.md).

## Validation at load time

Config case classes can validate their values in the constructor body:

```scala
case class PasswordResetConfig(resetLinkPattern: String, codeValid: Duration)
    derives ConfigReader:
  validate()

  def validate(): Unit =
    val testCode = "TEST_123"
    assert(
      String.format(resetLinkPattern, testCode).contains(testCode),
      s"Invalid reset link pattern: $resetLinkPattern"
    )
```

This runs when `Config.read` constructs the nested `PasswordResetConfig`, so an
invalid `resetLinkPattern` fails the application at startup, not when a user
first requests a password reset.
