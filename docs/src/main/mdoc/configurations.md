---
id: configurations
title: Configurations
---

[`ConfigValue`][configvalue] is the central concept in the library. It represents a single configuration value or a composition of multiple values. The library provides functions like `env`, `file`, and `prop` for creating `ConfigValue`s for environment variables, file contents, and system properties. The [ciris-http4s-aws](modules.md#http4s-aws) module adds support for reading values from the [AWS Systems Manager (SSM) Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html), while [external modules](overview.md#external-modules) provide support for additional configuration sources.

If a configuration value is missing, `or` lets us use a fallback.

```scala mdoc:reset-object:silent
import ciris._

val port: ConfigValue[Effect, Int] =
  env("API_PORT").or(prop("api.port")).as[Int]
```

Using `as` we can attempt to decode the value to a different type.

Note `Effect` here means the configuration value can be used with any effect type (with an `Async` instance). If we're working with a concrete effect type (e.g. `IO`) or an abstract effect type (i.e. `F[_]`), we can specify the return type explicitly or use `covary` in case we want to fix the effect type.

```scala mdoc
import cats.effect.IO

port: ConfigValue[IO, Int]

port.covary[IO]
```

Multiple values can be loaded and combined in parallel, and errors accumulated, using `parMapN`.

```scala mdoc:silent
import cats.syntax.all._
import scala.concurrent.duration._

final case class ApiConfig(port: Int, timeout: Option[Duration])

val timeout: ConfigValue[Effect, Option[Duration]] =
  env("API_TIMEOUT").as[Duration].option

val apiConfig: ConfigValue[Effect, ApiConfig] =
  (port, timeout).parMapN(ApiConfig)
```

We can also use `flatMap`, or for-comprehensions, to load values without error accumulation.

```scala mdoc:silent
for {
  port <- env("API_PORT").or(prop("api.port")).as[Int]
  timeout <- env("API_TIMEOUT").as[Duration].option
} yield ApiConfig(port, timeout)
```

Using `option` we wrap the value in `Option`, using `None` if the value is missing.

## Defaults

Instead of using `None` as default with `option`, we can specify a default with `default`.

```scala mdoc:silent
env("API_TIMEOUT").as[Duration].default(10.seconds)
```

Note that using `a.option` is equivalent to `a.map(_.some).default(None)`.

Default values will only be used if the value is missing. If the value is a composition of multiple values, the default will only be used if all of them are missing. Additionally, later defaults override any earlier defined defaults. This behaviour enables us to specify a default for a composition of values.

```scala mdoc:silent
(
  env("API_PORT").as[Int],
  env("API_TIMEOUT").as[Duration].option
).parMapN(ApiConfig).default {
  ApiConfig(3000, 20.seconds.some)
}
```

When using a fallback with `or`, defaults in the fallback will override earlier defaults.

```scala mdoc:silent
env("API_PORT").as[Int].default(9000)
  .or(prop("api.port").as[Int].default(3000))
```

We can create a default value using `default`, with `a.default(b)` equivalent to `a.or(default(b))`.

```scala mdoc:silent
env("API_PORT").as[Int].or(default(9000))
```

## Secrets

When loading sensitive configuration values, `secret` can be used.

```scala mdoc:silent
val apiKey: ConfigValue[Effect, Secret[String]] =
  env("API_KEY").secret
```

By using `secret`, the value is wrapped in [`Secret`][secret], which prevents the value from being shown. When shown, the value is replaced by the first 7 characters of the SHA-1 hash for the value. This enables us to check whether the correct secret is being used, while not exposing the value.

```scala mdoc
Secret("RacrqvWjuu4KVmnTG9b6xyZMTP7jnX")
```

To calculate the short hash ourselves, we can e.g. use `sha1sum`.

```bash
$ echo -n "RacrqvWjuu4KVmnTG9b6xyZMTP7jnX" | sha1sum | head -c 7
0a7425a
```

When using `secret`, sensitive details, like the value, are also redacted from errors.

### Redacting

In addition to `secret` there is also `redacted` which redacts sensitive details from errors, without wrapping the value in [`Secret`][secret]. We might not want to use [`Secret`][secret] and show the first 28/160 bits of the SHA-1 hash if there are few enough possible values to enable bruteforcing.

## Alternatives

Using `or`, we can provide a fallback configuration. Similarly, `alt` allows us to provide an alternative configuration. The difference between them is that `alt` considers the alternative configuration even though the first configuration has been partially loaded (while `or` only considers the fallback configuration when the first configuration is missing).

Consider the following example where we have different configurations across environments.

```scala mdoc:silent
sealed trait Config
case class DevConfig(port: Int, admin: Boolean) extends Config
case class ProdConfig(port: Int, key: Secret[String]) extends Config

val dev = (
  env("API_PORT").as[Int],
  env("ADMIN").as[Boolean]
).parMapN(DevConfig.apply).widen[Config]

val prod = (
  env("API_PORT").as[Int],
  env("API_KEY").secret
).parMapN(ProdConfig.apply).widen[Config]
```

If we set `API_PORT` and `API_KEY` and use `dev.or(prod)`, we will not attempt to load `prod`. This is because we succeeded to load `API_PORT` for `dev`, and will therefore not consider fallbacks. If we want to consider fallbacks even though a configuration was partially loaded, we can instead use `dev.alt(prod)`.

## Loading

In order to load a configuration, we can use `load` and specify an effect type.

```scala mdoc:silent
import cats.effect.{ExitCode, IOApp}

object Main extends IOApp {
  def run(args: List[String]): IO[ExitCode] =
    apiConfig.load[IO].as(ExitCode.Success)
}
```

We can use `attempt` instead if we want access to the [`ConfigError`][configerror] messages.

## Decoders

When decoding using `as`, a matching [`ConfigDecoder`][configdecoder] instance has to be available.

The library provides instances for many common types, but we can also write an instance.

```scala mdoc:silent
sealed abstract case class PosInt(value: Int)

object PosInt {
  def apply(value: Int): Option[PosInt] =
    if(value > 0)
      Some(new PosInt(value) {})
    else None

  implicit val posIntConfigDecoder: ConfigDecoder[String, PosInt] =
    ConfigDecoder[String, Int].mapOption("PosInt")(apply)
}

env("MAX_RETRIES").as[PosInt]
```

## Sources

To support new configuration sources, we can use the [`ConfigValue`][configvalue$] functions.

Following is an example showing how the `env` function can be defined.

```scala mdoc:silent
def env(name: String): ConfigValue[Effect, String] =
  ConfigValue.suspend {
    val key = ConfigKey.env(name)
    val value = System.getenv(name)

    if (value != null) {
      ConfigValue.loaded(key, value)
    } else {
      ConfigValue.missing(key)
    }
  }
```

The [`ConfigKey`][configkey] is a description of the key, e.g. `s"environment variable $name"`. The function returns `missing` when there is no value for the key, and `loaded` when a value is available. Since reading environment variables can throw `SecurityException`s, we capture side effects using `suspend`.

[configdecoder]: @API_BASE_URL@/ConfigDecoder.html
[configerror]: @API_BASE_URL@/ConfigError.html
[configkey]: @API_BASE_URL@/ConfigKey.html
[configvalue]: @API_BASE_URL@/ConfigValue.html
[configvalue$]: @API_BASE_URL@/ConfigValue$.html
[secret]: @API_BASE_URL@/Secret.html
