# Cybersecurity Home Lab

A growing collection of hands-on cybersecurity projects focused on defensive security, networking, system administration, traffic analysis, and infrastructure hardening.

This repository documents the environments I build, the tests I perform, the evidence I collect, and the conclusions I draw. My objective is to understand how systems behave before and after security controls are introduced, then verify those changes using scans, packet captures, logs, and local system state.

> Note: All testing in this repository is performed in isolated environments.

## Repository Goals

This home lab is intended to help me develop practical experience in areas including:

- Linux and Windows administration
- Network architecture and segmentation
- Service enumeration and exposure analysis
- Packet capture and protocol analysis
- Host-based firewall configuration
- Security logging and event correlation
- Endpoint telemetry and detection engineering
- Incident investigation and technical reporting
- Raspberry Pi and home-infrastructure security
- Security automation using scripting languages

Each project will follow a repeatable defensive workflow:

1. **Build** a controlled environment
2. **Establish a baseline** before making security changes
3. **Generate and observe activity** using appropriate tools
4. **Apply a security control** or configuration change
5. **Validate the result** from both local and remote perspectives
6. **Collect evidence** such as scans, packet captures, logs, and configuration output
7. **Document findings**, limitations, and possible extensions

## Projects

| Project | Primary Topics | Status |
|---|---|---|
| [01 — Network Baseline and Linux Firewall Hardening](./projects/01-network-baseline/) | VMware, Linux, Nmap, Wireshark, TCP/IP, UFW, firewalls | Complete |
| 02 — Defensive PCAP Investigation | Network forensics, timelines, indicators, incident reporting | Ongoing |
| 03 — Raspberry Pi DNS Security and Monitoring | Pi-hole, DNS, Linux, network visibility | Planned |
| 04 — Windows Endpoint Telemetry | Windows Event Logs, Sysmon, PowerShell, endpoint analysis | Planned |
| 05 — Mini-SOC Environment | Wazuh, centralized logging, alert triage, detection validation | Planned |

As always, this roadmap may change as my interests develop and I begin to find my niche.

## Repository Structure

```text
cybersecurity-home-lab/
├── README.md
└── projects/
    └── 01-network-baseline/
        ├── README.md
        ├── technical-notes.md
        ├── SHA256SUMS
        ├── artifacts/
        │   ├── logs/
        │   ├── pcaps/
        │   ├── scans/
        │   └── system/
        └── screenshots/
```

Most projects will have same general pattern:

- `README.md`: polished case study and project findings
- `technical-notes.md`: detailed concepts, commands, and troubleshooting notes
- `artifacts/`: raw evidence generated during the project
- `screenshots/`: selected visual evidence used in the writeup
- `diagrams/`: architecture and workflow diagrams
- `SHA256SUMS`: hashes used to verify the integrity of collected artifacts

## Evidence and Reproducibility

Where appropriate, projects include:

- Nmap output
- Listening-socket inventories
- Packet captures
- Firewall or OS logs
- Final validation reports
- Configuration summaries
- SHA-256 hashes for artifact-integrity verification

The raw evidence is included so that conclusions in the project writeups can be checked as opposed to blindly accepted. Sensitive information, credentials, private keys, and unrelated personal data are excluded from the repository.

## Project Documentation

Future project READMEs will generally include:

1. Executive summary
2. Objectives
3. Scope and safety controls
4. Architecture
5. Environment and tools
6. Methodology
7. Results and findings
8. Evidence
9. Final configuration or outcome
10. Lessons learned
11. Limitations
12. Future extensions

These READMEs will generally be based on a shared template, although the structure may vary by project.

## Responsible Use

The tools and techniques documented here are used only against systems and networks that I own or am explicitly authorized to test. The repository is intended for education, defensive experimentation, and professional development.
