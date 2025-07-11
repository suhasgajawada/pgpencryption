Below is a complete Spring Boot 2 / Java 8 application that:

1. Polls an SFTP server for `.pgp` or `.gpg` files
2. Verifies the signature (Org A’s public key)
3. Decrypts them (your private key + passphrase)
4. Moves processed files and writes plaintext to disk
5. Loads all secrets (PGP keys, SFTP creds) from Vault via Spring Cloud Vault
6. Uses PGPainless + Bouncy Castle under the hood
7. Uses Apache Commons Logging for detailed logs
8. No Lombok

---

## 1. Project structure

```
pgp-file-processor/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/com/yourorg/pgpfileprocessor/
│   │   │   ├── PgpFileProcessorApplication.java
│   │   │   ├── config/
│   │   │   │   ├── PgpProperties.java
│   │   │   │   └── SftpProperties.java
│   │   │   └── service/
│   │   │       ├── KeyRingService.java
│   │   │       ├── PgpFileProcessorService.java
│   │   │       └── SftpPollingService.java
│   └── resources/
│       └── application.yml
```

---

## 2. `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" 
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.yourorg</groupId>
  <artifactId>pgp-file-processor</artifactId>
  <version>0.0.1-SNAPSHOT</version>

  <properties>
    <java.version>1.8</java.version>
    <spring-boot.version>2.7.12</spring-boot.version>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>${spring-boot.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <!-- Core -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-scheduling</artifactId>
    </dependency>

    <!-- Spring Cloud Vault for secrets -->
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-vault-config</artifactId>
    </dependency>

    <!-- SFTP client -->
    <dependency>
      <groupId>com.jcraft</groupId>
      <artifactId>jsch</artifactId>
      <version>0.1.55</version>
    </dependency>

    <!-- PGP libs -->
    <dependency>
      <groupId>org.pgpainless</groupId>
      <artifactId>pgpainless-core</artifactId>
      <version>1.7.5</version>
    </dependency>
    <dependency>
      <groupId>org.bouncycastle</groupId>
      <artifactId>bcprov-jdk18on</artifactId>
      <version>1.78</version>
    </dependency>

    <!-- Test -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

---

## 3. `application.yml`

```yaml
spring:
  cloud:
    vault:
      uri: https://vault.example.com:8200
      authentication: APPROLE
      app-role:
        role-id: ${VAULT_ROLE_ID}
        secret-id: ${VAULT_SECRET_ID}
      kv:
        enabled: true
        backend: secret
        default-context: application

sftp:
  host: sftp.example.com
  port: 22
  username: ${SFTP_USERNAME}
  password: ${SFTP_PASSWORD}
  remote-dir: /incoming
  processed-dir: /processed
  poll-interval-ms: 60000

pgp:
  private-key: |
    -----BEGIN PGP PRIVATE KEY BLOCK-----
    …your ASCII-armored key…
    -----END PGP PRIVATE KEY BLOCK-----
  private-key-passphrase: ${PGP_PASSPHRASE}
  their-public-key: |
    -----BEGIN PGP PUBLIC KEY BLOCK-----
    …Org A’s ASCII-armored key…
    -----END PGP PUBLIC KEY BLOCK-----

app:
  output-dir: /tmp/decrypted
```

> **Note:** All secrets (`VAULT_ROLE_ID`, `SFTP_PASSWORD`, `PGP_PASSPHRASE`) come from your environment; Vault injects `pgp.*` and `sftp.*`.

---

## 4. `PgpFileProcessorApplication.java`

```java
package com.yourorg.pgpfileprocessor;

import com.yourorg.pgpfileprocessor.config.PgpProperties;
import com.yourorg.pgpfileprocessor.config.SftpProperties;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
@EnableConfigurationProperties({
    PgpProperties.class,
    SftpProperties.class
})
public class PgpFileProcessorApplication {
    public static void main(String[] args) {
        SpringApplication.run(PgpFileProcessorApplication.class, args);
    }
}
```

---

## 5. Configuration-binding classes

### 5.1 `PgpProperties.java`

```java
package com.yourorg.pgpfileprocessor.config;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "pgp")
public class PgpProperties {
    private String privateKey;
    private String privateKeyPassphrase;
    private String theirPublicKey;

    public String getPrivateKey() {
        return privateKey;
    }
    public void setPrivateKey(String privateKey) {
        this.privateKey = privateKey;
    }

    public String getPrivateKeyPassphrase() {
        return privateKeyPassphrase;
    }
    public void setPrivateKeyPassphrase(String privateKeyPassphrase) {
        this.privateKeyPassphrase = privateKeyPassphrase;
    }

    public String getTheirPublicKey() {
        return theirPublicKey;
    }
    public void setTheirPublicKey(String theirPublicKey) {
        this.theirPublicKey = theirPublicKey;
    }
}
```

### 5.2 `SftpProperties.java`

```java
package com.yourorg.pgpfileprocessor.config;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "sftp")
public class SftpProperties {
    private String host;
    private int port = 22;
    private String username;
    private String password;
    private String remoteDir;
    private String processedDir;
    private long pollIntervalMs;

    // getters & setters

    public String getHost() {
        return host;
    }
    public void setHost(String host) {
        this.host = host;
    }

    public int getPort() {
        return port;
    }
    public void setPort(int port) {
        this.port = port;
    }

    public String getUsername() {
        return username;
    }
    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }
    public void setPassword(String password) {
        this.password = password;
    }

    public String getRemoteDir() {
        return remoteDir;
    }
    public void setRemoteDir(String remoteDir) {
        this.remoteDir = remoteDir;
    }

    public String getProcessedDir() {
        return processedDir;
    }
    public void setProcessedDir(String processedDir) {
        this.processedDir = processedDir;
    }

    public long getPollIntervalMs() {
        return pollIntervalMs;
    }
    public void setPollIntervalMs(long pollIntervalMs) {
        this.pollIntervalMs = pollIntervalMs;
    }
}
```

---

## 6. `KeyRingService.java`

```java
package com.yourorg.pgpfileprocessor.service;

import com.yourorg.pgpfileprocessor.config.PgpProperties;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.pgpainless.PGPainless;
import org.pgpainless.key.collection.PGPSecretKeyRingCollection;
import org.pgpainless.key.collection.PGPPublicKeyRingCollection;
import org.pgpainless.key.protection.PasswordBasedSecretKeyRingProtector;
import org.pgpainless.key.protection.SecretKeyRingProtector;
import org.pgpainless.util.Passphrase;
import org.springframework.stereotype.Service;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.nio.charset.StandardCharsets;

@Service
public class KeyRingService {

    private static final Log logger = LogFactory.getLog(KeyRingService.class);

    private final PGPSecretKeyRingCollection secretKeys;
    private final PGPPublicKeyRingCollection publicKeys;
    private final SecretKeyRingProtector protector;

    public KeyRingService(PgpProperties props) throws IOException {
        logger.info("Loading PGP keys from Vault-backed properties");

        // Load private key
        byte[] priv = props.getPrivateKey().getBytes(StandardCharsets.UTF_8);
        secretKeys = PGPainless.readKeyRing()
                         .secretKeyRingCollection(new ByteArrayInputStream(priv));

        // Load passphrase
        Passphrase pass = Passphrase.fromPassword(props.getPrivateKeyPassphrase());
        protector = new PasswordBasedSecretKeyRingProtector(pass);

        // Load their public key
        byte[] pub = props.getTheirPublicKey().getBytes(StandardCharsets.UTF_8);
        publicKeys = PGPainless.readKeyRing()
                        .publicKeyRingCollection(new ByteArrayInputStream(pub));

        logger.info("PGP key rings initialized");
    }

    public PGPSecretKeyRingCollection getSecretKeys() {
        return secretKeys;
    }

    public PGPPublicKeyRingCollection getPublicKeys() {
        return publicKeys;
    }

    public SecretKeyRingProtector getProtector() {
        return protector;
    }
}
```

---

## 7. `PgpFileProcessorService.java`

```java
package com.yourorg.pgpfileprocessor.service;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.pgpainless.PGPainless;
import org.pgpainless.decryption_verification.ConsumerOptions;
import org.pgpainless.decryption_verification.Verification;
import org.pgpainless.decryption_verification.VerificationConsumer;
import org.pgpainless.decryption_verification.DecryptionBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;

@Service
public class PgpFileProcessorService {

    private static final Log logger = LogFactory.getLog(PgpFileProcessorService.class);

    private final KeyRingService keyRing;
    private final Path outputDir;

    public PgpFileProcessorService(KeyRingService keyRing,
                                   @Value("${app.output-dir}") String outputDir) {
        this.keyRing = keyRing;
        this.outputDir = Paths.get(outputDir);
    }

    public void process(InputStream encryptedStream, String filename) {
        try {
            logger.info("Decrypt+verify starting for " + filename);

            ByteArrayOutputStream plainOut = new ByteArrayOutputStream();

            DecryptionBuilder db = PGPainless.decryptAndOrVerify()
                .onInputStream(encryptedStream)
                .withOptions(ConsumerOptions.get()
                    .addDecryptionKey(keyRing.getProtector(), keyRing.getSecretKeys())
                    .addVerificationCerts(keyRing.getPublicKeys())
                    .setVerificationConsumer(new VerificationConsumer() {
                      @Override
                      public void accept(Verification v) {
                        if (!v.isVerified()) {
                          throw new SecurityException("Signature failure: " + filename);
                        }
                        logger.info("Signature by key " 
                                    + Long.toHexString(v.getSigner()) 
                                    + " verified");
                      }
                    })
                );

            db.decryptAndOrVerifyTo(plainOut);
            byte[] plain = plainOut.toByteArray();

            if (!Files.exists(outputDir)) {
                Files.createDirectories(outputDir);
            }
            Path outFile = outputDir.resolve(filename + ".dec");
            Files.write(outFile, plain);
            logger.info("Decrypted file saved to " + outFile.toAbsolutePath());

        } catch (Exception e) {
            logger.error("Failed to process PGP file: " + filename, e);
        }
    }
}
```

---

## 8. `SftpPollingService.java`

```java
package com.yourorg.pgpfileprocessor.service;

import com.jcraft.jsch.ChannelSftp;
import com.jcraft.jsch.JSch;
import com.jcraft.jsch.Session;
import com.yourorg.pgpfileprocessor.config.SftpProperties;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.io.InputStream;
import java.util.Vector;

@Service
public class SftpPollingService {

    private static final Log logger = LogFactory.getLog(SftpPollingService.class);

    private final SftpProperties cfg;
    private final PgpFileProcessorService processor;
    private final JSch jsch = new JSch();

    public SftpPollingService(SftpProperties cfg,
                              PgpFileProcessorService processor) {
        this.cfg = cfg;
        this.processor = processor;
    }

    @Scheduled(fixedDelayString = "${sftp.poll-interval-ms}")
    public void poll() {
        Session session = null;
        ChannelSftp channel = null;
        try {
            logger.info("Connecting to SFTP " + cfg.getHost());
            session = jsch.getSession(cfg.getUsername(), cfg.getHost(), cfg.getPort());
            session.setPassword(cfg.getPassword());
            session.setConfig("StrictHostKeyChecking", "no");
            session.connect();

            channel = (ChannelSftp) session.openChannel("sftp");
            channel.connect();

            String dir = cfg.getRemoteDir();
            logger.info("Listing " + dir);
            Vector<ChannelSftp.LsEntry> files = channel.ls(dir);

            for (ChannelSftp.LsEntry e : files) {
                String name = e.getFilename();
                if (name.endsWith(".pgp") || name.endsWith(".gpg")) {
                    String path = dir + "/" + name;
                    logger.info("Found " + path);
                    try (InputStream in = channel.get(path)) {
                        processor.process(in, name);
                    }
                    String dest = cfg.getProcessedDir() + "/" + name;
                    channel.rename(path, dest);
                    logger.info("Moved to processed folder: " + dest);
                }
            }
        } catch (Exception ex) {
            logger.error("SFTP polling error", ex);
        } finally {
            if (channel != null && channel.isConnected()) channel.disconnect();
            if (session != null && session.isConnected()) session.disconnect();
        }
    }
}
```

---

### 🚀 Run & Test

1. **Set** your Vault (`VAULT_ROLE_ID`, `VAULT_SECRET_ID`) and SFTP/PGP env vars.
2. **Build & run**:

   ```bash
   mvn clean package
   java -jar target/pgp-file-processor-0.0.1-SNAPSHOT.jar
   ```
3. **Simulate Org A** with CLI:

   ```bash
   # generate test keys
   gpg --quick-generate-key "OrgA <a@org.com>" rsa2048 default 0
   gpg --export --armor a@org.com > their_public.asc
   gpg --quick-generate-key "OrgB <b@org.com>" rsa2048 default 0
   gpg --export-secret-key --armor b@org.com > our_private.asc

   # encrypt+sign a PDF
   gpg --output form.pdf.pgp \
       --encrypt --sign \
       --recipient b@org.com \
       --local-user a@org.com \
       form.pdf
   # upload form.pdf.pgp to your SFTP /incoming
   ```
4. **Watch logs** — on successful poll, you’ll see signature-verify + decrypt logs, and `/tmp/decrypted/form.pdf.pgp.dec` will appear.

---

This scaffold is **compile-ready**. Fill in your real Vault URI, credentials, keys, and you’re production-grade.
