Perfect. Let’s walk through each piece of your Spring Boot application. You can copy-paste this entire content into your `README.md` for full reproducibility.

---

## 🧾 Overview

**Goal:**

* Every 15 minutes:

  * Read PDF files from a folder
  * Extract dynamic fields (like customer name, account number)
  * Build a JSON request
  * Send to downstream API using `RestTemplate`
* All configuration is in `application.yaml`

---

## 📁 Project Structure

```
📁 src/
 └── 📁 main/
     ├── 📁 java/com/example/pdfprocessor/
     │    ├── 📁 config/
     │    │    └── AppConfig.java
     │    ├── 📁 controller/
     │    │    └── TriggerController.java
     │    ├── 📁 service/
     │    │    └── PdfProcessingService.java
     │    └── 📁 util/
     │         └── PdfExtractor.java
     └── 📁 resources/
          ├── application.yaml
          └── logback-spring.xml
📄 pom.xml
```

---

## 📄 `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>pdfprocessor</artifactId>
  <version>1.0.0</version>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.3.1.RELEASE</version>
  </parent>

  <properties>
    <java.version>1.8</java.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
      <groupId>org.apache.pdfbox</groupId>
      <artifactId>pdfbox</artifactId>
      <version>2.0.24</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-scheduling</artifactId>
    </dependency>
    <dependency>
      <groupId>commons-logging</groupId>
      <artifactId>commons-logging</artifactId>
      <version>1.2</version>
    </dependency>
  </dependencies>

</project>
```

---

## 📄 `application.yaml`

```yaml
app:
  pdf:
    directory: src/main/resources/pdf
    fields:
      - Customer Name
      - Account Number
      - Form Name
    downstream-url: http://localhost:8081/api/process
  scheduler:
    cron: "0 0/15 * * * *"  # Every 15 mins
```

---

## 📄 `AppConfig.java`

```java
package com.example.pdfprocessor.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class AppConfig {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

---

## 📄 `TriggerController.java`

```java
package com.example.pdfprocessor.controller;

import com.example.pdfprocessor.service.PdfProcessingService;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class TriggerController {

    private final PdfProcessingService service;

    public TriggerController(PdfProcessingService service) {
        this.service = service;
    }

    @GetMapping("/trigger")
    public String triggerNow() {
        service.processAllPdfs();
        return "Triggered PDF processing.";
    }
}
```

---

## 📄 `PdfProcessingService.java`

```java
package com.example.pdfprocessor.service;

import com.example.pdfprocessor.util.PdfExtractor;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.io.File;
import java.util.*;

@Service
public class PdfProcessingService {

    private final RestTemplate restTemplate;
    private final List<String> fields;
    private final String pdfDirectory;
    private final String downstreamUrl;
    private static final Log logger = LogFactory.getLog(PdfProcessingService.class);

    public PdfProcessingService(RestTemplate restTemplate,
                                @Value("${app.pdf.fields}") List<String> fields,
                                @Value("${app.pdf.directory}") String pdfDirectory,
                                @Value("${app.pdf.downstream-url}") String downstreamUrl) {
        this.restTemplate = restTemplate;
        this.fields = fields;
        this.pdfDirectory = pdfDirectory;
        this.downstreamUrl = downstreamUrl;
    }

    @Scheduled(cron = "${app.scheduler.cron}")
    public void processAllPdfs() {
        File dir = new File(pdfDirectory);
        if (!dir.exists() || !dir.isDirectory()) {
            logger.warn("PDF directory does not exist.");
            return;
        }

        File[] files = dir.listFiles((d, name) -> name.toLowerCase().endsWith(".pdf"));
        if (files == null) return;

        for (File pdf : files) {
            try {
                Map<String, String> data = PdfExtractor.extractFields(pdf, fields);
                logger.info("Extracted: " + data);
                restTemplate.postForObject(downstreamUrl, data, String.class);
                logger.info("Sent to downstream.");
            } catch (Exception e) {
                logger.error("Error processing " + pdf.getName(), e);
            }
        }
    }
}
```

---

## 📄 `PdfExtractor.java`

```java
package com.example.pdfprocessor.util;

import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.text.PDFTextStripper;

import java.io.File;
import java.util.*;

public class PdfExtractor {

    public static Map<String, String> extractFields(File file, List<String> fieldNames) throws Exception {
        Map<String, String> extracted = new HashMap<>();

        try (PDDocument document = PDDocument.load(file)) {
            String text = new PDFTextStripper().getText(document);
            for (String field : fieldNames) {
                String value = extractFieldValue(text, field);
                extracted.put(field, value);
            }
        }

        return extracted;
    }

    private static String extractFieldValue(String text, String field) {
        int index = text.indexOf(field);
        if (index == -1) return "NOT_FOUND";

        int start = index + field.length();
        int end = Math.min(start + 100, text.length()); // Read next 100 chars

        String snippet = text.substring(start, end).split("\n")[0].trim();
        return snippet.split(" ")[0]; // crude assumption
    }
}
```

---

## 📄 `logback-spring.xml` (Optional)

```xml
<configuration>
    <include resource="org/springframework/boot/logging/logback/base.xml"/>
    <logger name="com.example.pdfprocessor" level="DEBUG"/>
</configuration>
```

---

Let me know if you want:

* PDF format-specific parsing logic (form fields, checkboxes)
* Test cases or mocks
* Externalized JSON structure instead of key-value map

Ready to scale it out with dynamic configs or real SFTP integration when you're ready.
