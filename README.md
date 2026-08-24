# `mypostgress` — Universal PostgreSQL Manager

[![Bash](https://img.shields.io/badge/Language-Bash%20%3E%3D%204.0-4EAA25?logo=gnu-bash&logoColor=white)](https://www.gnu.org/software/bash/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-12%20--%2018%2B-336791?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Linux](https://img.shields.io/badge/Platform-Arch%20%7C%20Debian%20%7C%20Ubuntu%20%7C%20Fedora%20%7C%20RHEL%20%7C%20Alpine%20%7C%20macOS-FCC624?logo=linux&logoColor=black)](https://kernel.org)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

A powerful, universal CLI and interactive terminal tool for effortless PostgreSQL administration across **any Linux distribution and macOS**.

Whether you need to initialize a fresh database cluster, start/stop services, generate copy-paste ready connection strings for your application stack, connect via pgAdmin 4, or run automated backups and restores — `mypostgress` handles it all with zero fuss.

---

## 🌟 Key Features

* 🌍 **Universal Multi-Distro Support**: Works out-of-the-box on **Arch Linux / CachyOS / Manjaro**, **Ubuntu / Debian / Mint**, **Fedora / RHEL / CentOS / Rocky**, **openSUSE**, **Alpine Linux**, and **macOS**.
* 📦 **Automatic Dependency & Distro Installer**: Detects missing PostgreSQL packages and offers 1-click installation via `pacman`, `apt`, `dnf`, `zypper`, `apk`, or `brew`.
* 🛠️ **Smart Cluster Initialization (`initdb`)**: Automatically detects missing/uninitialized `/var/lib/postgres/data` clusters and initializes them with proper UTF-8 locales.
* 👤 **1-Click User & Superuser Setup**: Easily creates a matching PostgreSQL superuser role for your Linux user so you can run `psql` and development tools without sudo prompts or peer-auth errors.
* 🔗 **Application Connection String Generator**: Instantly outputs formatted connection strings for **Standard URIs**, **psql CLI**, **`.env` files**, **Node.js (Prisma / TypeORM / Drizzle / Sequelize)**, **Python (SQLAlchemy / psycopg2 / asyncpg / Django)**, **Go (pgx / lib/pq)**, **Java/Kotlin (JDBC)**, and **PHP (PDO)**.
* 🐘 **pgAdmin 4 Integration**: Automatically finds and launches pgAdmin 4 (Flatpak, Native, Snap, Docker, or Web) and prints server connection parameters. If not installed, it offers an automatic installer.
* 💾 **Smart Backup & Restore**: Supports both plain SQL (`.sql`) and custom compressed archives (`.dump`, `.tar`, `.backup`). Auto-creates target databases during restore if they don't exist.
* ⚡ **Dual Interface**: Full-featured **interactive menu** when run without arguments, plus fast **direct CLI commands** for scripting and terminal efficiency.
* 🚀 **Service & Boot Management**: Easily start, stop, restart, inspect status with real-time logs, and toggle boot autostart (`enable` / `disable`).

---

## 📥 Installation

`mypostgress` installs to `~/.local/bin/mypostgress`, which is included in your user `$PATH`.

### Fast Install / Update
```bash
# Clone or navigate to the repository
cd ~/App

# Run self-installer
./mypostgress install
```

Once installed, you can run `mypostgress` (or the shortcut `mypostgres`) from **any terminal or directory**.

---

## 🚀 Quick Start

### 1. Interactive Menu Mode
Simply type `mypostgress` without any arguments:

```bash
mypostgress
```

```text
════════════════════════════════════════════════════════════════
             mypostgress Universal Management Tool              
════════════════════════════════════════════════════════════════
   1. Start PostgreSQL
   2. Stop PostgreSQL
   3. Restart PostgreSQL
   4. Check Service Status
   5. Show Connection Strings (URI, psql, Node, Python, JDBC...)
   6. Open / Connect in pgAdmin 4
   7. List All Databases
   8. Backup a Database
   9. Restore a Database
  10. Initialize Database Cluster (initdb)
  11. Setup Superuser Role for 'chanduu'
  12. Install / Reinstall PostgreSQL (Multi-Distro)
  13. Install / Setup pgAdmin 4
  14. Enable PostgreSQL on Boot
  15. Disable PostgreSQL on Boot
  16. Install / Update Global Command
   0. Exit
════════════════════════════════════════════════════════════════
Select an option [0-16]: 
```

---

### 2. Direct CLI Commands

You can run any operation directly from your terminal:

```bash
# Service Control
mypostgress start                         # Start PostgreSQL (auto-detects uninitialized clusters)
mypostgress stop                          # Stop PostgreSQL service
mypostgress restart                       # Restart PostgreSQL service
mypostgress status                        # Check service status & connectivity

# Connection Strings & Tools
mypostgress conn                          # Interactively generate connection strings
mypostgress conn mydb myuser mypass       # Generate connection strings with parameters
mypostgress pgadmin                       # Launch pgAdmin 4 (with server credentials helper)

# Database Management
mypostgress list                          # List all databases with size, owner, and encoding
mypostgress backup mydb ~/backups/db.sql  # Backup database to plain SQL or .dump
mypostgress restore mydb ~/backups/db.sql # Restore database (prompts to create DB if missing)

# Setup & Maintenance
mypostgress init                          # Initialize PostgreSQL cluster (initdb)
mypostgress setup-user                    # Create matching superuser role for current Linux user
mypostgress install-postgres              # Install/reinstall PostgreSQL packages via distro package manager
mypostgress install-pgadmin               # Install pgAdmin 4 (Flatpak / Native / Docker / Pip)
mypostgress enable                        # Enable PostgreSQL to start automatically on system boot
mypostgress disable                       # Disable autostart on boot

# Help & Utility
mypostgress version                       # Show version and detected OS
mypostgress help                          # Display help message
```

---

## 📖 Feature Walkthrough

### 🔗 Connection String Generator (`mypostgress conn`)
Generates ready-to-paste configurations tailored for your stack:

```bash
mypostgress conn my_app_db dev_user my_secret_pass
```

**Output:**
```text
╔════════════════════════════════════════════════════════════════════════════════╗
║                 PostgreSQL Connection Strings Reference                        ║
╠════════════════════════════════════════════════════════════════════════════════╣
  • Database: my_app_db   • User: dev_user   • Host: localhost:5432
╚════════════════════════════════════════════════════════════════════════════════╝

1. Standard Connection URI / URL:
   postgresql://dev_user:my_secret_pass@localhost:5432/my_app_db

2. psql CLI Command:
   psql -h localhost -p 5432 -U dev_user -d my_app_db
   psql "postgresql://dev_user:my_secret_pass@localhost:5432/my_app_db"

3. Environment Variable (.env / Docker):
   DATABASE_URL="postgresql://dev_user:my_secret_pass@localhost:5432/my_app_db?schema=public"

4. Node.js (Prisma / TypeORM / Sequelize / pg / Drizzle):
   DATABASE_URL="postgresql://dev_user:my_secret_pass@localhost:5432/my_app_db"

5. Python (SQLAlchemy / psycopg2 / asyncpg / Django):
   • SQLAlchemy:  postgresql+psycopg2://dev_user:my_secret_pass@localhost:5432/my_app_db
   • asyncpg:     postgresql+asyncpg://dev_user:my_secret_pass@localhost:5432/my_app_db

6. Go (pgx / lib/pq / GORM / Bun):
   postgres://dev_user:my_secret_pass@localhost:5432/my_app_db?sslmode=disable

7. Java / Kotlin (JDBC / Spring Boot):
   jdbc:postgresql://localhost:5432/my_app_db?user=dev_user&password=my_secret_pass&sslmode=prefer

8. PHP (PDO / Laravel):
   pgsql:host=localhost;port=5432;dbname=my_app_db;user=dev_user
```

---

### 🐘 pgAdmin 4 Manager (`mypostgress pgadmin`)
* **Smart Detection**: Detects Flatpak (`org.pgadmin.pgadmin4`), native package (`/usr/bin/pgadmin4`), Snap, macOS apps, or Docker containers (`dpage/pgadmin4`).
* **Connection Parameter Summary**: Displays the exact hostname, port, database, and user to input when adding a new server in pgAdmin.
* **Auto-Installer**: If pgAdmin is not present on your system, it offers multiple installation methods (Flatpak, Native repository, Docker, or Python virtualenv).

---

### 💾 Backup & Restore

#### Backup
Automatically detects format based on filename extension:
* `.sql` $\rightarrow$ Plain SQL script
* `.dump`, `.tar`, `.backup`, `.fc` $\rightarrow$ Custom compressed archive format (`pg_dump -F c`)

```bash
# Plain SQL backup
mypostgress backup ecommerce_db ~/backups/ecommerce.sql

# Compressed custom dump
mypostgress backup ecommerce_db ~/backups/ecommerce.dump
```

#### Restore
Detects format automatically and uses `psql` or `pg_restore` accordingly. If the destination database does not exist, `mypostgress` will prompt to create it automatically:

```bash
mypostgress restore ecommerce_db ~/backups/ecommerce.dump
```

---

## 🛠️ Operating System Support Matrix

| OS / Distribution | Package Manager | Service Manager | Default Data Directory |
|---|---|---|---|
| **Arch Linux / CachyOS / Manjaro** | `pacman` | `systemd` | `/var/lib/postgres/data` |
| **Ubuntu / Debian / Linux Mint / Pop!_OS** | `apt` | `systemd` / `service` | `/var/lib/postgresql/<ver>/main` |
| **Fedora / RHEL / CentOS / Rocky** | `dnf` / `yum` | `systemd` | `/var/lib/pgsql/data` |
| **openSUSE / SLES** | `zypper` | `systemd` | `/var/lib/pgsql/data` |
| **Alpine Linux** | `apk` | `OpenRC` (`rc-service`) | `/var/lib/postgresql/data` |
| **macOS** | `brew` | `brew services` / `pg_ctl` | `/opt/homebrew/var/postgresql@16` |

---

## ⚙️ Environment Variables (Optional)

Override connection settings for remote servers or custom ports:

```bash
export PGUSER="postgres"       # Default PostgreSQL username (default: postgres / $USER)
export PGHOST="localhost"      # PostgreSQL host (default: local socket / localhost)
export PGPORT="5432"           # PostgreSQL port (default: 5432)
export PGPASSWORD="your_pass"  # PostgreSQL password
export PGDATA="/custom/path"   # Custom database cluster directory
```

---

## ❓ Troubleshooting & FAQs

#### Q: "Database directory is missing or empty" when starting PostgreSQL?
**A:** On Arch/CachyOS/Fedora, database clusters must be initialized before first start. Simply run:
```bash
mypostgress start
```
`mypostgress` will detect the uninitialized cluster, prompt you to initialize it with `initdb`, and start the service automatically.

#### Q: `psql: error: FATAL: role "username" does not exist`?
**A:** Run `mypostgress setup-user`. This creates a superuser role and default database matching your Linux username, enabling passwordless, sudo-free local connections.

#### Q: How do I enable PostgreSQL on boot?
**A:** Run:
```bash
mypostgress enable
```

---

## 📄 License

This project is licensed under the MIT License — feel free to use and customize it for your workflow!
