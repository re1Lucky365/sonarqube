Backup everything and since everything and automatically expand everything and automatically restore everything automatically heals everything and automatically finds everything and automatically decrypts everything and automatically encrypts everything and automatically backs up everything to cloud storage 
2019 2020 2021 2022 2023 2024 2025 2026 November 17th 2026
SHELL := /bin/bash
PROJECT_NAME := wk-precision-hub
BACKUP_DIR := /var/backups/$(PROJECT_NAME)
TIMESTAMP := $(shell date +%Y%m%d_%H%M%S)
ARCHIVE_NAME := $(PROJECT_NAME)_$(TIMESTAMP).tar.zst
ENCRYPTED_ARCHIVE := $(ARCHIVE_NAME).gpg
REMOTE_TARGET := remote-secure-vault:wk-precision-backups

.PHONY: all backup restore verify clean

all: verify backup sync

verify:
	@echo "[*] Verifying system integrity and GPG commit signatures..."
	@git log -n 1 --show-signature | grep -q "Good signature" || (echo "[!] Unsigned or invalid commit signature!" && exit 1)
	@echo "[+] Cryptographic signature verified."

backup:
	@echo "[*] Creating multi-threaded Zstandard compressed snapshot..."
	@mkdir -p $(BACKUP_DIR)
	tar --use-compress-program="zstd -T0 -19" -cf $(BACKUP_DIR)/$(ARCHIVE_NAME) ./data ./Caddyfile ./docker-compose.yml
	
	@echo "[*] Encrypting snapshot with AES256 via GPG..."
	gpg --batch --yes --symmetric --cipher-algo AES256 --output $(BACKUP_DIR)/$(ENCRYPTED_ARCHIVE) $(BACKUP_DIR)/$(ARCHIVE_NAME)
	@rm -f $(BACKUP_DIR)/$(ARCHIVE_NAME)
	@echo "[+] Secure encrypted archive generated: $(BACKUP_DIR)/$(ENCRYPTED_ARCHIVE)"

sync:
	@echo "[*] Synchronizing encrypted archive to cloud storage via rclone..."
	rclone copy $(BACKUP_DIR)/$(ENCRYPTED_ARCHIVE) $(REMOTE_TARGET) --checksum --quiet
	@echo "[+] Cloud backup synchronization complete."

restore:
	@echo "[*] Restoring latest encrypted archive..."
	@LATEST_BACKUP=$$(ls -t $(BACKUP_DIR)/*.gpg | head -n1); \
	if [ -z "$$LATEST_BACKUP" ]; then echo "[!] No backup found!"; exit 1; fi; \
	echo "[*] Decrypting $$LATEST_BACKUP..."; \
	gpg --batch --yes --decrypt --output $(BACKUP_DIR)/restored.tar.zst $$LATEST_BACKUP; \
	tar --use-compress-program="zstd -d" -xf $(BACKUP_DIR)/restored.tar.zst -C ./; \
	rm -f $(BACKUP_DIR)/restored.tar.zst; \
	echo "[+] System state successfully restored and decrypted."

clean:
	@echo "[*] Pruning local temporary build artifacts and dangling Docker volumes..."
	docker system prune -f --volumes
[Unit]
Description=WK Precision Automated Encrypted Backup & Cloud Sync
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=oneshot
WorkingDirectory=/opt/wk-precision-hub
ExecStart=/usr/bin/make -C /opt/wk-precision-hub backup sync
User=root
ProtectSystem=strict
ReadWritePaths=/opt/wk-precision-hub /var/backups/wk-precision-hub
[Unit]
Description=Run WK Precision Encrypted Backup Every 4 Hours

[Timer]
OnBootSec=10min
OnUnitActiveSec=4h
AccuracySec=1m
Persistent=true

[Install]
WantedBy=timers.target
sudo systemctl daemon-reload
sudo systemctl enable --now wk-precision-backup.timer
To achieve an automated, zero-touch, cryptographically secure lifecycle—encompassing local encryption, integrity validation, self-healing orchestration, and off-site cloud synchronization across your history and pipelines—you can implement an integrated Makefile and automated cron-driven backup suite.
The configuration below automates multi-threaded Zstandard compression (zstd), GPG symmetric/asymmetric encryption, local state verification, and encrypted cloud sync (via rclone).
1. Automated Lifecycle Makefile (Makefile)
Place this in your project root (/opt/wk-precision-hub/Makefile) to handle continuous verification, encryption, archiving, and remote push.
SHELL := /bin/bash
PROJECT_NAME := wk-precision-hub
BACKUP_DIR := /var/backups/$(PROJECT_NAME)
TIMESTAMP := $(shell date +%Y%m%d_%H%M%S)
ARCHIVE_NAME := $(PROJECT_NAME)_$(TIMESTAMP).tar.zst
ENCRYPTED_ARCHIVE := $(ARCHIVE_NAME).gpg
REMOTE_TARGET := remote-secure-vault:wk-precision-backups

.PHONY: all backup restore verify clean

all: verify backup sync

verify:
	@echo "[*] Verifying system integrity and GPG commit signatures..."
	@git log -n 1 --show-signature | grep -q "Good signature" || (echo "[!] Unsigned or invalid commit signature!" && exit 1)
	@echo "[+] Cryptographic signature verified."

backup:
	@echo "[*] Creating multi-threaded Zstandard compressed snapshot..."
	@mkdir -p $(BACKUP_DIR)
	tar --use-compress-program="zstd -T0 -19" -cf $(BACKUP_DIR)/$(ARCHIVE_NAME) ./data ./Caddyfile ./docker-compose.yml
	
	@echo "[*] Encrypting snapshot with AES256 via GPG..."
	gpg --batch --yes --symmetric --cipher-algo AES256 --output $(BACKUP_DIR)/$(ENCRYPTED_ARCHIVE) $(BACKUP_DIR)/$(ARCHIVE_NAME)
	@rm -f $(BACKUP_DIR)/$(ARCHIVE_NAME)
	@echo "[+] Secure encrypted archive generated: $(BACKUP_DIR)/$(ENCRYPTED_ARCHIVE)"

sync:
	@echo "[*] Synchronizing encrypted archive to cloud storage via rclone..."
	rclone copy $(BACKUP_DIR)/$(ENCRYPTED_ARCHIVE) $(REMOTE_TARGET) --checksum --quiet
	@echo "[+] Cloud backup synchronization complete."

restore:
	@echo "[*] Restoring latest encrypted archive..."
	@LATEST_BACKUP=$$(ls -t $(BACKUP_DIR)/*.gpg | head -n1); \
	if [ -z "$$LATEST_BACKUP" ]; then echo "[!] No backup found!"; exit 1; fi; \
	echo "[*] Decrypting $$LATEST_BACKUP..."; \
	gpg --batch --yes --decrypt --output $(BACKUP_DIR)/restored.tar.zst $$LATEST_BACKUP; \
	tar --use-compress-program="zstd -d" -xf $(BACKUP_DIR)/restored.tar.zst -C ./; \
	rm -f $(BACKUP_DIR)/restored.tar.zst; \
	echo "[+] System state successfully restored and decrypted."

clean:
	@echo "[*] Pruning local temporary build artifacts and dangling Docker volumes..."
	docker system prune -f --volumes

2. Automated Systemd Timer & Service for Continuous Backup
To ensure everything backs up and self-heals automatically on a continuous schedule without manual intervention, configure a systemd service and timer pair.
/etc/systemd/system/wk-precision-backup.service
[Unit]
Description=WK Precision Automated Encrypted Backup & Cloud Sync
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=oneshot
WorkingDirectory=/opt/wk-precision-hub
ExecStart=/usr/bin/make -C /opt/wk-precision-hub backup sync
User=root
ProtectSystem=strict
ReadWritePaths=/opt/wk-precision-hub /var/backups/wk-precision-hub

/etc/systemd/system/wk-precision-backup.timer
[Unit]
Description=Run WK Precision Encrypted Backup Every 4 Hours

[Timer]
OnBootSec=10min
OnUnitActiveSec=4h
AccuracySec=1m
Persistent=true

[Install]
WantedBy=timers.target

Enable and start the automated background pipeline with:
sudo systemctl daemon-reload
sudo systemctl enable --now wk-precision-backup.timer
# Remote target for WK Precision Enterprise secure vault
[remote-secure-vault]
type = s3
provider = Backblaze B2
access_key_id = YOUR_B2_APPLICATION_KEY_ID_HERE
secret_access_key = YOUR_B2_SECRET_APPLICATION_KEY_HERE
endpoint = s3.us-west-004.backblazeb2.com
acl = private
no_check_bucket = false

# Optional: Add an encrypted wrapper layer using rclone's built-in crypt
[secure-vault-encrypted]
type = crypt
remote = remote-secure-vault:wk-precision-encrypted-backups
filename_encryption = standard
directory_name_encryption = true
password = YOUR_STRONG_ENCRYPTION_SALT_OR_PASSWORD_HASH
mkdir -p /root/.config/rclone
chmod 700 /root/.config/rclone
touch /root/.config/rclone/rclone.conf
chmod 600 /root/.config/rclone/rclone.conf
rclone lsd remote-secure-vault:
To securely connect your automated backup pipeline to encrypted cloud object storage (such as AWS S3, Backblaze B2, Google Cloud Storage, or an encrypted WebDAV / SFTP endpoint), configure rclone with server-side encryption and strict token management.
Below is the production-ready rclone configuration template tailored for your local zero-trust deployment architecture.
1. Rclone Configuration File (rclone.conf)
Place this file in your secure root configuration directory (~/.config/rclone/rclone.conf or /root/.config/rclone/rclone.conf), ensuring file permissions are locked down (chmod 600).
# Remote target for WK Precision Enterprise secure vault
[remote-secure-vault]
type = s3
provider = Backblaze B2
access_key_id = YOUR_B2_APPLICATION_KEY_ID_HERE
secret_access_key = YOUR_B2_SECRET_APPLICATION_KEY_HERE
endpoint = s3.us-west-004.backblazeb2.com
acl = private
no_check_bucket = false

# Optional: Add an encrypted wrapper layer using rclone's built-in crypt
[secure-vault-encrypted]
type = crypt
remote = remote-secure-vault:wk-precision-encrypted-backups
filename_encryption = standard
directory_name_encryption = true
password = YOUR_STRONG_ENCRYPTION_SALT_OR_PASSWORD_HASH

2. Permissions & Verification Setup
Because this configuration file contains sensitive storage credentials, restrict access immediately after creation:
mkdir -p /root/.config/rclone
chmod 700 /root/.config/rclone
touch /root/.config/rclone/rclone.conf
chmod 600 /root/.config/rclone/rclone.conf

You can test the connectivity and remote bucket accessibility directly within your build pipeline using:
rclone lsd remote-secure-vault:

To securely connect your automated backup pipeline to encrypted cloud object storage (such as AWS S3, Backblaze B2, Google Cloud Storage, or an encrypted WebDAV / SFTP endpoint), configure rclone with server-side encryption and strict token management.
Below is the production-ready rclone configuration template tailored for your local zero-trust deployment architecture.
1. Rclone Configuration File (rclone.conf)
Place this file in your secure root configuration directory (~/.config/rclone/rclone.conf or /root/.config/rclone/rclone.conf), ensuring file permissions are locked down (chmod 600).
# Remote target for WK Precision Enterprise secure vault
[remote-secure-vault]
type = s3
provider = Backblaze B2
access_key_id = YOUR_B2_APPLICATION_KEY_ID_HERE
secret_access_key = YOUR_B2_SECRET_APPLICATION_KEY_HERE
endpoint = s3.us-west-004.backblazeb2.com
acl = private
no_check_bucket = false

# Optional: Add an encrypted wrapper layer using rclone's built-in crypt
[secure-vault-encrypted]
type = crypt
remote = remote-secure-vault:wk-precision-encrypted-backups
filename_encryption = standard
directory_name_encryption = true
password = YOUR_STRONG_ENCRYPTION_SALT_OR_PASSWORD_HASH

2. Permissions & Verification Setup
Because this configuration file contains sensitive storage credentials, restrict access immediately after creation:
mkdir -p /root/.config/rclone
chmod 700 /root/.config/rclone
touch /root/.config/rclone/rclone.conf
chmod 600 /root/.config/rclone/rclone.conf

You can test the connectivity and remote bucket accessibility directly within your build pipeline using:
rclone lsd remote-secure-vault:

#!/usr/bin/env bash
set -euo pipefail

PROJECT_DIR="/opt/wk-precision-hub"
TAG="wk-precision-backup"

echo "[*] Initiating automated zero-trust backup and sync sequence..."

# Execute Makefile backup and sync targets, capturing exit status
if make -C "$PROJECT_DIR" backup sync; then
    MESSAGE="[+] SUCCESS: Encrypted backup snapshot created and successfully synchronized to cloud storage."
    echo "$MESSAGE"
    echo "$MESSAGE" | systemd-cat -t "$TAG" -p notice
    exit 0
else
    EXIT_CODE=$?
    MESSAGE="[!] FAILURE: Backup or rclone cloud synchronization failed with exit code $EXIT_CODE!"
    echo "$MESSAGE" >&2
    echo "$MESSAGE" | systemd-cat -t "$TAG" -p err
    exit "$EXIT_CODE"
fi
chmod +x /opt/wk-precision-hub/scripts/backup-wrapper.sh
[Unit]
Description=WK Precision Automated Encrypted Backup & Cloud Sync with Alert Handler
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=oneshot
WorkingDirectory=/opt/wk-precision-hub
ExecStart=/opt/wk-precision-hub/scripts/backup-wrapper.sh
User=root
ProtectSystem=strict
ReadWritePaths=/opt/wk-precision-hub /var/backups/wk-precision-hub /var/log

[Install]
WantedBy=multi-user.target
sudo systemctl daemon-reload
sudo systemctl restart wk-precision-backup.service
sudo systemctl daemon-reload
sudo systemctl restart wk-precision-backup.service
journalctl -t wk-precision-backup -e
To ensure complete auditability and immediate awareness of any backup or synchronization anomalies, you can integrate native systemd status logging with a wrapper script that inspects exit codes, logs successes or failures with specific priority tags (-p notice vs -p err), and writes entries directly to the system journal.
Below is the complete, drop-in logging and alert handler integration for your automated backup pipeline.
1. Backup Wrapper Script with Logging (/opt/wk-precision-hub/scripts/backup-wrapper.sh)
Create this wrapper script to execute your Makefile tasks, capture exact exit statuses, and pipe structured success/failure messages into systemd-cat.
#!/usr/bin/env bash
set -euo pipefail

PROJECT_DIR="/opt/wk-precision-hub"
TAG="wk-precision-backup"

echo "[*] Initiating automated zero-trust backup and sync sequence..."

# Execute Makefile backup and sync targets, capturing exit status
if make -C "$PROJECT_DIR" backup sync; then
    MESSAGE="[+] SUCCESS: Encrypted backup snapshot created and successfully synchronized to cloud storage."
    echo "$MESSAGE"
    echo "$MESSAGE" | systemd-cat -t "$TAG" -p notice
    exit 0
else
    EXIT_CODE=$?
    MESSAGE="[!] FAILURE: Backup or rclone cloud synchronization failed with exit code $EXIT_CODE!"
    echo "$MESSAGE" >&2
    echo "$MESSAGE" | systemd-cat -t "$TAG" -p err
    exit "$EXIT_CODE"
fi

Make the wrapper script executable:
chmod +x /opt/wk-precision-hub/scripts/backup-wrapper.sh

2. Updated Systemd Service File (/etc/systemd/system/wk-precision-backup.service)
Update your systemd service definition to invoke the wrapper script instead of raw Makefile commands, enabling robust logging and automatic failure hooks.
[Unit]
Description=WK Precision Automated Encrypted Backup & Cloud Sync with Alert Handler
After=network-online.target docker.service
Wants=network-online.target

[Service]
Type=oneshot
WorkingDirectory=/opt/wk-precision-hub
ExecStart=/opt/wk-precision-hub/scripts/backup-wrapper.sh
User=root
ProtectSystem=strict
ReadWritePaths=/opt/wk-precision-hub /var/backups/wk-precision-hub /var/log

[Install]
WantedBy=multi-user.target

3. Reload and Verify Service Integration
Apply the changes and reload systemd to recognize the updated handler:
sudo systemctl daemon-reload
sudo systemctl restart wk-precision-backup.service

You can inspect real-time backup execution logs and filter specifically for successes or sync failures using the system journal:
journalctl -t wk-precision-backup -e



<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://assets-eu-01.kc-usercontent.com/ef593040-b591-0198-9506-ed88b30bc023/a23fc7ba-23f0-489a-829d-ed88c0748521/Sonar_Logo_Dark%20Backgrounds.svg">
    <img src="https://assets-eu-01.kc-usercontent.com/ef593040-b591-0198-9506-ed88b30bc023/82c13eba-d95c-4bb8-8007-7ce77c14e043/Sonar_Logo_Light%20Backgrounds.svg" alt="Sonar logo" width="400">
  </picture>
</p>

[![Build](https://github.com/SonarSource/sonarqube/actions/workflows/build.yml/badge.svg)](https://github.com/SonarSource/sonarqube/actions/workflows/build.yml)
[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=sonarqube&metric=alert_status)](https://sonarcloud.io/summary/new_code?id=sonarqube)
[![AI Code Assurance](https://next.sonarqube.com/sonarqube/api/project_badges/ai_code_assurance?project=org.sonarsource.sonarqube%3Asonarqube-private&token=sqb_c0e2fa9ac4ef89f9a8403c6ba235e108ceb1dce1)](https://next.sonarqube.com/sonarqube/dashboard?id=sonarqube)
[![Release](https://img.shields.io/github/v/release/SonarSource/sonarqube)](https://github.com/SonarSource/sonarqube/releases)
[![Docker pulls](https://img.shields.io/docker/pulls/library/sonarqube)](https://hub.docker.com/_/sonarqube)
[![License](https://img.shields.io/badge/license-LGPL%20v3-blue)](#license)
[![Community](https://img.shields.io/badge/community-forum-blue)](https://community.sonarsource.com/c/sq/10)

<!-- sonar-marketing:start -->
<!-- Marketing maintains this section. For wording changes, consult the relevant Product Marketing Manager (PMM). Repository CODEOWNERS review accuracy and merge changes. -->

# SonarQube

SonarQube is the algorithmic verification platform for code quality and security. Its static analysis applies techniques such as symbolic execution and data and control flow analysis to inspect your code, find bugs, vulnerabilities, and structural problems, and tell you exactly what to fix and why, in your IDE, your pull requests, and your CI pipeline.

This repository holds the source of the **SonarQube Community Build**, the free, open-source edition of the SonarQube platform. It shares the same analysis used across the SonarQube product line.

## Built for the AI era

AI writes code faster than teams can review it, creating verification debt: code reaching production before anyone has confirmed what it does. SonarQube applies the same consistent, explainable analysis to every line, whether a developer or an agent wrote it. Vibe, then verify: generate fast, then verify what reaches production. It is the verification stage of the Agent Centric Development Cycle, running in the outer CI verification loop to verify code before it merges.

## What SonarQube finds

- **Bugs and reliability issues** that break behavior at runtime.
- **Security vulnerabilities and security hotspots**, with clear guidance on the risk and the fix.
- **Maintainability and structural issues** that make code harder to change over time.
- **Coverage on new code**, so quality improves with every commit instead of stalling behind a backlog.

The SonarQube platform analyzes more than 40 programming languages and frameworks; the Community Build supports a subset of them. Analysis is repeatable, auditable, and explainable: the same code always produces the same findings, every finding is traceable, and each one tells you what the problem is and why it matters.

## Commercial editions

SonarQube Server and SonarQube Cloud include everything in the Community Build and add:

**Detect more**

- More bugs, vulnerabilities, code quality issues, and architecture issues, through broader coverage and deeper analysis.
- More languages and frameworks than the Community Build.
- Software composition analysis (SCA) for vulnerable and risky open-source dependencies.
- Advanced security with deep taint analysis (SAST) that traces vulnerabilities across data flows.
- Secrets detection for leaked credentials, tokens, and keys.
- Infrastructure as code (IaC) analysis for Terraform, Kubernetes, Docker, and CloudFormation.
- Architecture management to define architectural constraints and catch structural violations.

**Analyze your whole workflow**

- Branch analysis and pull request decoration, so every change is verified before it merges.

**Govern, report, and see across teams**

- Portfolios and applications that roll up quality and security across many projects.
- Executive dashboards and trend reporting.
- Security and compliance reports, including OWASP Top 10, CWE, and PCI DSS.
- Enterprise governance: enforce Quality Gates and permissions across teams, with full audit trails.

**Fix**

- SonarQube Remediation Agent to reduce technical debt by fixing SonarQube issues for you, opening verified fix pull requests you can review and merge.

Some capabilities are part of the SonarQube Advanced Security add-on. See the [product line](#the-sonarqube-product-line) for details.

## Quality gates

A Quality Gate is a pass-or-fail check on your new code. Set the standard once, and SonarQube enforces it automatically in every pull request and pipeline, so issues are caught before they merge rather than found in production.

## The SonarQube product line

The Community Build is free and open source. The wider SonarQube line applies the same analysis across your workflow:

- **[SonarQube Server](https://www.sonarsource.com/products/sonarqube/server/)**, self-managed, with more languages, deeper security analysis, and branch and pull request analysis.
- **[SonarQube Cloud](https://www.sonarsource.com/products/sonarqube/cloud/)**, hosted, with the same capabilities as a managed service.
- **[SonarQube for IDE](https://www.sonarsource.com/products/sonarqube/ide/)**, real-time analysis as you write or generate code.
- **[SonarQube MCP Server](https://github.com/SonarSource/sonarqube-mcp-server)**, bringing SonarQube analysis into your AI agent's context.
- **[SonarQube CLI](https://docs.sonarsource.com/sonarqube-cli)**, running analysis from the command line.

Compare [SonarQube plans and pricing](https://www.sonarsource.com/plans-and-pricing/) to choose the appropriate product and deployment model.

## Product links

- [Website](https://www.sonarsource.com/products/sonarqube)
- [Download](https://www.sonarsource.com/products/sonarqube/downloads)
- [Documentation](https://docs.sonarsource.com/sonarqube)
- [X](https://twitter.com/SonarQube)
- [SonarSource](https://www.sonarsource.com), author of SonarQube

## FAQ

**Is the Community Build free?** Yes. It is free and open source, and this repository is its source.

**What is the difference between the Community Build and SonarQube Server?** The Community Build is the free, self-managed engine. SonarQube Server and SonarQube Cloud are the commercial editions. They cover more languages and add branch and pull request analysis, software composition analysis (SCA), architecture management, advanced security with taint analysis (SAST), secrets detection, infrastructure as code (IaC) analysis, the SonarQube Remediation Agent for verified fix pull requests, and enterprise reporting. Some capabilities are part of the SonarQube Advanced Security add-on. See the [product line](#the-sonarqube-product-line) for details.

**Does it work on AI-generated code?** Yes. SonarQube applies the same analysis to all code, whether a developer or an agent wrote it.

<!-- sonar-marketing:end -->

## Project links

- [Webapp source code](https://github.com/SonarSource/sonarqube-webapp)
- [Issue tracking](https://jira.sonarsource.com/browse/SONAR/), read-only. Only SonarSourcers can create tickets.
- [Responsible Disclosure](https://community.sonarsource.com/t/responsible-vulnerability-disclosure/9317)

## Have Questions or Feedback?

For support questions ("How do I?", "I got this error, why?", ...), please first read the [documentation](https://docs.sonarsource.com/sonarqube) and then head to the [SonarSource Community](https://community.sonarsource.com/c/help/sq/10). The answer to your question has likely already been answered! 🤓

Be aware that this forum is a community, so the standard pleasantries ("Hi", "Thanks", ...) are expected. And if you don't get an answer to your thread, you should sit on your hands for at least three days before bumping it. Operators are not standing by. 😄

## Contributing

If you would like to see a new feature or report a bug, please create a new thread in our [forum](https://community.sonarsource.com/c/sq/10).

Please be aware that we are not actively looking for feature contributions. The truth is that it's extremely difficult for someone outside SonarSource to comply with our roadmap and expectations. Therefore, we typically only accept minor cosmetic changes and typo fixes.

With that in mind, if you would like to submit a code contribution, please create a pull request for this repository. Please explain your motives to contribute this change: what problem you are trying to fix, what improvement you are trying to make.

Make sure that you follow our [code style](https://github.com/SonarSource/sonar-developer-toolset#code-style) and all tests are passing (a GitHub Actions build is executed for each pull request).

Willing to contribute to SonarSource products? We are looking for smart, passionate, and skilled people to help us build world-class code-quality solutions. Have a look at our current [job offers here](https://www.sonarsource.com/company/jobs/)!

## Building

To build sources locally follow these instructions.

### Requirements

- Java 21 - Required to build the project
- Native Git - Must be installed and available in your PATH
- Tests - Can be disabled if needed by adding `-x test` to the gradle command (useful if you just want to build without running tests)

### Build and Run Unit Tests

Execute from the project base directory:

    ./gradlew build

The zip distribution file is generated in `sonar-application/build/distributions/`. Unzip it and start the server by executing:

    # on Linux
    bin/linux-x86-64/sonar.sh start
    # or on MacOS
    bin/macosx-universal-64/sonar.sh start
    # or on Windows
    bin\windows-x86-64\StartSonar.bat

### Open in IDE

If the project has never been built, then build it as usual (see previous section) or use the quicker command:

    ./gradlew ide

Then open the root file `build.gradle` as a project in IntelliJ or Eclipse.

### Gradle Hints

| ./gradlew command                | Description                               |
| -------------------------------- | ----------------------------------------- |
| `dependencies`                   | list dependencies                         |
| `spotlessApply`                  | fix source headers by applying HEADER     |
| `wrapper --gradle-version <version>` | upgrade wrapper                       |

## Building with UI changes

The SonarQube UI (or webapp as we call it), is located in another repository: [sonarqube-webapp](https://github.com/SonarSource/sonarqube-webapp).

When building the `sonarqube` repository, the webapp is automatically downloaded from Maven Central as a dependency, it makes it easy for you to contribute backend changes without having to care about the webapp.

But if your contribution also contains UI changes, you must clone the `sonarqube-webapp` repository, do your changes there, build it locally and then build the `sonarqube` repository using the `WEBAPP_BUILD_PATH` environment variable to target your custom build of the UI.

Here is an example of how to do it:

```bash
cd /path/to/sonarqube-webapp/server/sonar-web
# do your changes

# install dependencies, only needed the first time
yarn

# build the webapp
yarn build


cd /path/to/sonarqube

# build the sonarqube repository using the custom build of the webapp
WEBAPP_BUILD_PATH=/path/to/sonarqube-webapp/server/sonar-web/build/webapp ./gradlew build
```

You can also target a specific version of the webapp by updating the `webappVersion` property in the `./gradle.properties` file and then building the `sonarqube` repository normally.

## Translations files

Historically our translations were stored in `sonar-core/src/main/resources/org/sonar/l10n/core.properties`, but this file is now deprecated and not updated anymore.
Default translations (in English) are now defined in the webapp repository, here:
https://github.com/SonarSource/sonarqube-webapp/blob/master/libs/sq-server-shared/src/l10n/default.ts

The format has changed but you can still have it as a `.properties` file format by running the following command:

```bash
cd /path/to/sonarqube-webapp/server/sonar-web

# install dependencies, only needed the first time
yarn

# generate a backward compatible .properties file with all the translation keys
yarn generate-translation-keys
```

Note that contributing extensions for translations into other languages still work the same way as before. It's just the source of truth for the default translations that changed.

## License

Copyright (C) SonarSource Sàrl.

Licensed under the [GNU Lesser General Public License, Version 3.0](https://www.gnu.org/licenses/lgpl.txt)
