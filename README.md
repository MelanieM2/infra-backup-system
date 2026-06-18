# Local Development Infrastructure & Backup System

This project implements a secure local development and backup infrastructure using SSH, rsync, and Git. It connects a WSL2 development environment to a hardened Linux server, enabling automated backups, Git mirroring, and remote filesystem access over SSH.

In a sense, this is **Project 0** in a multi-stage personal systems engineering stack, including:

* **Project 1:** AI-assisted research digester (arXiv + Gemini pipeline)
* **Project 2:** Linux inspection and system analysis tooling
* **Project 3:** Log Analyzer (not yet published)
* **etc:**  See below in projects list

---

## Project Overview

This system establishes a reliable and secure bridge between a primary development environment and a dedicated backup server. It is designed to ensure:

* Persistent and redundant data storage
* Secure remote access via SSH
* Automated synchronization of development artifacts
* Minimal manual intervention in backup workflows


---

## Architecture Overview
```
                 ┌──────────────┐
                 │   GitHub     │
                 │ Remote Repo  │
                 └──────▲───────┘
                        │ git push
                        │
┌───────────────────────┴───────────────────────┐
│         WSL2 Development System               │
│  - Python / uv environment                    │
│  - Git automation                             │
│  - Bash orchestration                         │
│  - SSH + rsync + SSHFS clients                │
└───────────────┬───────────────────────────────┘
                │
                │ Secure Sync Layer (SSH)
                ▼
      ┌────────────────────────────┐
      │   Linux Backup Server      │
      │ - Bare Git Mirror          │
      │ - rsync file backups       │
      │ - SSHFS remote filesystem  │
      └────────────────────────────┘
```






This creates a **three-layer redundancy model**:

* Local development history (WSL2)
* Remote Git mirror (Backup server)
* Cloud repository (GitHub)

---

## Core Components

### SSH Secure Access Layer

Secure communication between machines is established using **Ed25519 key-based authentication**.

Key features:

* Passwordless authentication between systems
* No dependency on external credential managers
* Secure identity-based login via SSH keys

Setup flow:

```bash
ssh-keygen -t ed25519 -C "dev-infrastructure"
ssh-copy-id user@REMOTE_MIRROR
```

The SSH server runs on the backup machine using OpenSSH with minimal configuration and firewall-controlled exposure.

---

### Firewall & Network Hardening

The backup server is secured using a minimal attack surface approach:

* Only a SSH port is exposed
* All other inbound ports are blocked via UFW
* WPS disabled on router to eliminate known security vulnerabilities
* Static DHCP reservations ensure stable internal addressing

Verification is performed using `nmap` to confirm only expected services are exposed.

---

### Remote Synchronization with rsync

File synchronization between systems is handled using `rsync`, chosen for its efficiency and incremental transfer model.

Key behaviors:

* Only changed files are transferred
* Directory structure preserved automatically
* Network usage minimized through delta transfers

Standard configuration:

```bash
rsync -avz \
  --exclude='.venv/' \
  --exclude='__pycache__/' \
  --exclude='*.pyc' \
  source/ user@REMOTE_MIRROR:/backup/path/
```

Important design rule:
Virtual environments are never synchronized. They are reproducible via `uv` and do not require backup.

---

### Automation Pipeline (automate.sh)

The automation script orchestrates the full workflow:

1. Execute project-level Python scripts
2. Detect changes in tracked output files via Git
3. Commit updates if changes exist
4. Synchronize to REMOTE_MIRROR
5. Push to GitHub remote repository
6. Perform optional rsync-based file backup

#### Conditional Execution Logic

The pipeline is designed to avoid unnecessary operations:

```bash
GIT_CHANGED=false

if git status --porcelain | grep -q "research_log.md"; then
    git add research_log.md
    git commit -m "Auto-update: $(date +'%Y-%m-%d %H:%M:%S')"
    GIT_CHANGED=true
fi

if [ "$GIT_CHANGED" = true ]; then
    git push origin main
    git push REMOTE_MIRROR main
    rsync -avz source/ REMOTE_MIRROR:/backup/path/
fi
```

#### Design Principles

* No redundant network operations
* Safe failure behavior (local state always preserved)
* Environment-variable-based configuration for portability
* Strict separation between source code and generated artifacts

---

### Headless Server Configuration

The backup server is configured as a headless Linux system:

* Runs in `multi-user.target` (no GUI)
* Optimized for remote access and background services
* Reduced memory and CPU overhead compared to desktop mode

Sleep and suspend states are disabled to ensure continuous availability for SSH and synchronization tasks.

---

### Remote Filesystem Access (SSHFS)

For development convenience, the remote server can be mounted as a local filesystem using SSHFS.

This enables direct file editing on the REMOTE_MIRROR from the primary machine.

```bash
sshfs user@REMOTE_MIRROR:/remote/path /mnt/server
```

A safe mount/unmount utility is provided via shell functions:

```bash
mountserver() {
    local remote_path="${1:-/home/user}"

    if mountpoint -q /mnt/server; then
        echo "[INFO] Already mounted."
        return 1
    fi

    sshfs user@REMOTE_MIRROR:"$remote_path" /mnt/server
}

alias umountserver='fusermount3 -u /mnt/server'
```

---

### System Administration & Tools

| Tool          | Purpose                             |
| ------------- | ----------------------------------- |
| SSH / OpenSSH | Secure remote access                |
| rsync         | Incremental file synchronization    |
| Git           | Version control and backup tracking |
| UFW           | Firewall management                 |
| nmap          | Network auditing                    |
| systemctl     | Service and boot management         |
| sshfs         | Remote filesystem mounting          |
| sed           | System configuration editing        |
| bash          | Automation scripting                |
| uv            | Python environment management       |

---

## Security Principles

This infrastructure is built around the following security practices:

* Key-based SSH authentication (no passwords)
* Minimal open network ports (SSH only)
* No hardcoded secrets in scripts or repositories
* Environment variables for all sensitive configuration
* Separation of local development and backup infrastructure
* Disabled unnecessary router features (e.g., WPS)
* Strict firewall enforcement on server systems


---

## Usage

### Initial setup

```bash
git clone git@github.com:your_username/infra-backup-system.git
cd infra-backup-system
```

### Run synchronization pipeline

```bash
./automate.sh
```

### Manual rsync test

```bash
rsync -avz source/ user@REMOTE_MIRROR:/backup/path/
```

### Mount remote filesystem

```bash
sshfs user@REMOTE_MIRROR:/remote/path /mnt/server
```

---

## Design Philosophy

This project treats infrastructure as code: reproducible, version-controlled, and automated where possible.

The goal is not only data safety, but also the creation of a stable foundation for higher-level automation systems such as AI-assisted research pipelines and system inspection tools.

By separating concerns into distinct layers (local development, remote mirror, and cloud backup), the system ensures resilience, portability, and long-term maintainability.


## Development Notes & AI Usage

### AI-Assisted Engineering Workflow

This project was developed using large language models (LLMs), including Google Gemini and Anthropic Claude, as interactive pair-programming tools to support exploration of system design, code structuring, and implementation details.

LLMs were used for:

* Clarifying unfamiliar concepts
* Exploring architectural design options
* Reviewing and iterating on code structure
* Accelerating development of boilerplate and automation logic

All generated code was carefully reviewed, tested, and validated by the author to ensure correctness and full understanding of the implemented system.

The final codebase reflects a combination of AI-assisted development and manual engineering decisions, with emphasis on maintainability, security, system-level clarity, and personal learning.

---

## Project Context

The infrastructure in this project supports a broader set of ongoing experiments across systems engineering, data science, machine learning, and AI-assisted development workflows.

The topics listed below offer a flexible, evolving, experimental playground for learning, rather than a fixed or exhaustive roadmap. These topics, or aspects thereof, are being developed as concrete projects by the author, in an iterative and parallel manner, and are published to GitHub incrementally once they reach a stable and reasonably complete state.

### Systems Engineering & Infrastructure
- Linux automation tools
- Log analysis utilities
- File synchronization systems
- Backup and remote infrastructure (Project 0 foundation)
- System inspection tooling (see 'linux-system-inspector' Project)

### Data Science & Applied Analysis
- Regional housing price dynamics and affordability modeling under macroeconomic conditions (Austria / Europe)
- Macroeconomic signal decomposition (inflation, wages, interest rates) and structural trend analysis
- Financial time series modeling for non-stationary and regime-dependent data
- Data pipelines for reproducible economic analysis and structured indicator extraction

---

### Machine Learning Systems
- Supervised learning systems for structured prediction tasks (regression and classification)
- Model evaluation under dataset shift and real-world noise conditions
- Feature engineering pipelines for tabular and temporal datasets
- Baseline-to-iterative model refinement workflows for applied prediction tasks

---

### Deep Learning Exploration
- Implementation and training of neural networks from scratch for conceptual grounding
- Transformer architectures using TensorFlow and PyTorch with focus on attention mechanisms
- Experimental exploration of representation learning in sequence-based models
- Analysis of training stability, optimization behavior, and scaling effects in deep networks

---

### AI-Assisted Data Engineering
- End-to-end research paper summarization pipeline (see 'research_digester' Project)
- LLM-assisted extraction, transformation, and structuring of unstructured scientific text
- AI-supported data exploration and automated insight generation workflows
- Integration of large language models into reproducible data processing pipelines

---

### Geometric, Graph-Based, and Algebraic Foundations of Deep Learning
- Geometric deep learning experiments on graph-structured data
- Differential geometric methods for representation learning and optimization
- Spectral geometry approaches to learning on manifolds and graphs
- Algebraic and category-theoretic perspectives on deep learning architectures and compositional structure
- AI-assisted mathematical reasoning tools for theoretical exploration and model abstraction
---

## Long-Term Direction

The long-term goal is to develop expertise at the intersection of mathematics, data science, artificial intelligence, and systems engineering, bridging theoretical foundations in machine learning and deep learning with practical experience in software development, automation, and production-grade systems.

## Feedback

This project is part of an ongoing learning and engineering journey. Constructive feedback, corrections, and suggestions for improvement are greatly appreciated. Please feel free to open an issue or contact the author through GitHub.