Great. Below is the updated code for **parallel journey-based processing** using `@Async` in `README.md` style. This version:

* Loads multiple journeys from `application.yaml`
* Uses a thread pool to process each journey in parallel
* Handles retries and file movement logic as you described
* Built with Java 8 + Spring Boot 2.3.1 (fully compatible)

---

## 📁 `src/main/java/com/example/pdfprocessor/PdfProcessorApplication.java`

```java
package com.example.pdfprocessor;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.EnableAsync;

@SpringBootApplication
@EnableScheduling
@EnableAsync
public class PdfProcessorApplication {
    public static void main(String[] args) {
        SpringApplication.run(PdfProcessorApplication.class, args);
    }
}
```

---

## 📁 `src/main/java/com/example/pdfprocessor/config/AsyncConfig.java`

```java
package com.example.pdfprocessor.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Bean;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "journeyExecutor")
    public Executor journeyExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5); // number of parallel journeys allowed
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("JourneyExecutor-");
        executor.initialize();
        return executor;
    }
}
```

---

## 📁 `src/main/java/com/example/pdfprocessor/service/JourneyProcessorService.java`

```java
package com.example.pdfprocessor.service;

import com.example.pdfprocessor.model.JourneyConfig;
import com.example.pdfprocessor.util.RetryUtils;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.Resource;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.io.File;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.List;

@Service
public class JourneyProcessorService {

    private static final Log log = LogFactory.getLog(JourneyProcessorService.class);

    private final FileUploadService fileUploadService;
    private final PdfProcessingService pdfProcessingService;

    public JourneyProcessorService(FileUploadService fileUploadService,
                                   PdfProcessingService pdfProcessingService) {
        this.fileUploadService = fileUploadService;
        this.pdfProcessingService = pdfProcessingService;
    }

    @Async("journeyExecutor")
    public void processJourney(JourneyConfig journey) {
        try {
            File sourceDir = new File(journey.getSourcePath());
            File[] files = sourceDir.listFiles((dir, name) -> name.toLowerCase().endsWith(".pdf"));

            if (files == null || files.length == 0) {
                log.info("No files found for journey: " + journey.getCountry());
                return;
            }

            for (File file : files) {
                try {
                    boolean uploaded = fileUploadService.uploadFileToFilenet(file, journey);
                    if (!uploaded) {
                        log.warn("Upload failed. Will retry for file: " + file.getName());
                        continue; // do not process or move
                    }

                    String json = pdfProcessingService.extractAndBuildJson(file, journey);
                    boolean sent = RetryUtils.retry(() ->
                        pdfProcessingService.sendJsonToDownstream(json, journey), 3);

                    if (sent) {
                        Files.move(file.toPath(), Paths.get(journey.getProcessedPath(), file.getName()));
                        log.info("File processed successfully: " + file.getName());
                    } else {
                        log.warn("Downstream send failed for file: " + file.getName());
                    }

                } catch (Exception e) {
                    log.error("Processing failed for file: " + file.getName(), e);
                    Files.move(file.toPath(), Paths.get(journey.getFailedPath(), file.getName()));
                }
            }

        } catch (Exception e) {
            log.error("Failed to process journey: " + journey.getCountry(), e);
        }
    }
}
```

---

## 📁 `src/main/java/com/example/pdfprocessor/service/SchedulerService.java`

```java
package com.example.pdfprocessor.service;

import com.example.pdfprocessor.model.JourneyConfig;
import com.example.pdfprocessor.config.AppConfig;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class SchedulerService {

    private final AppConfig appConfig;
    private final JourneyProcessorService journeyProcessorService;

    public SchedulerService(AppConfig appConfig,
                            JourneyProcessorService journeyProcessorService) {
        this.appConfig = appConfig;
        this.journeyProcessorService = journeyProcessorService;
    }

    @Scheduled(cron = "${schedule.cron}")
    public void triggerJourneyProcessing() {
        List<JourneyConfig> journeys = appConfig.getJourneys();
        for (JourneyConfig journey : journeys) {
            journeyProcessorService.processJourney(journey);
        }
    }
}
```

---

## 📁 `src/main/resources/application.yaml`

```yaml
schedule:
  cron: "0 */15 * * * *"  # Every 15 minutes

journeys:
  - country: USA
    sourcePath: "./data/usa/source"
    processedPath: "./data/usa/processed"
    failedPath: "./data/usa/failed"
    filenetUrl: "https://filenet-usa.internal/api/upload"
    downstreamUrl: "https://downstream-usa.internal/api/process"
    filenetMeta:
      documentType: "Form"
      department: "Finance"
    jsonFields:
      customerName: "Customer Name:"
      accountNumber: "Account Number:"
      formType: "Form Type:"
  - country: INDIA
    sourcePath: "./data/india/source"
    processedPath: "./data/india/processed"
    failedPath: "./data/india/failed"
    filenetUrl: "https://filenet-india.internal/api/upload"
    downstreamUrl: "https://downstream-india.internal/api/process"
    filenetMeta:
      documentType: "KYC"
      department: "Compliance"
    jsonFields:
      customerName: "नाम:"
      accountNumber: "खाता संख्या:"
      formType: "फॉर्म प्रकार:"
```

---

## 📁 `src/main/java/com/example/pdfprocessor/config/AppConfig.java`

```java
package com.example.pdfprocessor.config;

import com.example.pdfprocessor.model.JourneyConfig;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

import java.util.List;

@Configuration
@ConfigurationProperties
public class AppConfig {

    private List<JourneyConfig> journeys;

    public List<JourneyConfig> getJourneys() {
        return journeys;
    }

    public void setJourneys(List<JourneyConfig> journeys) {
        this.journeys = journeys;
    }
}
```

---

## 📁 `src/main/java/com/example/pdfprocessor/model/JourneyConfig.java`

```java
package com.example.pdfprocessor.model;

import java.util.Map;

public class JourneyConfig {
    private String country;
    private String sourcePath;
    private String processedPath;
    private String failedPath;
    private String filenetUrl;
    private String downstreamUrl;
    private FilenetMeta filenetMeta;
    private Map<String, String> jsonFields;

    // Getters and Setters
    public String getCountry() { return country; }
    public void setCountry(String country) { this.country = country; }

    public String getSourcePath() { return sourcePath; }
    public void setSourcePath(String sourcePath) { this.sourcePath = sourcePath; }

    public String getProcessedPath() { return processedPath; }
    public void setProcessedPath(String processedPath) { this.processedPath = processedPath; }

    public String getFailedPath() { return failedPath; }
    public void setFailedPath(String failedPath) { this.failedPath = failedPath; }

    public String getFilenetUrl() { return filenetUrl; }
    public void setFilenetUrl(String filenetUrl) { this.filenetUrl = filenetUrl; }

    public String getDownstreamUrl() { return downstreamUrl; }
    public void setDownstreamUrl(String downstreamUrl) { this.downstreamUrl = downstreamUrl; }

    public FilenetMeta getFilenetMeta() { return filenetMeta; }
    public void setFilenetMeta(FilenetMeta filenetMeta) { this.filenetMeta = filenetMeta; }

    public Map<String, String> getJsonFields() { return jsonFields; }
    public void setJsonFields(Map<String, String> jsonFields) { this.jsonFields = jsonFields; }
}
```

---

## 📁 `src/main/java/com/example/pdfprocessor/model/FilenetMeta.java`

```java
package com.example.pdfprocessor.model;

public class FilenetMeta {
    private String documentType;
    private String department;

    public String getDocumentType() { return documentType; }
    public void setDocumentType(String documentType) { this.documentType = documentType; }

    public String getDepartment() { return department; }
    public void setDepartment(String department) { this.department = department; }
}
```

---

## 📁 `src/main/java/com/example/pdfprocessor/service/FileUploadService.java`

```java
package com.example.pdfprocessor.service;

import com.example.pdfprocessor.model.JourneyConfig;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.io.File;

@Service
public class FileUploadService {

    private static final Log log = LogFactory.getLog(FileUploadService.class);
    private final RestTemplate restTemplate = new RestTemplate();

    public boolean uploadFileToFilenet(File file, JourneyConfig journey) {
        try {
            // mock call - replace with actual multipart/form-data logic
            log.info("Uploading " + file.getName() + " to Filenet at " + journey.getFilenetUrl());
            return true;
        } catch (Exception e) {
            log.error("Upload to Filenet failed", e);
            return false;
        }
    }
}
```

---

## 📁 `src/main/java/com/example/pdfprocessor/service/PdfProcessingService.java`

```java
package com.example.pdfprocessor.service;

import com.example.pdfprocessor.model.JourneyConfig;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.io.File;
import java.nio.file.Files;
import java.util.HashMap;
import java.util.Map;

@Service
public class PdfProcessingService {

    private static final Log log = LogFactory.getLog(PdfProcessingService.class);
    private final RestTemplate restTemplate = new RestTemplate();

    public String extractAndBuildJson(File file, JourneyConfig journey) throws Exception {
        // Simulated text extraction
        String content = new String(Files.readAllBytes(file.toPath()));

        Map<String, String> jsonMap = new HashMap<>();
        for (Map.Entry<String, String> entry : journey.getJsonFields().entrySet()) {
            String key = entry.getKey();
            String marker = entry.getValue();
            String extracted = extractValue(content, marker);
            jsonMap.put(key, extracted);
        }

        return new ObjectMapper().writeValueAsString(jsonMap);
    }

    public boolean sendJsonToDownstream(String json, JourneyConfig journey) {
        try {
            restTemplate.postForEntity(journey.getDownstreamUrl(), json, String.class);
            return true;
        } catch (Exception e) {
            log.error("Failed to send JSON to downstream", e);
            return false;
        }
    }

    private String extractValue(String content, String marker) {
        int index = content.indexOf(marker);
        if (index == -1) return "";
        int start = index + marker.length();
        int end = content.indexOf("\n", start);
        return content.substring(start, end == -1 ? content.length() : end).trim();
    }
}
```

---

## 📁 `src/main/java/com/example/pdfprocessor/util/RetryUtils.java`

```java
package com.example.pdfprocessor.util;

import java.util.function.Supplier;

public class RetryUtils {

    public static boolean retry(Supplier<Boolean> action, int maxAttempts) {
        int attempts = 0;
        while (attempts < maxAttempts) {
            if (action.get()) return true;
            attempts++;
        }
        return false;
    }
}
```

---

Let me know if you want:

* `logback-spring.xml` for structured logging
* A JUnit test class
* A Copilot prompt to regenerate this project anytime
* Or a single markdown-concatenated file

This is now **multi-journey, parallel, resilient, and fully configurable**.
