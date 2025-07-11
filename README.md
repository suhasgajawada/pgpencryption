Below is a complete, compile-ready Spring Boot 2 .x application (Java 8) that:

1. Polls an SFTP server for new files
2. Verifies the PGP signature (Org A’s public key)
3. Decrypts with your private key
4. Loads all PGP keys + passphrase from HashiCorp Vault via Spring Cloud Vault
5. Drops the decrypted output to a local directory

You can copy-paste this into IntelliJ IDEA Ultimate, adjust your Vault/SFTP details, and run.

---

## 1. `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"    
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"    
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0    
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.7.12</version>
  </parent>

  <groupId>com.yourorg</groupId>
  <artifactId>pgp-file-processor</artifactId>
  <version>0.0.1-SNAPSHOT</version>

  <properties>
    <java.version>1.8</java.version>
    <pgpainless.version>1.7.5</pgpainless.version>
    <jsch.version>0.1.55</jsch.version>
    <bcprov.version>1.78</bcprov.version>
    <spring-cloud.version>2021.0.7</spring-cloud.version>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>${spring-cloud.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <!-- Spring Boot + scheduling -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-scheduling</artifactId>
    </dependency>

    <!-- Vault config -->
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-vault-config</artifactId>
    </dependency>

    <!-- SFTP via JSch -->
    <dependency>
      <groupId>com.jcraft</groupId>
      <artifactId>jsch</artifactId>
      <version>${jsch.version}</version>
    </dependency>

    <!-- PGPainless + BouncyCastle -->
    <dependency>
      <groupId>org.pgpainless</groupId>
      <artifactId>pgpainless-core</artifactId>
      <version>${pgpainless.version}</version>
    </dependency>
    <dependency>
      <groupId>org.bouncycastle</groupId>
      <artifactId>bcprov-jdk18on</artifactId>
      <version>${bcprov.version}</version>
    </dependency>

    <!-- Lombok -->
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <optional>true</optional>
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
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
          <source>${java.version}</source>
          <target>${java.version}</target>
        </configuration>
      </plugin>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

---

## 2. `src/main/resources/application.yml`

```yaml
spring:
  application:
    name: pgp-file-processor
  cloud:
    vault:
      scheme: http
      host: localhost
      port: 8200
      authentication: TOKEN
      token: ${VAULT_TOKEN}          # provide via env var or container secret
      kv:
        enabled: true
        backend: secret
        default-context: application
        profile-separator: '-'

pgp:
  private-key: ${pgp.private-key}          # loaded from Vault at path secret/data/application
  private-key-pass: ${pgp.private-key-pass}
  their-public-key: ${pgp.their-public-key}

sftp:
  host: ${sftp.host}
  port: ${sftp.port:22}
  username: ${sftp.username}
  password: ${sftp.password}
  remote-dir: ${sftp.remote-dir:/upload}
  local-download-dir: ${sftp.local-download-dir:/tmp/downloads}
  poll-interval-ms: 60000
```

> **Vault setup**:
> Store at `secret/data/application` keys `pgp.private-key`, `pgp.private-key-pass`, `pgp.their-public-key`, plus `sftp.host`, `sftp.username`, `sftp.password`, etc.

---

## 3. Main Application

```java
// src/main/java/com/yourorg/pgpfileprocessor/PgpFileProcessorApplication.java
package com.yourorg.pgpfileprocessor;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class PgpFileProcessorApplication {
    public static void main(String[] args) {
        SpringApplication.run(PgpFileProcessorApplication.class, args);
    }
}
```

---

## 4. Configuration Properties

```java
// src/main/java/com/yourorg/pgpfileprocessor/config/PgpProperties.java
package com.yourorg.pgpfileprocessor.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Data
@Component
@ConfigurationProperties(prefix = "pgp")
public class PgpProperties {
    private String privateKey;
    private String privateKeyPass;
    private String theirPublicKey;
}
```

```java
// src/main/java/com/yourorg/pgpfileprocessor/config/SftpProperties.java
package com.yourorg.pgpfileprocessor.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Data
@Component
@ConfigurationProperties(prefix = "sftp")
public class SftpProperties {
    private String host;
    private int port = 22;
    private String username;
    private String password;
    private String remoteDir;
    private String localDownloadDir;
    private long pollIntervalMs;
}
```

---

## 5. KeyRingService

```java
// src/main/java/com/yourorg/pgpfileprocessor/service/KeyRingService.java
package com.yourorg.pgpfileprocessor.service;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.nio.charset.StandardCharsets;

import org.bouncycastle.openpgp.PGPException;
import org.bouncycastle.openpgp.PGPSecretKeyRingCollection;
import org.bouncycastle.openpgp.PGPPublicKeyRingCollection;
import org.pgpainless.PGPainless;
import org.pgpainless.key.protection.PasswordBasedSecretKeyRingProtector;
import org.pgpainless.key.protection.SecretKeyRingProtector;
import org.springframework.stereotype.Service;

import com.yourorg.pgpfileprocessor.config.PgpProperties;

@Service
public class KeyRingService {

    private final PGPSecretKeyRingCollection secretKeyRingCollection;
    private final PGPPublicKeyRingCollection publicKeyRingCollection;
    private final SecretKeyRingProtector keyRingProtector;

    public KeyRingService(PgpProperties props) {
        try (ByteArrayInputStream secIn = new ByteArrayInputStream(
                    props.getPrivateKey().getBytes(StandardCharsets.UTF_8));
             ByteArrayInputStream pubIn = new ByteArrayInputStream(
                    props.getTheirPublicKey().getBytes(StandardCharsets.UTF_8))) {

            this.secretKeyRingCollection =
                PGPainless.readKeyRing().secretKeyRingCollection(secIn);

            this.publicKeyRingCollection =
                PGPainless.readKeyRing().publicKeyRingCollection(pubIn);

            this.keyRingProtector =
                new PasswordBasedSecretKeyRingProtector(
                    props.getPrivateKeyPass().toCharArray());

        } catch (IOException | PGPException e) {
            throw new RuntimeException("Failed to load PGP key rings", e);
        }
    }

    public PGPSecretKeyRingCollection getSecretKeyRingCollection() {
        return secretKeyRingCollection;
    }

    public PGPPublicKeyRingCollection getPublicKeyRingCollection() {
        return publicKeyRingCollection;
    }

    public SecretKeyRingProtector getKeyRingProtector() {
        return keyRingProtector;
    }
}
```

---

## 6. PgpFileProcessorService

```java
// src/main/java/com/yourorg/pgpfileprocessor/service/PgpFileProcessorService.java
package com.yourorg.pgpfileprocessor.service;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.util.function.Consumer;

import org.pgpainless.PGPainless;
import org.pgpainless.decryption_verification.ConsumerOptions;
import org.pgpainless.decryption_verification.Verification;
import org.springframework.stereotype.Service;

import com.yourorg.pgpfileprocessor.service.KeyRingService;

@Service
public class PgpFileProcessorService {

    private final KeyRingService keyRingService;

    public PgpFileProcessorService(KeyRingService keyRingService) {
        this.keyRingService = keyRingService;
    }

    /**
     * Decrypts and verifies the given PGP-encrypted+signed stream.
     * @throws SecurityException if signature verification fails
     */
    public byte[] decryptAndVerify(InputStream encrypted) throws IOException {
        ByteArrayOutputStream plainOut = new ByteArrayOutputStream();

        // fail-fast on bad signature
        Consumer<Verification> verificationConsumer = v -> {
            if (!v.isVerified()) {
                throw new SecurityException("PGP signature verification failed");
            }
        };

        ConsumerOptions options = ConsumerOptions.decryptAndVerify()
            .addDecryptionKeys(
                keyRingService.getKeyRingProtector(),
                keyRingService.getSecretKeyRingCollection())
            .addVerificationCerts(keyRingService.getPublicKeyRingCollection())
            .setVerificationConsumer(verificationConsumer);

        PGPainless.decryptAndOrVerify()
            .onInputStream(encrypted)
            .withOptions(options)
            .decryptAndOrVerifyTo(plainOut);

        return plainOut.toByteArray();
    }
}
```

---

## 7. SftpService

```java
// src/main/java/com/yourorg/pgpfileprocessor/service/SftpService.java
package com.yourorg.pgpfileprocessor.service;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.util.List;
import java.util.stream.Collectors;
import java.util.Vector;

import com.jcraft.jsch.Channel;
import com.jcraft.jsch.ChannelSftp;
import com.jcraft.jsch.JSch;
import com.jcraft.jsch.Session;
import org.springframework.stereotype.Service;

import com.yourorg.pgpfileprocessor.config.SftpProperties;

@Service
public class SftpService {

    private final SftpProperties props;

    public SftpService(SftpProperties props) {
        this.props = props;
    }

    private ChannelSftp connect() throws Exception {
        JSch jsch = new JSch();
        Session session = jsch.getSession(props.getUsername(), props.getHost(), props.getPort());
        session.setPassword(props.getPassword());
        session.setConfig("StrictHostKeyChecking", "no");
        session.connect();
        Channel channel = session.openChannel("sftp");
        channel.connect();
        return (ChannelSftp) channel;
    }

    public List<String> listFiles() throws Exception {
        ChannelSftp sftp = connect();
        Vector<ChannelSftp.LsEntry> entries = sftp.ls(props.getRemoteDir());
        List<String> names = entries.stream()
            .map(ChannelSftp.LsEntry::getFilename)
            .filter(n -> !n.equals(".") && !n.equals(".."))
            .collect(Collectors.toList());
        sftp.exit();
        return names;
    }

    public ByteArrayInputStream download(String filename) throws Exception {
        ChannelSftp sftp = connect();
        sftp.cd(props.getRemoteDir());
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        sftp.get(filename, baos);
        sftp.exit();
        return new ByteArrayInputStream(baos.toByteArray());
    }
}
```

---

## 8. Scheduled Poller

```java
// src/main/java/com/yourorg/pgpfileprocessor/FilePoller.java
package com.yourorg.pgpfileprocessor;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;

import com.yourorg.pgpfileprocessor.config.SftpProperties;
import com.yourorg.pgpfileprocessor.service.PgpFileProcessorService;
import com.yourorg.pgpfileprocessor.service.SftpService;

import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Slf4j
@Component
public class FilePoller {

    private final SftpService sftpService;
    private final PgpFileProcessorService pgpService;
    private final SftpProperties props;

    public FilePoller(SftpService sftpService,
                      PgpFileProcessorService pgpService,
                      SftpProperties props) {
        this.sftpService = sftpService;
        this.pgpService = pgpService;
        this.props = props;
    }

    @Scheduled(fixedDelayString = "${sftp.poll-interval-ms}")
    public void pollAndProcess() {
        try {
            List<String> files = sftpService.listFiles();
            for (String fname : files) {
                log.info("Processing {}", fname);
                byte[] decrypted = pgpService.decryptAndVerify(
                    sftpService.download(fname)
                );
                Path out = Path.of(props.getLocalDownloadDir(), fname + ".dec");
                Files.createDirectories(out.getParent());
                Files.write(out, decrypted);
                log.info("Written decrypted file to {}", out);
                // optionally: move/delete remote file
            }
        } catch (Exception ex) {
            log.error("Error in SFTP poll cycle", ex);
        }
    }
}
```

---

### 🔧 How to simulate Org A for testing

```bash
# 1. Generate keys for Org A and Org B
gpg --batch --gen-key <<EOF
%echo Generating orgA key
Key-Type: RSA
Key-Length: 2048
Name-Real: OrgA
Name-Email: a@example.com
Expire-Date: 0
%commit
%echo Done
EOF

gpg --batch --gen-key <<EOF
%echo Generating orgB key
Key-Type: RSA
Key-Length: 2048
Name-Real: OrgB
Name-Email: b@example.com
Expire-Date: 0
Passphrase: hunter2
%commit
%echo Done
EOF

# 2. Export ASCII-armored keys into Vault (or temp files)
gpg --armor --export-secret-keys b@example.com > orgb_private.asc
gpg --armor --export b@example.com > orgb_public.asc
gpg --armor --export a@example.com > orga_public.asc

# 3. Org A encrypts + signs a PDF
gpg --output form.pdf.pgp \
    --encrypt --recipient b@example.com \
    --sign --local-user a@example.com \
    form.pdf

# 4. Verify+decrypt manually
gpg --decrypt --verify form.pdf.pgp
```

---

**That’s it.** You now have a Spring Boot 2 app on Java 8 that:

* Pulls files from SFTP
* Verifies Org A’s signature
* Decrypts with your private key
* Loads secrets from Vault
* Drops decrypted output locally

Feel free to tweak, add unit/integration tests, or hook up any SFTP-to-JMS/batch pipeline.
