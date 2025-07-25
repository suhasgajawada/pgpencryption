Here’s a powerful and precise GitHub Copilot prompt you can paste into a new file (e.g., `README.md` or just start typing it in your `App.java`):

---

### 🧠 GitHub Copilot Prompt (Java + Spring Boot PDF Processor)

```java
// Build a Spring Boot 2.3.1.RELEASE application (Java 8) that:
// 1. Every 15 minutes, reads all PDF files from a configured directory
// 2. Extracts specific, configurable fields (like "Customer Name", "Account Number", etc.) from each PDF
// 3. Constructs a JSON object from these fields
// 4. Sends the JSON to a configured downstream REST endpoint using RestTemplate
// 
// Requirements:
// - All configurable properties like schedule interval, field names, file path, and downstream URL should be in application.yaml
// - Use Apache PDFBox for PDF text extraction
// - Use Apache Commons Logging (no SLF4J or Logback directly)
// - Do not use Lombok
// - Use standard @Service, @Controller, @Configuration components
// - Include a REST controller with a `/trigger` endpoint to manually invoke the processing job
// - Provide meaningful logging using Apache Commons Logging
// - Ignore .pdf files that fail to parse, but log them
// - Use `@Scheduled(cron = "...")` for 15 min polling
// - Directory should be `src/main/resources/pdf`
// - No security, no database, no external dependencies beyond Spring Boot Starter, PDFBox, Commons Logging
//
// Start by writing the pom.xml with correct dependencies and Java 8 setup.
```
