# SQL Persistence

## Dependencies

- `"com.augustnagro" %% "magnum"` — Scala 3 database client
- `"org.postgresql" % "postgresql"` — JDBC driver
- `"com.zaxxer" % "HikariCP"` — connection pool
- `"org.flywaydb" % "flyway-database-postgresql"` — database migrations

---

## Database setup

The `DB` class wraps a `DataSource` with a Magnum `Transactor`:

```scala
class DB(dataSource: DataSource & Closeable) extends AutoCloseable:
  private val transactor = Transactor(
    dataSource = dataSource,
    sqlLogger = SqlLogger.logSlowQueries(200.millis)
  )
```

The connection pool uses HikariCP with virtual threads:

```scala
val hikariConfig = new HikariConfig()
hikariConfig.setJdbcUrl(config.url)
hikariConfig.setUsername(config.username)
hikariConfig.setPassword(config.password.value)
hikariConfig.setThreadFactory(Thread.ofVirtual().factory())
```

> **Required:** `setThreadFactory(Thread.ofVirtual().factory())` is needed so
> HikariCP uses virtual threads internally. Without it, the pool creates
> platform threads, which can cause thread starvation under load.

## Schema migrations

Flyway runs migrations at startup with retry:

```scala
@tailrec
def connectAndMigrate(ds: DataSource): Unit =
  try
    migrate()
    testConnection(ds)
    logger.info("Database migration & connection test complete")
  catch
    case NonFatal(e) =>
      logger.warn("Database not available, waiting 5 seconds to retry...", e)
      sleep(5.seconds)
      connectAndMigrate(ds)
```

Migration files live in `src/main/resources/db/migration/` following Flyway's
naming convention (`V1__create_schema.sql`, etc.).

## Defining models

Case classes map to tables via Magnum's `@Table` annotation:

```scala
@Table(PostgresDbType, SqlNameMapper.CamelToSnakeCase)
@SqlName("users")
case class User(
    id: Id[User],
    login: String,
    @SqlName("login_lowercase") loginLowerCase: LowerCased,
    @SqlName("email_lowercase") emailLowerCase: LowerCased,
    @SqlName("password") passwordHash: Hashed,
    createdOn: Instant
)
```

## Custom type codecs

Opaque types and custom types need `DbCodec` instances:

```scala
object Magnum:
  given DbCodec[Instant] =
    summon[DbCodec[OffsetDateTime]].biMap(_.toInstant, _.atOffset(ZoneOffset.UTC))

  given idCodec[T]: DbCodec[Id[T]] =
    DbCodec.StringCodec.biMap(_.asId[T], _.toString)

  given DbCodec[Hashed] = DbCodec.StringCodec.biMap(_.asHashed, _.toString)
  given DbCodec[LowerCased] = DbCodec.StringCodec.biMap(_.toLowerCased, _.toString)
```

`Id[T]`, `Hashed`, and `LowerCased` are opaque types over `String` (see below),
so they use `StringCodec`. `Instant` is stored as `TIMESTAMPTZ` in PostgreSQL
and mapped via `OffsetDateTime`.

## Opaque types for domain values

```scala
object Strings:
  opaque type Id[T] = String
  opaque type LowerCased <: String = String
  opaque type Hashed <: String = String

  extension (s: String)
    def asId[T]: Id[T] = s
    def asHashed: Hashed = s
    def toLowerCased[T]: LowerCased = s.toLowerCase(Locale.ENGLISH)
```

## Repository pattern

`Repo` provides standard CRUD operations. `TableInfo` provides table/column
metadata for custom SQL:

```scala
class UserModel:
  private val userRepo = Repo[User, User, Id[User]]
  private val u = TableInfo[User, User, Id[User]]

  export userRepo.{insert, findById}

  private def findBy(by: Spec[User])(using DbTx): Option[User] =
    userRepo.findAll(by).headOption

  def findByEmail(email: LowerCased)(using DbTx): Option[User] =
    findBy(Spec[User].where(sql"${u.emailLowerCase} = $email"))

  def updatePassword(userId: Id[User], newPassword: Hashed)(using DbTx): Unit =
    sql"""UPDATE $u SET ${u.passwordHash} = $newPassword WHERE ${u.id} = $userId"""
      .update.run().discard
```

Custom queries use Magnum's `sql` interpolation — values become bind parameters,
and `$u` / `${u.fieldName}` resolve to table and column names.

## Transactions

All database operations require a `DbTx` context parameter. The `DB` class
provides two transaction entry points (see [Error
Handling](200-error-handling.md) for `transactEither`):

```scala
def transact[T](f: DbTx ?=> T)(using NotGiven[T <:< Either[?, ?]]): T =
  com.augustnagro.magnum.transact(transactor)(f)
```
