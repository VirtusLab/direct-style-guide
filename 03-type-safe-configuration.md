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

`derives ConfigReader` uses Scala 3's type class derivation to automatically
generate a reader that maps HOCON keys to case class fields. PureConfig handles
the `camelCase` → `kebab-case` conversion (e.g., `migrateOnStart` reads from
`migrate-on-start`).

Standard types (`String`, `Int`, `Boolean`, `Duration`, `FiniteDuration`) are
supported out of the box. Custom types need a `given ConfigReader` instance.

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

The `${?VAR}` syntax means: if the environment variable is set, use its value;
otherwise, keep the default. This is a HOCON feature, not PureConfig-specific.
It provides 12-factor app configuration without additional code.

## Sensitive values

Passwords and API keys use a `Sensitive` wrapper that overrides `toString` to
prevent accidental logging:

```scala
case class Sensitive(value: String):
  override def toString: String = "***"

object Sensitive:
  given ConfigReader[Sensitive] = pureconfig.ConfigReader[String].map(Sensitive(_))
```

The `given ConfigReader[Sensitive]` instance lets PureConfig read `Sensitive`
fields from plain strings in the config file. The `.value` accessor is used when
the actual string is needed (e.g., when connecting to the database).

Usage in a config class:

```scala
case class DBConfig(username: String, password: Sensitive, ...) derives ConfigReader
```

Logging the config shows `password: ***` instead of the actual value.

## Loading and logging

Configuration is loaded once at startup:

```scala
object Config:
  def read: Config = ConfigSource.default.loadOrThrow[Config]
```

`loadOrThrow` reads from the default source (`application.conf` on the
classpath, with system property and environment variable overrides) and throws
an exception with a detailed error message if any required field is missing or
has the wrong type. This fails fast at startup rather than at first use.

The config is logged at startup for debugging, with `Sensitive` values masked:

```scala
def log(config: Config): Unit =
  logger.info(s"""
    |Bootzooka configuration:
    |DB:    ${config.db}
    |API:   ${config.api}
    |Email: ${config.email}
    |""".stripMargin)
```

Because `Sensitive.toString` returns `***`, passwords and API keys are masked in
the log output. The production code also logs build metadata here — see [Version
API](11-version-api.md).

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
