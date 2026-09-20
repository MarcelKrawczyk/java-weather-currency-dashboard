# java-weather-currency-dashboard

A small Java desktop app (Swing + an embedded JavaFX WebView) that, for a city and a country, shows the current weather, the exchange rate of the country's currency to a chosen target currency, the National Bank of Poland (NBP) mid rate for that currency in PLN, and the city's Wikipedia page.

Java 21, Maven. No third-party libraries are declared in the `pom.xml`.

---

## What it does

Enter a city, a country (English name, for example `Poland` or `United States`) and a target currency code (for example `USD`), then press Fetch (or Enter). The results panel shows three sections, and the right panel loads the Wikipedia article for the city.

| Section | Source | Content |
|---|---|---|
| WEATHER | OpenWeatherMap current weather (`/data/2.5/weather`, metric units) | conditions, temperature and "feels like" in degrees C, humidity, wind speed in m/s |
| EXCHANGE RATE | Fixer (`/api/latest`) | `1 <country currency> = x <target currency>` |
| NBP RATE | NBP Web API (`api.nbp.pl`, no key needed) | `1 <country currency> = x PLN` (NBP mid rate, table A, then table B) |

The country's currency and ISO country code are derived from the country name with `java.util.Locale` / `java.util.Currency`, not from a lookup table.

## Repository layout

```
src/main/java/java_weather_currency_dashboard/
  Main.java      Swing UI, SwingWorker orchestration, WebView panel
  Service.java   HTTP calls, string-based JSON extraction, country -> currency mapping
pom.xml
```

---

## Requirements and running

- JDK 21
- JavaFX (modules `javafx.controls`, `javafx.web`, `javafx.swing`). The `pom.xml` declares no dependencies, so JavaFX has to come from your JDK distribution or from a JavaFX SDK configured in your IDE. With an SDK, run `Main` with the VM options
  `--module-path <javafx-sdk>/lib --add-modules javafx.controls,javafx.web,javafx.swing`
- API keys for OpenWeatherMap and Fixer. Put them in the two constants at the top of `Service.java` (`OPENWEATHER_API_KEY`, `FIXER_API_KEY`); they currently hold the placeholder `YOUR_API_KEY`. The NBP section works without a key.

Entry point: `java_weather_currency_dashboard.Main`.

---

## Engineering notes

### 1. FX arithmetic and precision
- Fixer returns rates per 1 EUR. The app requests two symbols and computes the cross rate `rate[target] / rate[base]`, that is triangulation through EUR. That is the correct formula for "1 base = x target".
- The result carries the rounding of two quoted inputs, and the division is done in `double`. That is fine for display, but it is not the right type for computing amounts of money (`BigDecimal` with an explicit rounding mode is).
- Rates are printed with `%.4f`. For a currency worth much less than one unit of the target, four decimals leave only one or two significant digits (this is why FX conventions quote some pairs with 2 decimals and others with 4 or 5, or quote the inverse pair). Display precision should follow the size of the rate.
- `String.format` is called without a `Locale`, so the decimal separator follows the machine's default locale (a comma on a Polish system). This matters as soon as the output is parsed or compared.
- The guard `baseRate == 0` before the division is correct and prevents a division by zero.

### 2. Provenance and staleness of the data
- Both providers return an as-of time (Fixer `timestamp` and `date`, NBP `effectiveDate`), but the app discards it. The UI cannot tell a live rate from yesterday's rate or from a weekend rate.
- The NBP call tries table A, then table B. Table A is published on business days; table B, which covers less common currencies, is published weekly, so the same UI line can show data of very different age.
- Both rates are reference or mid rates, not tradable prices. NBP publishes buy/sell (bid/ask) rates separately in table C.
- The two rates are shown side by side and are never reconciled. The difference between them (the basis between two sources) is a natural number to compute and show, in basis points.
- The NBP request uses plain `http://`. The data is unauthenticated in transit and can be altered by anyone on the path; the NBP API is also available over HTTPS.
- Label bug: the section header reads `NBP RATE (PLN -> XXX)`, but the value printed is `1 XXX = x PLN`, which is the opposite direction.

### 3. Latency and concurrency
- The three data calls run one after another inside one `SwingWorker`, so the UI thread stays responsive, but the total latency is the **sum** of the calls. Each request has explicit connect and read timeouts of 8 s, so the worst case is about a minute (weather, Fixer, NBP table A, NBP table B, roughly 16 s each).
- The calls are independent, so they can run concurrently (`CompletableFuture`), which turns the total into the **maximum** of the calls instead of the sum.
- There is no caching, retry or backoff. The free tiers of these APIs are rate limited, so repeated presses of Fetch spend quota.
- `done()` runs on the Swing event thread and creates a second `Service` only to build the Wikipedia URL. The constructor scans all locales (see section 5), so that work happens on the UI thread.
- NBP returns HTTP 404 for a currency that is not in table A. The code catches the exception, prints a stack trace, and moves on to table B, so a normal lookup for such a currency produces stack traces on stderr. Control flow by exception is noisy and slow; checking the status code is cleaner.

### 4. Parsing and error signalling
- JSON is read with string scanning (`indexOf`, `substring`, `split`). It works on the compact JSON these APIs return today, and breaks on whitespace changes, escaped quotes or nested objects (the Fixer `rates` block is found by searching for the exact text `"rates":{`). Every key is the **first** match in the document.
- A missing key ends up as `N/A` in the output, so a structural change in a response looks like missing data instead of an error.
- Errors are signalled in-band: `getWeather` returns a JSON-looking string `{"error":"..."}` built from the exception message without escaping, and the caller detects it with `contains("\"error\"")`.
- Empty `catch (Exception e) {}` blocks in the country lookup hide failures completely.

### 5. Country to currency mapping
- `Service` scans `Locale.getAvailableLocales()` (on the order of a thousand entries) once for the country code and once for the currency, comparing the English display country name of each locale until one matches (a full scan when nothing matches). The lookup is O(L) per call, and a `Service` is constructed twice per Fetch. A static `Map` built once turns it into O(1).
- The input must match the JDK's English display name (`United States`, not `USA`). If nothing matches, the currency silently **defaults to USD** and the country code to an empty string, so a typo in the country name produces a plausible-looking result for the wrong currency. An unknown country should be an error.
- The target currency is not validated (it is only upper-cased), so an empty or malformed code goes straight into the Fixer request.

### 6. Secrets and error paths
- API keys are constants in source code, which makes it easy to commit a real key by accident. Reading them from environment variables keeps them out of the repository.
- Keys are sent as URL query parameters (`appid`, `access_key`). When an HTTP request fails, `HttpURLConnection` puts the full URL in the exception message. The weather error path shows that message in the results area and the Fixer error path prints it to stderr, so the key can end up on screen, in screenshots and in logs.

### 7. Structure and testability
- There are no tests. `Service` creates its own HTTP connections and mixes three concerns (transport, parsing, country lookup), so nothing can be tested without network access.
- The parsing and the cross-rate arithmetic are pure functions of a JSON string and can be tested offline with fixture responses. A small interface for the HTTP layer would allow the rest to be tested with a fake.
- The output text is assembled inside `SwingWorker.doInBackground`, so presentation and data fetching are coupled in the UI class.
