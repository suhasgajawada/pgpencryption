Here’s your **GitHub Copilot prompt** to regenerate the exact project setup from scratch, with all the right expectations:

---

### ✅ Copilot Prompt (Paste in VSCode or Copilot Chat)

```txt
Build a Java 8 Spring Boot 2.3.1 application called `pdf-journey-processor`. It should:

1. Use a `@Scheduled` cron job that runs every 15 minutes (configurable in `application.yaml`).
2. Load multiple "journeys" from `application.yaml`. Each journey represents a country and includes:
    - `sourcePath`: Path to pick up `.pdf` files
    - `processedPath`: Path to move files after success
    - `failedPath`: Path to move files after failure
    - `filenetUrl`: Endpoint to upload the file
    - `downstreamUrl`: Endpoint to send JSON data
    - `filenetMeta`: Additional metadata (documentType, department)
    - `jsonFields`: A map of JSON keys and PDF field markers to extract text
3. For each journey:
    - Process all `.pdf` files in its source path.
    - First upload the file to `filenetUrl` (simulate this with a dummy POST).
    - If upload succeeds, extract PDF text (simulate with `Files.readAllBytes`) and parse values based on markers in `jsonFields`.
    - Convert extracted values to JSON.
    - POST the JSON to the `downstreamUrl`.
    - If both steps succeed, move file to `processedPath`.
    - If file upload fails, leave the file in `sourcePath`.
    - If JSON extraction or sending fails, move to `failedPath`.
    - Retry downstream JSON send 3 times before failing.
4. Parallelize processing of journeys using `@Async` with a thread pool.
5. Use `RestTemplate` for HTTP calls.
6. Use Apache Commons Logging (`LogFactory.getLog`) instead of SLF4J or Lombok.
7. Avoid Lombok. Use plain Java beans.
8. Structure the code with clean packages: `config`, `model`, `service`, `util`.
9. Add a `RetryUtils` class to help with 3 retry attempts for downstream call.

```

---

This prompt is **stateless and reproducible** — it’ll let Copilot (or GPT-4/ChatGPT+VSCode) regenerate the entire project with correct logic, packages, and parallel handling. Use it when:

* Starting over
* Giving instructions to another developer
* Debugging by regenerating broken files
