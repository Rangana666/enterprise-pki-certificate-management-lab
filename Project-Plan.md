# Enterprise PKI & Certificate Lifecycle Management Lab

A self-initiated, hands-on Public Key Infrastructure (PKI) laboratory built on Ubuntu Linux with OpenSSL, Python, Bash and Apache. It simulates how an organization runs a two-tier Certificate Authority and manages the full life of a digital certificate: request, approval, issuance, validation, monitoring, renewal, revocation, incident response and audit.

> **Project type:** Personal cybersecurity and infrastructure security project
> **Status:** In development (see [Project Status](#15-project-status))
> **Environment:** Ubuntu Linux 24.04 LTS (optional Windows Server VM for the AD CS module)
> **Core technologies:** OpenSSL, Python 3, Bash, Apache2, SoftHSM, Git

---

## Table of Contents

1. [Why This Project](#1-why-this-project)
2. [Objectives](#2-objectives)
3. [Skills Map: Job Responsibilities vs This Lab](#3-skills-map-job-responsibilities-vs-this-lab)
4. [Technologies](#4-technologies)
5. [PKI Architecture](#5-pki-architecture)
6. [Modules in Detail](#6-modules-in-detail)
7. [Repository Structure](#7-repository-structure)
8. [Development Roadmap](#8-development-roadmap)
9. [Key Command Reference](#9-key-command-reference)
10. [Testing and Validation](#10-testing-and-validation)
11. [Troubleshooting Journal](#11-troubleshooting-journal)
12. [Documentation and Evidence](#12-documentation-and-evidence)
13. [Security Considerations](#13-security-considerations)
14. [Learning Outcomes](#14-learning-outcomes)
15. [Project Status](#15-project-status)
16. [Author](#16-author)

---

## 1. Why This Project

Organizations rely on PKI for HTTPS, VPN, email security, device identity and user authentication. When a certificate expires unnoticed, or a CA key is mishandled, systems fail or trust is lost. PKI administration is therefore less about running a single command and more about **process**: policies, approvals, monitoring, documentation and incident handling.

This lab was started to learn that whole picture by building it, rather than only reading about it. It covers the technical side (CAs, keys, certificates, TLS) and the operational side (policy, RA workflow, monitoring, audit records, incident runbooks).

## 2. Objectives

- Design and build a two-tier PKI (Root CA and Intermediate CA).
- Generate and protect RSA key pairs, and explore hardware-style key protection with SoftHSM (PKCS#11).
- Create CSRs and issue X.509 certificates with correct extensions (SAN, key usage, EKU).
- Simulate a Registration Authority (RA) approval workflow with a request/approval record.
- Verify signatures and certificate chains, and diagnose chain, trust, hostname and expiry errors.
- Monitor certificate validity and generate expiry alerts and reports.
- Implement renewal, revocation, CRL and OCSP.
- Deploy CA-issued certificates on Apache HTTPS and test TLS.
- Write a Certificate Policy / Practice Statement (CP/CPS-lite) and map controls to recognized standards.
- Keep audit logs and write incident response runbooks.
- Compare the OpenSSL lab with Microsoft Active Directory Certificate Services (AD CS).
- Automate repeatable tasks with Bash and Python and document everything.

## 3. Skills Map: Job Responsibilities vs This Lab

| Typical PKI role responsibility / requirement | Where it is practiced in this lab |
|---|---|
| Manage and maintain PKI, digital certificates and cryptographic keys | Modules A, B, C, F (CA setup, issuance, key management) |
| Monitor certificate validity, renewals, revocations, expiry dates | Modules D, E, H (inventory, expiry monitor, CRL/OCSP) |
| Support CA and RA implementation and maintenance | Modules A, B, C (two-tier CA, simulated RA) |
| Compliance with security policies, cryptographic standards, best practices | Module G (CP/CPS-lite, NIST and CA/B Forum mapping) |
| Monitor PKI for incidents, vulnerabilities, operational issues | Modules H, I (audit logging, monitoring report, runbooks) |
| Troubleshoot certificate, authentication and encryption problems | Module J (TLS troubleshooting lab) |
| Maintain PKI documentation, audit records, procedures | `docs/`, `policy/`, `runbooks/`, audit log |
| Collaborate with IT, network and security teams | Module C ticket workflow, `docs/team-collaboration.md` |
| Windows/Linux, Active Directory, network security (advantage) | Module K (AD CS comparison / optional Windows build) |
| Key management systems, security protocols (advantage) | Module F (SoftHSM, key lifecycle), TLS testing |

## 4. Technologies

| Technology | Purpose |
|---|---|
| Ubuntu Linux 24.04 LTS | Lab environment |
| OpenSSL 3.x | CA operations, CSR, certificates, CRL, OCSP, TLS testing |
| RSA 4096 / 2048, SHA-256 | Key pairs and signatures |
| X.509 v3 | Certificate standard and extensions |
| SoftHSM2 + pkcs11-tool | Software HSM for key protection practice |
| Python 3 | Inventory, monitoring, reporting, audit logging |
| Bash | Operational scripts |
| Apache2 | HTTPS test server |
| cURL, `openssl s_client` | TLS and trust testing |
| Windows Server (optional) | AD CS comparison module |
| Git / GitHub | Version control and publication |

## 5. PKI Architecture

```text
                    ROOT CA  (offline in practice)
                       |  signs
                       v
               INTERMEDIATE (ISSUING) CA
                       |  issues
          +------------+-------------+
          |                          |
   Web server certs            Client / test certs
          |                          |
   Apache HTTPS                Client authentication

   Supporting services
   - RA workflow (request -> review -> approve/reject -> issue)
   - CRL publication and OCSP responder
   - Certificate inventory and expiry monitor
   - Audit log and monitoring reports
```

**Design decisions to be able to explain**

- The Root CA signs only the Intermediate CA. After that it is kept offline, so day-to-day issuance never exposes the root key.
- The Intermediate CA issues end-entity certificates. If it is compromised, it can be revoked without replacing the root.
- Revocation is offered two ways: CRL (a signed list) and OCSP (live status query).
- The RA is a **simulation**: it records who requested what, who approved it and why. It is not a real identity verification service.

## 6. Modules in Detail

### Module A: Root CA
- Directory layout (`private/`, `certs/`, `newcerts/`, `crl/`, `index.txt`, `serial`).
- Encrypted 4096-bit root key, self-signed root certificate with `CA:TRUE` and `keyCertSign, cRLSign`.
- Long validity (e.g. 20 years) compared with the intermediate.
- **Deliverable:** root certificate, `docs/03-root-ca-implementation.md`.

### Module B: Intermediate CA
- Intermediate key and CSR, signed by the root with `pathlen:0`.
- Chain file (`ca-chain.crt`) and chain verification.
- Intermediate CA configuration with a server and client certificate policy.
- **Deliverable:** working signing CA, `docs/04-intermediate-ca-implementation.md`.

### Module C: Certificate Issuance with a Simulated RA Workflow
- Request form (markdown or CSV): requester, common name, SANs, purpose, requested validity, ticket ID.
- `ra_workflow.py` records the decision (approved or rejected), approver, reason and timestamp, then hands approved requests to the Intermediate CA.
- Issued certificates get SAN, key usage and extended key usage (`serverAuth` or `clientAuth`).
- Rejection cases are tested too (wrong domain, weak key size, missing SAN).
- Short `docs/team-collaboration.md` describing how requests, approvals and incidents would be coordinated with network and security teams.
- **Deliverable:** issuance scripts, request log, `docs/05-certificate-issuance.md`.

### Module D: Certificate Inventory and Lifecycle
- `certificate_inventory.py` scans the CA database and certificate directories and builds an inventory: subject, issuer, serial, SANs, not-before, not-after, status (valid, expiring, expired, revoked).
- Renewal workflow: new CSR (optionally same key policy), reissue, replace on server, record the event.
- **Deliverable:** inventory CSV/JSON, renewal script, `docs/07-certificate-lifecycle-management.md`.

### Module E: Expiry Monitoring and Reporting
- `check-expiry.sh` uses `openssl x509 -checkend`.
- `certificate_monitor.py` classifies certificates (30 / 14 / 7 days) and writes a weekly HTML or CSV report: issued, expiring, expired, revoked.
- Optional alert by email or log file entry.
- **Deliverable:** report samples and `docs/10-security-monitoring-and-auditing.md`.

### Module F: Key Management with SoftHSM
- Initialize a SoftHSM token and generate the Intermediate CA key inside it (PKCS#11), using `pkcs11-tool` and the OpenSSL PKCS#11 provider.
- Key lifecycle document: generation, storage, access, backup, rotation, retirement, destruction.
- Compare file-based keys with HSM-protected keys and note the limits of a software HSM.
- This is a stretch module. If PKCS#11 integration with the CA proves difficult, document what was attempted and what failed.
- **Deliverable:** `docs/key-management.md`.

### Module G: Policy and Compliance (CP/CPS-lite)
- A 3 to 4 page policy covering: certificate types, key sizes and algorithms, validity periods, who may request, approval rules, revocation reasons, key protection, audit and record retention.
- A control mapping table linking each rule to a recognized reference, such as NIST SP 800-57 (key management), NIST SP 800-52 (TLS), RFC 5280 (X.509/CRL) and CA/Browser Forum Baseline Requirements where relevant.
- A simple self-audit checklist run against the lab, with findings recorded.
- **Deliverable:** `policy/CP-CPS-lite.md`, `policy/control-mapping.md`, `policy/self-audit-checklist.md`.

### Module H: Revocation, CRL and OCSP
- Revoke with reasons (`keyCompromise`, `superseded`, `cessationOfOperation`).
- Generate and publish a CRL, verify revoked status with `openssl verify -crl_check`.
- Run a local OCSP responder with its own signing certificate and query it.
- Document the difference between CRL and OCSP and when each fails (stale CRL, responder down).
- **Deliverable:** `docs/08-certificate-revocation.md`.

### Module I: Security Monitoring, Audit Logging and Incident Runbooks
- `audit_logger.py` writes structured entries (timestamp, actor, action, serial, result) for issue, renew, revoke and failed requests.
- Runbooks in `runbooks/`:
  1. Expired certificate causing a service outage
  2. Private key compromise of an end-entity certificate
  3. Suspected Intermediate CA key compromise (revoke, reissue, redistribute chain)
- Run at least one runbook as a simulation and record the timeline and lessons learned.
- **Deliverable:** audit log samples, runbooks, one incident report.

### Module J: Apache HTTPS and TLS Troubleshooting Lab
- Deploy a CA-issued certificate on Apache with the full chain, and test with browser, cURL and `openssl s_client`.
- Deliberately break things and document the diagnosis and fix:
  - missing intermediate in the served chain
  - hostname / SAN mismatch
  - expired certificate
  - untrusted root
  - revoked certificate
  - key and certificate mismatch
- Include a table of error message, cause, how it was found and the fix.
- **Deliverable:** `docs/09-ssl-tls-implementation.md`, `docs/11-testing-and-troubleshooting.md`.

### Module K: Windows and Active Directory Comparison
- **Option 1 (preferred if resources allow):** Windows Server evaluation VM with AD CS, a domain-joined client, root certificate distribution through Group Policy, and a certificate template with auto-enrollment.
- **Option 2:** If a Windows lab is not possible, write a clearly labelled comparison between this OpenSSL lab and AD CS (templates, auto-enrollment, GPO trust distribution, role separation) based on study. Do not present it as hands-on work.
- Either way, state in the README which option was actually done.
- **Deliverable:** `docs/windows-adcs-comparison.md`.

## 7. Repository Structure

```text
enterprise-pki-certificate-management-lab/
│
├── README.md
├── LICENSE
├── .gitignore                  # excludes keys, passphrases, CA databases
│
├── docs/
│   ├── 01-environment-setup.md
│   ├── 02-pki-architecture.md
│   ├── 03-root-ca-implementation.md
│   ├── 04-intermediate-ca-implementation.md
│   ├── 05-certificate-issuance.md
│   ├── 06-certificate-validation.md
│   ├── 07-certificate-lifecycle-management.md
│   ├── 08-certificate-revocation.md
│   ├── 09-ssl-tls-implementation.md
│   ├── 10-security-monitoring-and-auditing.md
│   ├── 11-testing-and-troubleshooting.md
│   ├── key-management.md
│   ├── team-collaboration.md
│   └── windows-adcs-comparison.md
│
├── policy/
│   ├── CP-CPS-lite.md
│   ├── control-mapping.md
│   └── self-audit-checklist.md
│
├── runbooks/
│   ├── expired-certificate-outage.md
│   ├── end-entity-key-compromise.md
│   └── intermediate-ca-compromise.md
│
├── configs/
│   ├── root-ca.cnf
│   ├── intermediate-ca.cnf
│   ├── ocsp.cnf
│   └── apache-ssl.conf
│
├── scripts/
│   ├── setup.sh
│   ├── generate-csr.sh
│   ├── issue-certificate.sh
│   ├── verify-certificate.sh
│   ├── renew-certificate.sh
│   ├── revoke-certificate.sh
│   ├── generate-crl.sh
│   ├── start-ocsp.sh
│   └── check-expiry.sh
│
├── src/
│   ├── ra_workflow.py
│   ├── certificate_inventory.py
│   ├── certificate_monitor.py
│   ├── report_generator.py
│   └── audit_logger.py
│
├── requests/                   # sample RA request forms (no secrets)
├── reports/                    # sample generated reports
│
├── tests/
│   ├── test_certificate_validation.sh
│   ├── test_certificate_expiry.sh
│   ├── test_certificate_revocation.sh
│   └── test_ocsp.sh
│
├── web/
│   └── index.html
│
└── screenshots/
    ├── 01-environment/
    ├── 03-root-ca/
    ├── 04-intermediate-ca/
    ├── 05-certificate-issuance/
    ├── 06-validation/
    ├── 07-lifecycle/
    ├── 08-revocation/
    ├── 09-https/
    └── 10-incident-simulation/
```

> Private keys, passphrases and the live CA database must never be committed. Only configuration, scripts, documentation and sample (non-sensitive) outputs are published.

## 8. Development Roadmap

| Stage | Focus | Modules |
|---|---|---|
| 1 | Environment preparation and tool verification | Setup |
| 2 | PKI design, OpenSSL config files, directory layout | Architecture |
| 3 | Root CA and Intermediate CA, chain verification | A, B |
| 4 | RA workflow and certificate issuance | C |
| 5 | Inventory, expiry monitoring, renewal, reports | D, E |
| 6 | Validation and TLS troubleshooting scenarios | J (part) |
| 7 | Revocation, CRL, OCSP, audit logging | H, I (part) |
| 8 | Apache HTTPS deployment | J |
| 9 | Policy / CP-CPS-lite and control mapping | G |
| 10 | Key management with SoftHSM | F |
| 11 | Incident runbooks and one simulation | I |
| 12 | Windows / AD CS module | K |
| 13 | Automation, tests, screenshots, final documentation, publication | All |

Build the core (stages 1 to 8) first. Policy, HSM, runbooks and AD CS come after the core works, and each is documented honestly as done, partial or not attempted.

## 9. Key Command Reference

Paths assume the CA directories from `setup.sh`. Adjust names to match your configs.

**Root CA**

```bash
openssl genrsa -aes256 -out private/root-ca.key 4096
openssl req -config configs/root-ca.cnf -key private/root-ca.key \
  -new -x509 -days 7300 -sha256 -extensions v3_ca -out certs/root-ca.crt
openssl x509 -in certs/root-ca.crt -noout -text
```

**Intermediate CA**

```bash
openssl genrsa -aes256 -out private/intermediate-ca.key 4096
openssl req -config configs/intermediate-ca.cnf -new -sha256 \
  -key private/intermediate-ca.key -out csr/intermediate-ca.csr
openssl ca -config configs/root-ca.cnf -extensions v3_intermediate_ca \
  -days 3650 -notext -md sha256 -in csr/intermediate-ca.csr -out certs/intermediate-ca.crt
openssl verify -CAfile certs/root-ca.crt certs/intermediate-ca.crt
cat certs/intermediate-ca.crt certs/root-ca.crt > certs/ca-chain.crt
```

**Server certificate with SAN**

```bash
openssl genrsa -out private/server.key 2048
openssl req -new -key private/server.key -out csr/server.csr \
  -subj "/C=LK/O=PKI Lab/CN=pki-lab.local" \
  -addext "subjectAltName=DNS:pki-lab.local,DNS:www.pki-lab.local"
openssl ca -config configs/intermediate-ca.cnf -extensions server_cert \
  -days 365 -notext -md sha256 -in csr/server.csr -out certs/server.crt
openssl verify -CAfile certs/ca-chain.crt certs/server.crt
```

**Expiry check (30 days)**

```bash
openssl x509 -checkend 2592000 -noout -in certs/server.crt
openssl x509 -enddate -noout -in certs/server.crt
```

**Revocation and CRL**

```bash
openssl ca -config configs/intermediate-ca.cnf -revoke certs/server.crt -crl_reason keyCompromise
openssl ca -config configs/intermediate-ca.cnf -gencrl -out crl/intermediate.crl.pem
openssl crl -in crl/intermediate.crl.pem -noout -text
openssl verify -crl_check -CAfile certs/ca-chain.crt -CRLfile crl/intermediate.crl.pem certs/server.crt
```

**OCSP**

```bash
# Responder
openssl ocsp -port 2560 -text -sha256 -index index.txt \
  -CA certs/ca-chain.crt -rkey private/ocsp.key -rsigner certs/ocsp.crt
# Client query
openssl ocsp -CAfile certs/ca-chain.crt -issuer certs/intermediate-ca.crt \
  -cert certs/server.crt -url http://127.0.0.1:2560 -resp_text
```

**TLS testing**

```bash
openssl s_client -connect pki-lab.local:443 -servername pki-lab.local -CAfile certs/ca-chain.crt -showcerts
curl --cacert certs/ca-chain.crt https://pki-lab.local/
```

**SoftHSM (key management module)**

```bash
softhsm2-util --init-token --free --label "IntermediateCA" --so-pin <SO_PIN> --pin <USER_PIN>
pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so --login --keypairgen \
  --key-type rsa:4096 --label "intermediate-ca-key"
```

Never paste real PINs or passphrases into committed files.

## 10. Testing and Validation

| # | Test scenario | Expected result |
|---|---|---|
| 1 | Valid CA-signed certificate | Verification succeeds |
| 2 | Tampered certificate | Signature verification fails |
| 3 | Expired certificate | Expiry detected, monitor alert |
| 4 | Certificate expiring within 30 days | Warning in report |
| 5 | Untrusted CA | Trust validation fails |
| 6 | Wrong hostname / missing SAN | Hostname validation fails |
| 7 | Missing intermediate in served chain | Chain verification error |
| 8 | Revoked certificate (CRL) | `verify -crl_check` fails |
| 9 | Revoked certificate (OCSP) | Responder returns `revoked` |
| 10 | Stale CRL | Failure or warning documented |
| 11 | RA request rejected (weak key / bad domain) | No certificate issued, audit entry written |
| 12 | Valid HTTPS connection | TLS connection succeeds |
| 13 | Renewal | New certificate issued, old one superseded, event logged |

Real results, including failures, are recorded in `docs/11-testing-and-troubleshooting.md`.

## 11. Troubleshooting Journal

A running table of real problems met during the build:

| Date | Symptom / error | Root cause | How it was diagnosed | Fix |
|---|---|---|---|---|
| (to be filled as problems occur) | | | | |

Only real issues from the build go here.

## 12. Documentation and Evidence

Every module includes:
- purpose and design reasoning
- the exact commands and configs used
- terminal output
- verification evidence (genuine screenshots taken during the build)
- problems met and how they were fixed

## 13. Security Considerations

This is an educational lab, **not** a production CA.

- Root CA key encrypted and kept offline after signing the intermediate (simulated by storing it separately).
- Private keys and passphrases excluded from Git via `.gitignore`.
- Restrictive file permissions on key directories (`chmod 700` / `600`).
- Separate directories and configs for Root and Intermediate CA.
- Appropriate extensions: `basicConstraints`, `keyUsage`, `extendedKeyUsage`, SAN.
- All issuance, renewal and revocation actions logged.
- A production PKI would additionally need a hardware HSM, offline root ceremonies with multiple custodians, formal identity vetting, backup and recovery of keys, redundancy, and independent audit.

## 14. Learning Outcomes

- PKI architecture and CA hierarchy design
- X.509 certificate structure and extensions
- CSR, issuance, renewal and revocation workflows
- CRL and OCSP, and the trade-offs between them
- Key management and key lifecycle concepts
- Policy writing and mapping controls to standards
- TLS configuration and certificate troubleshooting
- Monitoring, reporting and audit logging
- Incident response for certificate and key compromise
- Linux administration and scripting for security operations
- Comparing open-source PKI with Microsoft AD CS

## 15. Project Status

**Current status:** In development. Items are marked *Completed* only after they have been built and tested.

| Module | Status |
|---|---|
| Environment preparation | Planned |
| A. Root CA | Planned |
| B. Intermediate CA | Planned |
| C. RA workflow and issuance | Planned |
| D. Inventory and lifecycle | Planned |
| E. Expiry monitoring and reporting | Planned |
| F. Key management (SoftHSM) | Planned |
| G. Policy and compliance (CP/CPS-lite) | Planned |
| H. Revocation, CRL and OCSP | Planned |
| I. Audit logging and incident runbooks | Planned |
| J. Apache HTTPS and troubleshooting | Planned |
| K. Windows / AD CS | Planned |
| Automation, tests and publication | Planned |

## 16. Author

**Ravindu Rangana**
Computer Science | Linux Administration | Cybersecurity | Network and Infrastructure Security

- GitHub: https://github.com/Rangana666
- LinkedIn: https://linkedin.com/in/ravindurangana

---

**Disclaimer:** This repository is a self-directed educational project. It is not intended to provide production-grade PKI services, and the RA workflow is a simulation. Any module marked as planned or partial has not been completed.
