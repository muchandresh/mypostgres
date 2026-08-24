#!/usr/bin/env bash
# ==============================================================================
# Command: mypostgress
# Description: Universal CLI command & interactive tool for PostgreSQL operations:
#              1. Start PostgreSQL (Auto-detects distro & initializes if needed)
#              2. Stop PostgreSQL
#              3. Restart PostgreSQL
#              4. Check PostgreSQL service status
#              5. Show Connection Strings (URI, psql, Node.js, Python, Go, JDBC)
#              6. Open / Connect in pgAdmin 4 (Flatpak, Native, Snap, Docker)
#              7. List all databases
#              8. Backup a database to a specified location
#              9. Restore a database from a specified location
#             10. Initialize database cluster (initdb)
#             11. Setup superuser role for current Linux user
#             12. Universal Install / Reinstall (Arch, Debian, Ubuntu, Fedora, RHEL, Alpine, macOS)
#             13. Universal pgAdmin 4 Installer
#             14. Enable / Disable autostart on boot
# ==============================================================================

set -uo pipefail

# ANSI color codes for formatted terminal output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
MAGENTA='\033[0;35m'
BOLD='\033[1m'
NC='\033[0m' # No Color

# Script and binary information
SCRIPT_NAME="mypostgress"
VERSION="1.2.0"
INSTALL_DIR="${HOME}/.local/bin"

# Default connection settings (can be overridden by environment variables)
PG_USER="${PGUSER:-postgres}"
PG_HOST="${PGHOST:-}"
PG_PORT="${PGPORT:-5432}"

# ------------------------------------------------------------------------------
# Distro & Package Manager Detection
# ------------------------------------------------------------------------------
detect_distro() {
    if [ -f /etc/os-release ]; then
        # shellcheck source=/dev/null
        . /etc/os-release
        DISTRO_ID="${ID:-unknown}"
        DISTRO_LIKE="${ID_LIKE:-}"
        DISTRO_NAME="${PRETTY_NAME:-$NAME}"
    elif [ "$(uname -s)" = "Darwin" ]; then
        DISTRO_ID="macos"
        DISTRO_LIKE="bsd"
        DISTRO_NAME="macOS $(sw_vers -productVersion 2>/dev/null || true)"
    elif [ -f /etc/alpine-release ]; then
        DISTRO_ID="alpine"
        DISTRO_LIKE="busybox"
        DISTRO_NAME="Alpine Linux"
    else
        DISTRO_ID="unknown"
        DISTRO_LIKE=""
        DISTRO_NAME="Linux (Generic)"
    fi
}

detect_package_manager() {
    if command -v pacman &>/dev/null; then
        PKG_MGR="pacman"
    elif command -v apt-get &>/dev/null; then
        PKG_MGR="apt"
    elif command -v dnf &>/dev/null; then
        PKG_MGR="dnf"
    elif command -v yum &>/dev/null; then
        PKG_MGR="yum"
    elif command -v zypper &>/dev/null; then
        PKG_MGR="zypper"
    elif command -v apk &>/dev/null; then
        PKG_MGR="apk"
    elif command -v brew &>/dev/null; then
        PKG_MGR="brew"
    else
        PKG_MGR="unknown"
    fi
}

# ------------------------------------------------------------------------------
# 0. Check if PostgreSQL is Installed (with Auto-Installer)
# ------------------------------------------------------------------------------
check_postgres_installed() {
    local missing_tools=()

    # Check for required client/utility binaries
    for tool in psql pg_dump pg_restore; do
        if ! command -v "$tool" &> /dev/null; then
            missing_tools+=("$tool")
        fi
    done

    if [ ${#missing_tools[@]} -ne 0 ]; then
        detect_distro
        detect_package_manager
        echo -e "${YELLOW}${BOLD}[NOTICE] PostgreSQL is not installed or missing from your PATH.${NC}"
        echo -e "Missing commands: ${missing_tools[*]}"
        echo -e "Detected OS: ${CYAN}${DISTRO_NAME}${NC} (Package Manager: ${BOLD}${PKG_MGR}${NC})"
        echo ""
        read -rp "Would you like to install PostgreSQL automatically now? (Y/n): " do_install
        if [[ ! "$do_install" =~ ^[nN](o)?$ ]]; then
            install_postgresql_packages
        else
            echo -e "${YELLOW}Please install PostgreSQL using your package manager:${NC}"
            echo -e "  • Arch Linux / CachyOS: ${CYAN}sudo pacman -S postgresql postgresql-contrib${NC}"
            echo -e "  • Debian / Ubuntu:      ${CYAN}sudo apt update && sudo apt install postgresql postgresql-contrib${NC}"
            echo -e "  • RHEL / Fedora:        ${CYAN}sudo dnf install postgresql-server postgresql${NC}"
            echo -e "  • openSUSE:             ${CYAN}sudo zypper install postgresql-server postgresql${NC}"
            echo -e "  • Alpine:               ${CYAN}sudo apk add postgresql postgresql-client${NC}"
            echo -e "  • macOS:                ${CYAN}brew install postgresql@16${NC}"
            echo ""
            exit 1
        fi
    fi
}

# ------------------------------------------------------------------------------
# Helper: Check if PostgreSQL server is actively running
# ------------------------------------------------------------------------------
is_postgres_running() {
    if command -v systemctl &>/dev/null; then
        if systemctl is-active --quiet postgresql 2>/dev/null; then
            return 0
        fi
    fi
    if command -v pg_isready &>/dev/null; then
        if pg_isready -h "${PG_HOST:-localhost}" -p "$PG_PORT" &>/dev/null; then
            return 0
        fi
    fi
    if execute_psql "SELECT 1;" >/dev/null 2>&1; then
        return 0
    fi
    return 1
}

# Helper to run psql queries with proper privilege/credentials
execute_psql() {
    local query="$1"
    local db="${2:-postgres}"

    if [ -n "$PG_HOST" ]; then
        psql -h "$PG_HOST" -p "$PG_PORT" -U "$PG_USER" -d "$db" -c "$query"
    elif [ "$(id -u)" -eq 0 ]; then
        su - postgres -c "psql -d '$db' -c \"$query\""
    else
        # 1. Try direct connection with default credentials / current user
        if psql -d "$db" -c "$query" 2>/dev/null; then
            return 0
        fi
        # 2. Try direct connection with PG_USER
        if psql -U "$PG_USER" -d "$db" -c "$query" 2>/dev/null; then
            return 0
        fi
        # 3. Fallback to sudo -u postgres
        sudo -u postgres psql -d "$db" -c "$query"
    fi
}

# ------------------------------------------------------------------------------
# 1. Start PostgreSQL Service (Multi-Distro)
# ------------------------------------------------------------------------------
start_postgres() {
    echo -e "${CYAN}==> Starting PostgreSQL service...${NC}"

    if is_postgres_running; then
        echo -e "${GREEN}● PostgreSQL is already running.${NC}"
        return 0
    fi

    # Check for uninitialized database cluster before starting
    local default_data_dir="/var/lib/postgres/data"
    if [ -n "${PGDATA:-}" ]; then
        default_data_dir="$PGDATA"
    elif [ -d "/var/lib/pgsql" ]; then
        default_data_dir="/var/lib/pgsql/data"
    elif [ -d "/var/lib/postgresql" ] && [ ! -d "/var/lib/postgres" ]; then
        default_data_dir="/var/lib/postgresql/data"
    fi

    local uninit_detected=false
    if [ -f "/usr/bin/postgresql-check-db-dir" ]; then
        if ! /usr/bin/postgresql-check-db-dir "$default_data_dir" &>/dev/null; then
            uninit_detected=true
        fi
    elif [ -d "$(dirname "$default_data_dir")" ] && ! sudo test -f "${default_data_dir}/PG_VERSION" 2>/dev/null; then
        # Only treat as uninit if Debian clusters aren't present
        if [ ! -d "/etc/postgresql" ] && [ ! -d "/var/lib/postgresql" ]; then
            uninit_detected=true
        fi
    fi

    if [ "$uninit_detected" = true ]; then
        echo -e "${YELLOW}⚠ PostgreSQL database cluster is not initialized (${default_data_dir} is missing or empty).${NC}"
        read -rp "Would you like to initialize the database cluster now with initdb? (Y/n): " do_init
        if [[ ! "$do_init" =~ ^[nN](o)?$ ]]; then
            if ! init_postgres_cluster "$default_data_dir"; then
                echo -e "${RED}✗ Cannot start PostgreSQL without initialized cluster.${NC}"
                return 1
            fi
        else
            echo -e "${RED}✗ PostgreSQL cannot start without an initialized database directory.${NC}"
            echo -e "${YELLOW}Run 'mypostgress init' or 'sudo -u postgres initdb -D ${default_data_dir}' first.${NC}"
            return 1
        fi
    fi

    # Attempt to start service across different init systems
    local start_success=false
    if command -v systemctl &> /dev/null; then
        if sudo systemctl start postgresql; then
            start_success=true
        else
            echo -e "${RED}✗ Failed to start PostgreSQL service using systemctl.${NC}"
            echo -e "${YELLOW}Inspecting recent service logs:${NC}"
            journalctl -u postgresql -n 8 --no-pager 2>/dev/null || systemctl status postgresql --no-pager 2>/dev/null || true

            # If uninitialized cluster error is detected in logs, offer initdb
            local journal_err
            journal_err=$(journalctl -u postgresql -n 10 --no-pager 2>/dev/null || true)
            if echo "$journal_err" | grep -qi "is missing or empty\|initdb"; then
                echo ""
                echo -e "${YELLOW}The service failed because the database directory is uninitialized.${NC}"
                read -rp "Initialize database cluster now? (Y/n): " do_init_retry
                if [[ ! "$do_init_retry" =~ ^[nN](o)?$ ]]; then
                    if init_postgres_cluster "$default_data_dir"; then
                        echo -e "${CYAN}Retrying to start PostgreSQL service...${NC}"
                        if sudo systemctl start postgresql; then
                            start_success=true
                        fi
                    fi
                fi
            fi
        fi
    elif command -v rc-service &> /dev/null; then
        if sudo rc-service postgresql start; then
            start_success=true
        fi
    elif command -v service &> /dev/null; then
        if sudo service postgresql start; then
            start_success=true
        else
            echo -e "${RED}✗ Failed to start PostgreSQL service using service.${NC}"
            sudo service postgresql status 2>/dev/null || true
        fi
    elif command -v brew &> /dev/null && [ "$(uname -s)" = "Darwin" ]; then
        if brew services start postgresql@16 2>/dev/null || brew services start postgresql 2>/dev/null; then
            start_success=true
        fi
    elif command -v pg_ctl &> /dev/null && [ -n "${PGDATA:-}" ]; then
        if pg_ctl -D "$PGDATA" start; then
            start_success=true
        else
            echo -e "${RED}✗ Failed to start PostgreSQL via pg_ctl.${NC}"
        fi
    else
        echo -e "${RED}[ERROR] No supported service manager found to start PostgreSQL.${NC}"
        return 1
    fi

    if [ "$start_success" = true ]; then
        echo -e "${GREEN}✓ PostgreSQL service started successfully.${NC}"
        return 0
    else
        echo -e "${RED}✗ Could not start PostgreSQL.${NC}"
        return 1
    fi
}

# ------------------------------------------------------------------------------
# 2. Stop PostgreSQL Service
# ------------------------------------------------------------------------------
stop_postgres() {
    echo -e "${CYAN}==> Stopping PostgreSQL service...${NC}"

    if command -v systemctl &> /dev/null; then
        if sudo systemctl stop postgresql; then
            echo -e "${GREEN}✓ PostgreSQL service stopped successfully.${NC}"
            return 0
        else
            echo -e "${RED}✗ Failed to stop PostgreSQL service using systemctl.${NC}"
            return 1
        fi
    elif command -v rc-service &> /dev/null; then
        if sudo rc-service postgresql stop; then
            echo -e "${GREEN}✓ PostgreSQL service stopped successfully.${NC}"
            return 0
        fi
    elif command -v service &> /dev/null; then
        if sudo service postgresql stop; then
            echo -e "${GREEN}✓ PostgreSQL service stopped successfully.${NC}"
            return 0
        else
            echo -e "${RED}✗ Failed to stop PostgreSQL service using service.${NC}"
            return 1
        fi
    elif command -v brew &> /dev/null && [ "$(uname -s)" = "Darwin" ]; then
        brew services stop postgresql@16 2>/dev/null || brew services stop postgresql 2>/dev/null
        echo -e "${GREEN}✓ PostgreSQL stopped successfully.${NC}"
        return 0
    elif command -v pg_ctl &> /dev/null && [ -n "${PGDATA:-}" ]; then
        if pg_ctl -D "$PGDATA" stop; then
            echo -e "${GREEN}✓ PostgreSQL stopped successfully via pg_ctl.${NC}"
            return 0
        else
            echo -e "${RED}✗ Failed to stop PostgreSQL via pg_ctl.${NC}"
            return 1
        fi
    else
        echo -e "${RED}[ERROR] No supported service manager found to stop PostgreSQL.${NC}"
        return 1
    fi
}

# ------------------------------------------------------------------------------
# 3. Restart PostgreSQL Service
# ------------------------------------------------------------------------------
restart_postgres() {
    echo -e "${CYAN}==> Restarting PostgreSQL service...${NC}"
    if command -v systemctl &> /dev/null; then
        if sudo systemctl restart postgresql; then
            echo -e "${GREEN}✓ PostgreSQL service restarted successfully.${NC}"
            return 0
        fi
    fi
    stop_postgres && sleep 1 && start_postgres
}

# ------------------------------------------------------------------------------
# 4. Service Status Helper
# ------------------------------------------------------------------------------
status_postgres() {
    echo -e "${CYAN}==> Checking PostgreSQL service status...${NC}"
    if command -v systemctl &> /dev/null; then
        if systemctl is-active --quiet postgresql 2>/dev/null; then
            echo -e "${GREEN}● PostgreSQL is ACTIVE (running).${NC}"
            if command -v pg_isready &>/dev/null; then
                pg_isready -h "${PG_HOST:-localhost}" -p "$PG_PORT" || true
            fi
        else
            echo -e "${YELLOW}○ PostgreSQL is INACTIVE (stopped).${NC}"
        fi
        echo ""
        systemctl status postgresql --no-pager -l || true
    elif command -v rc-service &> /dev/null; then
        sudo rc-service postgresql status
    elif command -v service &> /dev/null; then
        sudo service postgresql status
    else
        if execute_psql "SELECT 1;" > /dev/null 2>&1; then
            echo -e "${GREEN}● PostgreSQL server is accepting connections.${NC}"
        else
            echo -e "${YELLOW}○ PostgreSQL server is not responding.${NC}"
        fi
    fi
}

# ------------------------------------------------------------------------------
# 5. Show Connection Strings (Multi-Language / Multi-Tool)
# ------------------------------------------------------------------------------
show_connection_strings() {
    local db_name="${1:-}"
    local user="${2:-}"
    local password="${3:-}"
    local host="${PG_HOST:-localhost}"
    local port="${PG_PORT:-5432}"

    echo -e "${CYAN}==> PostgreSQL Connection Strings Generator${NC}"
    echo ""

    # Prompt for missing fields interactively if not provided
    if [ -z "$db_name" ]; then
        read -rp "Enter Database Name [Default: postgres]: " input_db
        db_name="${input_db:-postgres}"
    fi

    if [ -z "$user" ]; then
        read -rp "Enter Database User [Default: ${PGUSER:-$USER}]: " input_user
        user="${input_user:-${PGUSER:-$USER}}"
    fi

    if [ $# -lt 3 ]; then
        read -rsp "Enter Database Password (optional, press Enter if none): " input_pass
        echo ""
        password="${input_pass:-$password}"
    fi

    # Build auth string
    local auth_part="$user"
    local auth_masked="$user"
    local pass_param=""
    if [ -n "$password" ]; then
        auth_part="${user}:${password}"
        auth_masked="${user}:********"
        pass_param="password=${password}&"
    fi

    local conn_uri="postgresql://${auth_part}@${host}:${port}/${db_name}"
    local psql_cmd="psql -h ${host} -p ${port} -U ${user} -d ${db_name}"
    local psql_uri_cmd="psql \"${conn_uri}\""

    echo ""
    echo -e "${BOLD}${BLUE}╔════════════════════════════════════════════════════════════════════════════════╗${NC}"
    echo -e "${BOLD}${BLUE}║                 PostgreSQL Connection Strings Reference                        ║${NC}"
    echo -e "${BOLD}${BLUE}╠════════════════════════════════════════════════════════════════════════════════╣${NC}"
    echo -e "  • ${BOLD}Database:${NC} ${CYAN}${db_name}${NC}   • ${BOLD}User:${NC} ${CYAN}${user}${NC}   • ${BOLD}Host:${NC} ${CYAN}${host}:${port}${NC}"
    echo -e "${BOLD}${BLUE}╚════════════════════════════════════════════════════════════════════════════════╝${NC}"
    echo ""

    echo -e "${BOLD}${MAGENTA}1. Standard Connection URI / URL:${NC}"
    echo -e "   ${GREEN}${conn_uri}${NC}"
    echo ""

    echo -e "${BOLD}${MAGENTA}2. psql CLI Command:${NC}"
    echo -e "   ${GREEN}${psql_cmd}${NC}"
    echo -e "   ${GREEN}${psql_uri_cmd}${NC}"
    echo ""

    echo -e "${BOLD}${MAGENTA}3. Environment Variable (.env / Docker):${NC}"
    echo -e "   ${GREEN}DATABASE_URL=\"${conn_uri}?schema=public\"${NC}"
    echo ""

    echo -e "${BOLD}${MAGENTA}4. Node.js (Prisma / TypeORM / Sequelize / pg / Drizzle):${NC}"
    echo -e "   ${GREEN}DATABASE_URL=\"${conn_uri}\"${NC}"
    echo ""

    echo -e "${BOLD}${MAGENTA}5. Python (SQLAlchemy / psycopg2 / asyncpg / Django):${NC}"
    echo -e "   • SQLAlchemy:  ${GREEN}postgresql+psycopg2://${auth_part}@${host}:${port}/${db_name}${NC}"
    echo -e "   • asyncpg:     ${GREEN}postgresql+asyncpg://${auth_part}@${host}:${port}/${db_name}${NC}"
    echo ""

    echo -e "${BOLD}${MAGENTA}6. Go (pgx / lib/pq / GORM / Bun):${NC}"
    echo -e "   ${GREEN}postgres://${auth_part}@${host}:${port}/${db_name}?sslmode=disable${NC}"
    echo ""

    echo -e "${BOLD}${MAGENTA}7. Java / Kotlin (JDBC / Spring Boot):${NC}"
    echo -e "   ${GREEN}jdbc:postgresql://${host}:${port}/${db_name}?user=${user}&${pass_param}sslmode=prefer${NC}"
    echo ""

    echo -e "${BOLD}${MAGENTA}8. PHP (PDO / Laravel):${NC}"
    echo -e "   ${GREEN}pgsql:host=${host};port=${port};dbname=${db_name};user=${user}${NC}"
    echo ""
}

# ------------------------------------------------------------------------------
# 6. Open / Connect in pgAdmin 4 (Universal Launcher & Installer)
# ------------------------------------------------------------------------------
find_pgadmin_cmd() {
    # Check Flatpak
    if command -v flatpak &>/dev/null && flatpak list --app 2>/dev/null | grep -qi "org.pgadmin.pgadmin4"; then
        echo "flatpak run org.pgadmin.pgadmin4"
        return 0
    fi
    # Check Native binaries
    for bin in pgadmin4 pgadmin /usr/pgadmin4/bin/pgadmin4 /opt/pgadmin4/bin/pgadmin4; do
        if command -v "$bin" &>/dev/null; then
            echo "$bin"
            return 0
        fi
    done
    # Check Snap
    if command -v snap &>/dev/null && snap list 2>/dev/null | grep -qi "pgadmin4"; then
        echo "snap run pgadmin4"
        return 0
    fi
    # Check macOS app
    if [ -d "/Applications/pgAdmin 4.app" ]; then
        echo "open -a 'pgAdmin 4'"
        return 0
    fi
    return 1
}

open_pgadmin() {
    echo -e "${CYAN}==> Connecting to pgAdmin 4...${NC}"

    # Ensure PostgreSQL is running first
    if ! is_postgres_running; then
        echo -e "${YELLOW}PostgreSQL service is currently inactive.${NC}"
        read -rp "Would you like to start PostgreSQL service first? (Y/n): " start_choice
        if [[ ! "$start_choice" =~ ^[nN](o)?$ ]]; then
            start_postgres
        fi
    fi

    # Display server connection parameters for reference in pgAdmin
    echo ""
    echo -e "${BOLD}${BLUE}╔══════════════════════════════════════════════════════════════╗${NC}"
    echo -e "${BOLD}${BLUE}║             pgAdmin Connection Parameters                     ║${NC}"
    echo -e "${BOLD}${BLUE}╠══════════════════════════════════════════════════════════════╣${NC}"
    echo -e "  • ${BOLD}Host name/address:${NC} ${CYAN}${PG_HOST:-localhost}${NC} (or 127.0.0.1)"
    echo -e "  • ${BOLD}Port:${NC}              ${CYAN}${PG_PORT:-5432}${NC}"
    echo -e "  • ${BOLD}Maintenance DB:${NC}    ${CYAN}postgres${NC}"
    echo -e "  • ${BOLD}Username:${NC}          ${CYAN}${PG_USER:-$USER}${NC}"
    echo -e "${BOLD}${BLUE}╚══════════════════════════════════════════════════════════════╝${NC}"
    echo ""

    local launch_cmd
    if launch_cmd=$(find_pgadmin_cmd); then
        echo -e "${GREEN}Launching pgAdmin 4 (${launch_cmd})...${NC}"
        nohup $launch_cmd >/dev/null 2>&1 &
        echo -e "${GREEN}✓ pgAdmin 4 launched in the background.${NC}"
        return 0
    fi

    # Check if pgAdmin container is running
    if command -v docker &>/dev/null && docker ps 2>/dev/null | grep -qi "pgadmin"; then
        echo -e "${GREEN}pgAdmin container detected. Opening http://localhost:5050 in browser...${NC}"
        if command -v xdg-open &>/dev/null; then
            xdg-open "http://localhost:5050" >/dev/null 2>&1 &
        elif command -v open &>/dev/null; then
            open "http://localhost:5050"
        fi
        return 0
    fi

    # Not found
    echo -e "${YELLOW}pgAdmin 4 is not installed on this system.${NC}"
    read -rp "Would you like to install pgAdmin 4 now? (Y/n): " do_install
    if [[ ! "$do_install" =~ ^[nN](o)?$ ]]; then
        if install_pgadmin; then
            if launch_cmd=$(find_pgadmin_cmd); then
                nohup $launch_cmd >/dev/null 2>&1 &
                echo -e "${GREEN}✓ pgAdmin 4 launched!${NC}"
            fi
        fi
    fi
}

install_pgadmin() {
    detect_distro
    detect_package_manager

    echo -e "${CYAN}==> Installing pgAdmin 4...${NC}"
    echo -e "Choose an installation method:"
    echo -e "  ${BOLD}1.${NC} Flatpak (Universal, recommended across all Linux distros)"
    echo -e "  ${BOLD}2.${NC} System Package Manager (${PKG_MGR})"
    echo -e "  ${BOLD}3.${NC} Docker container (dpage/pgadmin4)"
    echo -e "  ${BOLD}4.${NC} Python Virtualenv / pipx"
    read -rp "Select method [1-4, default: 1]: " pgadmin_method
    pgadmin_method="${pgadmin_method:-1}"

    case "$pgadmin_method" in
        1)
            if ! command -v flatpak &>/dev/null; then
                echo -e "${YELLOW}Flatpak is not installed. Installing flatpak...${NC}"
                case "$PKG_MGR" in
                    pacman) sudo pacman -S --needed flatpak ;;
                    apt) sudo apt-get update && sudo apt-get install -y flatpak ;;
                    dnf) sudo dnf install -y flatpak ;;
                    zypper) sudo zypper install -y flatpak ;;
                    apk) sudo apk add flatpak ;;
                    *) echo -e "${RED}Please install flatpak first.${NC}" ;;
                esac
            fi
            echo -e "${BLUE}Configuring Flathub repo...${NC}"
            flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo 2>/dev/null || true
            echo -e "${BLUE}Installing pgAdmin 4 via Flatpak...${NC}"
            flatpak install -y flathub org.pgadmin.pgadmin4
            ;;
        2)
            case "$PKG_MGR" in
                pacman)
                    if command -v yay &>/dev/null; then
                        yay -S --needed pgadmin4-desktop
                    elif command -v paru &>/dev/null; then
                        paru -S --needed pgadmin4-desktop
                    else
                        echo -e "${YELLOW}No AUR helper found. Installing via Flatpak instead...${NC}"
                        flatpak install -y flathub org.pgadmin.pgadmin4
                    fi
                    ;;
                apt)
                    echo -e "${BLUE}Setting up official pgAdmin repository...${NC}"
                    curl -fsS https://www.pgadmin.org/static/packages_pgadmin_org.pub | sudo gpg --dearmor -o /usr/share/keyrings/packages-pgadmin-org.gpg 2>/dev/null || true
                    sudo sh -c 'echo "deb [signed-by=/usr/share/keyrings/packages-pgadmin-org.gpg] https://ftp.postgresql.org/pub/pgadmin/pgadmin4/apt/$(lsb_release -cs) pgadmin4 main" > /etc/apt/sources.list.d/pgadmin4.list'
                    sudo apt-get update && (sudo apt-get install -y pgadmin4-desktop || sudo apt-get install -y pgadmin4)
                    ;;
                dnf)
                    sudo dnf install -y https://ftp.postgresql.org/pub/pgadmin/pgadmin4/redhat/pgadmin4-fedora-repo-2-1.noarch.rpm
                    sudo dnf install -y pgadmin4-desktop
                    ;;
                brew)
                    brew install --cask pgadmin4
                    ;;
                *)
                    echo -e "${YELLOW}Falling back to Flatpak...${NC}"
                    flatpak install -y flathub org.pgadmin.pgadmin4
                    ;;
            esac
            ;;
        3)
            if command -v docker &>/dev/null; then
                echo -e "${BLUE}Starting pgAdmin Docker container on port 5050...${NC}"
                docker run -d --name pgadmin4 -p 5050:80 -e "PGADMIN_DEFAULT_EMAIL=admin@local.host" -e "PGADMIN_DEFAULT_PASSWORD=admin" dpage/pgadmin4
                echo -e "${GREEN}✓ pgAdmin container running! Access at http://localhost:5050 (Login: admin@local.host / admin)${NC}"
            else
                echo -e "${RED}Docker is not installed.${NC}"
                return 1
            fi
            ;;
        4)
            if command -v pipx &>/dev/null; then
                pipx install pgadmin4
            else
                python3 -m pip install --user pgadmin4
            fi
            ;;
        *)
            echo -e "${RED}Invalid choice.${NC}"
            return 1
            ;;
    esac
}

# ------------------------------------------------------------------------------
# 7. List All Databases
# ------------------------------------------------------------------------------
list_databases() {
    echo -e "${CYAN}==> Listing all PostgreSQL databases...${NC}"
    echo ""

    local query="SELECT datname AS \"Database Name\", 
                        pg_catalog.pg_get_userbyid(datdba) AS \"Owner\",
                        pg_catalog.pg_size_pretty(pg_catalog.pg_database_size(datname)) AS \"Size\",
                        pg_catalog.pg_encoding_to_char(encoding) AS \"Encoding\"
                 FROM pg_catalog.pg_database 
                 WHERE datistemplate = false 
                 ORDER BY datname;"

    if ! execute_psql "$query"; then
        echo -e "${RED}✗ Failed to retrieve databases. Ensure PostgreSQL is running.${NC}"
        return 1
    fi
}

# ------------------------------------------------------------------------------
# 8. Backup a Database to a Specific Location
# ------------------------------------------------------------------------------
backup_database() {
    local db_name="${1:-}"
    local backup_path="${2:-}"

    # Prompt if not supplied via arguments
    if [ -z "$db_name" ]; then
        read -rp "Enter Database Name to backup: " db_name
    fi

    if [ -z "$db_name" ]; then
        echo -e "${RED}[ERROR] Database name cannot be empty.${NC}"
        return 1
    fi

    if [ -z "$backup_path" ]; then
        local timestamp
        timestamp=$(date +"%Y%m%d_%H%M%S")
        local default_path="./backups/${db_name}_${timestamp}.sql"
        read -rp "Enter destination backup file path [Default: $default_path]: " backup_path
        backup_path="${backup_path:-$default_path}"
    fi

    # Expand tilde if present
    backup_path="${backup_path/#\~/$HOME}"

    # Ensure parent directory exists
    local backup_dir
    backup_dir=$(dirname "$backup_path")
    if [ ! -d "$backup_dir" ]; then
        mkdir -p "$backup_dir"
        echo -e "${BLUE}Created directory: $backup_dir${NC}"
    fi

    echo -e "${CYAN}==> Backing up database '${db_name}' to '${backup_path}'...${NC}"

    # Format detection
    local is_custom_format=false
    if [[ "$backup_path" =~ \.(dump|tar|backup|fc)$ ]]; then
        is_custom_format=true
    fi

    local status=0
    if [ "$is_custom_format" = true ]; then
        if [ -n "$PG_HOST" ]; then
            pg_dump -h "$PG_HOST" -p "$PG_PORT" -U "$PG_USER" -F c -b -v -f "$backup_path" "$db_name"
            status=$?
        elif pg_dump -F c -b -v -f "$backup_path" "$db_name" 2>/dev/null; then
            status=0
        elif pg_dump -U "$PG_USER" -F c -b -v -f "$backup_path" "$db_name" 2>/dev/null; then
            status=0
        else
            sudo -u postgres pg_dump -F c -b -v -f "$backup_path" "$db_name"
            status=$?
        fi
    else
        if [ -n "$PG_HOST" ]; then
            pg_dump -h "$PG_HOST" -p "$PG_PORT" -U "$PG_USER" "$db_name" > "$backup_path"
            status=$?
        elif pg_dump "$db_name" > "$backup_path" 2>/dev/null; then
            status=0
        elif pg_dump -U "$PG_USER" "$db_name" > "$backup_path" 2>/dev/null; then
            status=0
        else
            sudo -u postgres pg_dump "$db_name" > "$backup_path"
            status=$?
        fi
    fi

    if [ $status -eq 0 ] && [ -f "$backup_path" ]; then
        local file_size
        file_size=$(du -h "$backup_path" | cut -f1)
        echo -e "${GREEN}✓ Backup successfully completed!${NC}"
        echo -e "  • File: ${BOLD}${backup_path}${NC}"
        echo -e "  • Size: ${BOLD}${file_size}${NC}"
    else
        echo -e "${RED}✗ Backup failed.${NC}"
        return 1
    fi
}

# ------------------------------------------------------------------------------
# 9. Restore a Database from a Specific Location
# ------------------------------------------------------------------------------
restore_database() {
    local db_name="${1:-}"
    local backup_path="${2:-}"

    # Prompt if not supplied via arguments
    if [ -z "$backup_path" ]; then
        read -rp "Enter path of the backup file to restore: " backup_path
    fi

    backup_path="${backup_path/#\~/$HOME}"

    if [ -z "$backup_path" ] || [ ! -f "$backup_path" ]; then
        echo -e "${RED}[ERROR] Backup file does not exist: '${backup_path}'${NC}"
        return 1
    fi

    if [ -z "$db_name" ]; then
        read -rp "Enter target Database Name to restore into: " db_name
    fi

    if [ -z "$db_name" ]; then
        echo -e "${RED}[ERROR] Target database name cannot be empty.${NC}"
        return 1
    fi

    # Check if target database exists; if not, ask to create it
    echo -e "${BLUE}Checking if target database '${db_name}' exists...${NC}"
    local db_exists_query="SELECT 1 FROM pg_database WHERE datname='${db_name}';"
    local db_exists
    db_exists=$(execute_psql "$db_exists_query" "postgres" 2>/dev/null | grep -E "1 row|1" || true)

    if [ -z "$db_exists" ]; then
        echo -e "${YELLOW}Database '${db_name}' does not exist.${NC}"
        read -rp "Create database '${db_name}' now? (y/N): " create_choice
        if [[ "$create_choice" =~ ^[yY](es)?$ ]]; then
            if execute_psql "CREATE DATABASE \"${db_name}\";" "postgres"; then
                echo -e "${GREEN}✓ Database '${db_name}' created successfully.${NC}"
            else
                echo -e "${RED}✗ Failed to create database '${db_name}'.${NC}"
                return 1
            fi
        else
            echo -e "${YELLOW}Restore aborted because target database does not exist.${NC}"
            return 1
        fi
    fi

    echo -e "${CYAN}==> Restoring backup '${backup_path}' into database '${db_name}'...${NC}"

    # Detect file type (Plain SQL vs pg_dump Custom/Tar Archive)
    local is_custom_format=false
    if command -v file &>/dev/null && file "$backup_path" | grep -qi "PostgreSQL custom database dump"; then
        is_custom_format=true
    elif [[ "$backup_path" =~ \.(dump|tar|backup|fc)$ ]]; then
        is_custom_format=true
    fi

    local status=0
    if [ "$is_custom_format" = true ]; then
        echo -e "${BLUE}Using pg_restore (Custom/Archive format)...${NC}"
        if [ -n "$PG_HOST" ]; then
            pg_restore -h "$PG_HOST" -p "$PG_PORT" -U "$PG_USER" -d "$db_name" -v "$backup_path"
            status=$?
        elif pg_restore -d "$db_name" -v "$backup_path" 2>/dev/null; then
            status=0
        elif pg_restore -U "$PG_USER" -d "$db_name" -v "$backup_path" 2>/dev/null; then
            status=0
        else
            sudo -u postgres pg_restore -d "$db_name" -v "$backup_path"
            status=$?
        fi
    else
        echo -e "${BLUE}Using psql (Plain SQL format)...${NC}"
        if [ -n "$PG_HOST" ]; then
            psql -h "$PG_HOST" -p "$PG_PORT" -U "$PG_USER" -d "$db_name" -f "$backup_path"
            status=$?
        elif psql -d "$db_name" -f "$backup_path" 2>/dev/null; then
            status=0
        elif psql -U "$PG_USER" -d "$db_name" -f "$backup_path" 2>/dev/null; then
            status=0
        else
            sudo -u postgres psql -d "$db_name" -f "$backup_path"
            status=$?
        fi
    fi

    if [ $status -eq 0 ]; then
        echo -e "${GREEN}✓ Database '${db_name}' restored successfully from '${backup_path}'!${NC}"
    else
        echo -e "${YELLOW}⚠ Restore process completed with warnings or non-zero status ($status). Check output above.${NC}"
    fi
}

# ------------------------------------------------------------------------------
# 10. Initialize PostgreSQL Database Cluster (initdb)
# ------------------------------------------------------------------------------
init_postgres_cluster() {
    local target_dir="${1:-}"

    echo -e "${CYAN}==> Initializing PostgreSQL database cluster...${NC}"

    # Determine default data directory if not specified
    if [ -z "$target_dir" ]; then
        if [ -n "${PGDATA:-}" ]; then
            target_dir="$PGDATA"
        elif [ -d "/var/lib/postgres" ] || [ -f "/usr/bin/postgresql-check-db-dir" ]; then
            target_dir="/var/lib/postgres/data"
        elif [ -d "/var/lib/pgsql" ]; then
            target_dir="/var/lib/pgsql/data"
        elif [ -d "/var/lib/postgresql" ]; then
            target_dir="/var/lib/postgresql/data"
        else
            target_dir="/var/lib/postgres/data"
        fi
    fi

    # Check if target_dir is already initialized
    local is_already_init=false
    if sudo test -f "${target_dir}/PG_VERSION" 2>/dev/null; then
        is_already_init=true
    fi

    if [ "$is_already_init" = true ]; then
        echo -e "${YELLOW}Database cluster at '${target_dir}' is already initialized.${NC}"
        read -rp "Do you want to re-initialize? WARNING: This will fail or overwrite existing data. (y/N): " reinit_choice
        if [[ ! "$reinit_choice" =~ ^[yY](es)?$ ]]; then
            echo -e "${BLUE}Initialization cancelled.${NC}"
            return 0
        fi
    fi

    echo -e "${BLUE}Initializing database cluster at: ${BOLD}${target_dir}${NC}"

    local init_success=false

    # Method 1: On Fedora/RHEL with postgresql-setup
    if command -v postgresql-setup &>/dev/null; then
        echo -e "${BLUE}Running 'postgresql-setup --initdb'...${NC}"
        if sudo postgresql-setup --initdb; then
            init_success=true
        fi
    fi

    # Method 2: sudo -u postgres initdb
    if [ "$init_success" = false ] && command -v initdb &>/dev/null; then
        echo -e "${BLUE}Running 'initdb' as postgres user...${NC}"
        if sudo -u postgres initdb --locale=C.UTF-8 --encoding=UTF8 -D "$target_dir"; then
            init_success=true
        elif sudo su -l postgres -c "initdb --locale=C.UTF-8 --encoding=UTF8 -D '${target_dir}'"; then
            init_success=true
        elif sudo -u postgres initdb -D "$target_dir"; then
            init_success=true
        fi
    fi

    # Method 3: pg_createcluster (Debian/Ubuntu)
    if [ "$init_success" = false ] && command -v pg_createcluster &>/dev/null; then
        local pg_ver
        pg_ver=$(pg_config --version 2>/dev/null | awk '{print $2}' | cut -d. -f1 || echo "16")
        echo -e "${BLUE}Running 'pg_createcluster ${pg_ver} main'...${NC}"
        if sudo pg_createcluster "$pg_ver" main --start; then
            init_success=true
        fi
    fi

    if [ "$init_success" = true ]; then
        echo -e "${GREEN}✓ PostgreSQL database cluster initialized successfully!${NC}"
        
        # Prompt to create user role for current user
        if [ "$USER" != "postgres" ] && [ "$USER" != "root" ]; then
            echo ""
            echo -e "${CYAN}==> User Configuration${NC}"
            echo -e "Would you like to create a PostgreSQL superuser role for your Linux user '${BOLD}${USER}${NC}'?"
            echo -e "This allows you to run psql, pg_dump, and mypostgress without entering sudo password."
            read -rp "Create superuser role for '${USER}'? (Y/n): " create_user_choice
            if [[ ! "$create_user_choice" =~ ^[nN](o)?$ ]]; then
                setup_user_role "$USER"
            fi
        fi
        return 0
    else
        echo -e "${RED}✗ Failed to initialize PostgreSQL database cluster.${NC}"
        echo -e "${YELLOW}Manual command you can run in your terminal:${NC}"
        echo -e "  ${CYAN}sudo -u postgres initdb --locale=C.UTF-8 --encoding=UTF8 -D '${target_dir}'${NC}"
        return 1
    fi
}

# ------------------------------------------------------------------------------
# 11. Setup Superuser Role for Current User
# ------------------------------------------------------------------------------
setup_user_role() {
    local target_user="${1:-$USER}"
    echo -e "${CYAN}==> Setting up PostgreSQL superuser role for '${target_user}'...${NC}"

    # Ensure postgres service is running first
    if ! is_postgres_running; then
        echo -e "${BLUE}Starting PostgreSQL service to configure user role...${NC}"
        if ! start_postgres; then
            echo -e "${RED}✗ Cannot create user role because PostgreSQL service failed to start.${NC}"
            return 1
        fi
        sleep 1
    fi

    # Create superuser role
    if sudo -u postgres createuser --superuser "$target_user" 2>/dev/null; then
        echo -e "${GREEN}✓ Superuser role '${target_user}' created.${NC}"
    elif sudo -u postgres psql -c "ALTER ROLE \"${target_user}\" WITH SUPERUSER;" 2>/dev/null; then
        echo -e "${GREEN}✓ Role '${target_user}' granted superuser privileges.${NC}"
    else
        if execute_psql "CREATE ROLE \"${target_user}\" WITH SUPERUSER LOGIN CREATEDB CREATEROLE;" >/dev/null 2>&1; then
            echo -e "${GREEN}✓ Superuser role '${target_user}' created.${NC}"
        else
            echo -e "${YELLOW}Note: Role '${target_user}' may already exist.${NC}"
        fi
    fi

    # Create default user database
    if sudo -u postgres createdb "$target_user" 2>/dev/null; then
        echo -e "${GREEN}✓ Default database '${target_user}' created.${NC}"
    elif execute_psql "CREATE DATABASE \"${target_user}\";" >/dev/null 2>&1; then
        echo -e "${GREEN}✓ Default database '${target_user}' created.${NC}"
    fi

    echo -e "${GREEN}✓ PostgreSQL user setup complete! You can now use psql and mypostgress seamlessly.${NC}"
}

# ------------------------------------------------------------------------------
# 12. Universal Package Installer (PostgreSQL + pgAdmin)
# ------------------------------------------------------------------------------
install_postgresql_packages() {
    detect_distro
    detect_package_manager

    echo -e "${CYAN}==> Installing PostgreSQL for ${BOLD}${DISTRO_NAME}${NC} (via ${BOLD}${PKG_MGR}${NC})...${NC}"

    case "$PKG_MGR" in
        pacman)
            echo -e "${BLUE}Running: sudo pacman -Sy --needed postgresql postgresql-contrib${NC}"
            sudo pacman -Sy --needed postgresql postgresql-contrib
            ;;
        apt)
            echo -e "${BLUE}Running: sudo apt-get update && sudo apt-get install -y postgresql postgresql-contrib postgresql-client${NC}"
            sudo apt-get update && sudo apt-get install -y postgresql postgresql-contrib postgresql-client
            ;;
        dnf)
            echo -e "${BLUE}Running: sudo dnf install -y postgresql-server postgresql postgresql-contrib${NC}"
            sudo dnf install -y postgresql-server postgresql postgresql-contrib
            ;;
        yum)
            echo -e "${BLUE}Running: sudo yum install -y postgresql-server postgresql postgresql-contrib${NC}"
            sudo yum install -y postgresql-server postgresql postgresql-contrib
            ;;
        zypper)
            echo -e "${BLUE}Running: sudo zypper install -y postgresql-server postgresql${NC}"
            sudo zypper install -y postgresql-server postgresql
            ;;
        apk)
            echo -e "${BLUE}Running: sudo apk add postgresql postgresql-contrib postgresql-client${NC}"
            sudo apk add postgresql postgresql-contrib postgresql-client
            ;;
        brew)
            echo -e "${BLUE}Running: brew install postgresql@16${NC}"
            brew install postgresql@16
            ;;
        *)
            echo -e "${RED}[ERROR] No supported package manager found.${NC}"
            return 1
            ;;
    esac

    if [ $? -eq 0 ]; then
        echo -e "${GREEN}✓ PostgreSQL installed successfully!${NC}"
        # Prompt to initialize cluster
        read -rp "Initialize database cluster (initdb) now? (Y/n): " do_init
        if [[ ! "$do_init" =~ ^[nN](o)?$ ]]; then
            init_postgres_cluster
        fi
        # Prompt to install pgAdmin 4
        read -rp "Would you also like to install / setup pgAdmin 4? (Y/n): " do_pgadmin
        if [[ ! "$do_pgadmin" =~ ^[nN](o)?$ ]]; then
            install_pgadmin
        fi
    else
        echo -e "${RED}✗ Package installation failed.${NC}"
        return 1
    fi
}

# ------------------------------------------------------------------------------
# 13. Enable / Disable Autostart on Boot
# ------------------------------------------------------------------------------
enable_postgres() {
    echo -e "${CYAN}==> Enabling PostgreSQL service on boot...${NC}"
    if command -v systemctl &> /dev/null; then
        if sudo systemctl enable postgresql; then
            echo -e "${GREEN}✓ PostgreSQL service enabled to start on system boot.${NC}"
        else
            echo -e "${RED}✗ Failed to enable PostgreSQL service.${NC}"
        fi
    elif command -v rc-update &> /dev/null; then
        sudo rc-update add postgresql default
        echo -e "${GREEN}✓ PostgreSQL enabled on boot via OpenRC.${NC}"
    else
        echo -e "${YELLOW}Boot autostart manager not available.${NC}"
    fi
}

disable_postgres() {
    echo -e "${CYAN}==> Disabling PostgreSQL service on boot...${NC}"
    if command -v systemctl &> /dev/null; then
        if sudo systemctl disable postgresql; then
            echo -e "${GREEN}✓ PostgreSQL service disabled from boot.${NC}"
        else
            echo -e "${RED}✗ Failed to disable PostgreSQL service.${NC}"
        fi
    elif command -v rc-update &> /dev/null; then
        sudo rc-update del postgresql default
        echo -e "${GREEN}✓ PostgreSQL disabled from boot via OpenRC.${NC}"
    else
        echo -e "${YELLOW}Boot autostart manager not available.${NC}"
    fi
}

# ------------------------------------------------------------------------------
# 14. Self-Installation Helper
# ------------------------------------------------------------------------------
install_global() {
    mkdir -p "$INSTALL_DIR"
    local target_bin="${INSTALL_DIR}/${SCRIPT_NAME}"
    local script_source
    script_source="$(realpath "$0")"

    echo -e "${CYAN}==> Installing '${SCRIPT_NAME}' globally to '${INSTALL_DIR}'...${NC}"

    cp "$script_source" "$target_bin"
    chmod +x "$target_bin"

    # Also create alias/symlink for 'mypostgres' (single 's' common typo)
    ln -sf "$target_bin" "${INSTALL_DIR}/mypostgres"

    echo -e "${GREEN}✓ Installed successfully!${NC}"
    echo -e "You can now run ${BOLD}${CYAN}mypostgress${NC} or ${BOLD}${CYAN}mypostgres${NC} from any terminal."
}

uninstall_global() {
    echo -e "${CYAN}==> Uninstalling '${SCRIPT_NAME}' from '${INSTALL_DIR}'...${NC}"
    rm -f "${INSTALL_DIR}/${SCRIPT_NAME}" "${INSTALL_DIR}/mypostgres"
    echo -e "${GREEN}✓ Uninstalled successfully.${NC}"
}

# ------------------------------------------------------------------------------
# 15. Interactive Menu
# ------------------------------------------------------------------------------
show_menu() {
    detect_distro
    echo ""
    echo -e "${BOLD}${BLUE}════════════════════════════════════════════════════════════════${NC}"
    echo -e "${BOLD}${CYAN}             mypostgress Universal Management Tool              ${NC}"
    echo -e "${BOLD}${BLUE}════════════════════════════════════════════════════════════════${NC}"
    echo -e "  ${BOLD} 1.${NC} Start PostgreSQL"
    echo -e "  ${BOLD} 2.${NC} Stop PostgreSQL"
    echo -e "  ${BOLD} 3.${NC} Restart PostgreSQL"
    echo -e "  ${BOLD} 4.${NC} Check Service Status"
    echo -e "  ${BOLD} 5.${NC} Show Connection Strings (URI, psql, Node, Python, JDBC...)"
    echo -e "  ${BOLD} 6.${NC} Open / Connect in pgAdmin 4"
    echo -e "  ${BOLD} 7.${NC} List All Databases"
    echo -e "  ${BOLD} 8.${NC} Backup a Database"
    echo -e "  ${BOLD} 9.${NC} Restore a Database"
    echo -e "  ${BOLD}10.${NC} Initialize Database Cluster (initdb)"
    echo -e "  ${BOLD}11.${NC} Setup Superuser Role for '${USER}'"
    echo -e "  ${BOLD}12.${NC} Install / Reinstall PostgreSQL (Multi-Distro)"
    echo -e "  ${BOLD}13.${NC} Install / Setup pgAdmin 4"
    echo -e "  ${BOLD}14.${NC} Enable PostgreSQL on Boot"
    echo -e "  ${BOLD}15.${NC} Disable PostgreSQL on Boot"
    echo -e "  ${BOLD}16.${NC} Install / Update Global Command"
    echo -e "  ${BOLD} 0.${NC} Exit"
    echo -e "${BOLD}${BLUE}════════════════════════════════════════════════════════════════${NC}"
    read -rp "Select an option [0-16]: " choice
    echo ""

    case "$choice" in
        1) start_postgres ;;
        2) stop_postgres ;;
        3) restart_postgres ;;
        4) status_postgres ;;
        5) show_connection_strings ;;
        6) open_pgadmin ;;
        7) list_databases ;;
        8) backup_database ;;
        9) restore_database ;;
        10) init_postgres_cluster ;;
        11) setup_user_role ;;
        12) install_postgresql_packages ;;
        13) install_pgadmin ;;
        14) enable_postgres ;;
        15) disable_postgres ;;
        16) install_global ;;
        0) echo -e "${GREEN}Goodbye!${NC}"; exit 0 ;;
        *) echo -e "${RED}Invalid option: '$choice'. Please choose 0 to 16.${NC}" ;;
    esac
}

# ------------------------------------------------------------------------------
# 16. Main Entry Point
# ------------------------------------------------------------------------------
main() {
    # 1. Check CLI arguments
    if [ $# -gt 0 ]; then
        case "$1" in
            start)
                check_postgres_installed
                start_postgres
                ;;
            stop)
                check_postgres_installed
                stop_postgres
                ;;
            restart)
                check_postgres_installed
                restart_postgres
                ;;
            status)
                check_postgres_installed
                status_postgres
                ;;
            conn|connection-string|url)
                show_connection_strings "${2:-}" "${3:-}" "${4:-}"
                ;;
            pgadmin|open-pgadmin)
                open_pgadmin
                ;;
            install-pgadmin)
                install_pgadmin
                ;;
            install-postgres|install-pg)
                install_postgresql_packages
                ;;
            list)
                check_postgres_installed
                list_databases
                ;;
            backup)
                check_postgres_installed
                backup_database "${2:-}" "${3:-}"
                ;;
            restore)
                check_postgres_installed
                restore_database "${2:-}" "${3:-}"
                ;;
            init|initdb)
                check_postgres_installed
                init_postgres_cluster "${2:-}"
                ;;
            setup-user|init-user|create-user)
                check_postgres_installed
                setup_user_role "${2:-$USER}"
                ;;
            enable)
                enable_postgres
                ;;
            disable)
                disable_postgres
                ;;
            install)
                install_global
                ;;
            uninstall)
                uninstall_global
                ;;
            version|-v|--version)
                detect_distro
                echo "mypostgress version ${VERSION} (${DISTRO_NAME})"
                ;;
            help|--help|-h)
                detect_distro
                echo -e "${BOLD}Usage:${NC} ${CYAN}mypostgress${NC} [command] [arguments]"
                echo ""
                echo -e "${BOLD}Detected OS:${NC} ${DISTRO_NAME}"
                echo ""
                echo -e "${BOLD}Commands:${NC}"
                echo "  start                           Start PostgreSQL service (auto-detects uninitialized cluster)"
                echo "  stop                            Stop PostgreSQL service"
                echo "  restart                         Restart PostgreSQL service"
                echo "  status                          Check service status"
                echo "  conn [dbname] [user] [pass]     Show connection strings (URI, psql, Node, Python, JDBC...)"
                echo "  pgadmin                         Open / launch pgAdmin 4 (with server parameters)"
                echo "  list                            List all databases"
                echo "  backup [db_name] [file]         Backup database to a destination file"
                echo "  restore [db_name] [file]        Restore database from a backup file"
                echo "  init [dir]                      Initialize PostgreSQL database cluster (initdb)"
                echo "  setup-user [username]           Create superuser role for current Linux user"
                echo "  install-postgres                Install/reinstall PostgreSQL packages (Multi-distro)"
                echo "  install-pgadmin                 Install pgAdmin 4 (Flatpak / Native / Docker / Pip)"
                echo "  enable                          Enable PostgreSQL service on boot"
                echo "  disable                         Disable PostgreSQL service on boot"
                echo "  install                         Install/update 'mypostgress' to ~/.local/bin"
                echo "  uninstall                       Remove 'mypostgress' from ~/.local/bin"
                echo "  version                         Show version and OS info"
                echo "  help                            Show this help message"
                echo ""
                echo "Run without any arguments to open the interactive menu."
                ;;
            *)
                echo -e "${RED}Unknown command: $1${NC}"
                echo "Run 'mypostgress help' for available commands."
                exit 1
                ;;
        esac
    else
        # Prerequisite check before interactive loop
        check_postgres_installed

        # Interactive loop
        while true; do
            show_menu
        done
    fi
}

if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
