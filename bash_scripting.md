# Bash/Shell Scripting for DevOps Interviews (4+ Years Experience)

## 1. Basic Syntax

**Variables, Conditions, Loops, and Arguments**

```bash
#!/bin/bash
# Enable strict mode: 
# -e (exit on error), -u (exit on unset variables), -o pipefail (catch errors in pipes)
set -euo pipefail

# Variables (No spaces around '=')
ENV_NAME="Production"
LOG_DIR="/var/log/myapp"

# Positional Arguments
if [ "$#" -ne 1 ]; then
    echo "Usage: $0 <service_name>"
    exit 1
fi
SERVICE=$1

# Conditionals
if [ "$ENV_NAME" == "Production" ]; then
    echo "Running in production mode..."
elif [ "$ENV_NAME" == "Staging" ]; then
    echo "Running in staging..."
else
    echo "Unknown environment."
fi

# Loops
for file in "$LOG_DIR"/*.log; do
    # Check if file exists to prevent wildcard literal issues if directory is empty
    if [ -f "$file" ]; then
        echo "Processing $file..."
    fi
done
```

---

## 2. Intermediate Examples

**Functions, Arrays, and Log Parsing (`grep`, `awk`)**

```bash
#!/bin/bash
set -euo pipefail

# Array definition
SERVICES=("nginx" "docker" "ssh")

# Function definition
check_status() {
    local svc_name=$1 # 'local' scopes the variable to the function
    
    if systemctl is-active --quiet "$svc_name"; then
        echo "[OK] $svc_name is running."
    else
        echo "[ERROR] $svc_name is down!"
    fi
}

# Iterate over array
for svc in "${SERVICES[@]}"; do
    check_status "$svc"
done

# Parsing a log file using awk and grep
# Extract all IP addresses that hit a 404 in Nginx logs
echo "Top 404 IP addresses:"
grep " 404 " /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -nr | head -n 5
```

---

## 3. Advanced Examples

**Process Management, Background Jobs, and `xargs`**

```bash
#!/bin/bash

# Run a process in the background
echo "Starting database backup..."
tar -czf /backup/db-$(date +%F).tar.gz /var/lib/mysql &
BACKUP_PID=$!

echo "Backup is running with PID: $BACKUP_PID"

# Wait for the specific background process to finish
wait $BACKUP_PID
if [ $? -eq 0 ]; then
    echo "Backup completed successfully."
else
    echo "Backup failed!"
    exit 1
fi

# Advanced `xargs` usage:
# Find all .log files older than 30 days and delete them concurrently using 4 processes
find /var/log/app -name "*.log" -mtime +30 -print0 | xargs -0 -I {} -P 4 rm {}
```

---

## 4. Interview Coding Exercises

### Problem 1: Health Check Script
**Task:** Write a script to ping a list of servers from a file (`servers.txt`). If a server doesn't respond, append it to `failed.txt`.

**Solution:**
```bash
#!/bin/bash

FILE="servers.txt"
OUT_FILE="failed.txt"

# Clear out file
> "$OUT_FILE"

# Read file line by line safely
while IFS= read -r server; do
    # Skip empty lines
    [ -z "$server" ] && continue
    
    # ping -c 1 (1 packet), -W 2 (timeout 2 seconds)
    if ! ping -c 1 -W 2 "$server" &> /dev/null; then
        echo "$server" >> "$OUT_FILE"
    fi
done < "$FILE"
```
**Explanation:** Using `while IFS= read -r line` is the safest way to read a file line-by-line in Bash without word splitting or backslash interpretation issues.

### Problem 2: Delete Old Files
**Task:** Write a one-liner to find and delete `.tmp` files in `/data` that are older than 7 days.

**Solution:**
```bash
find /data -type f -name "*.tmp" -mtime +7 -exec rm {} \;
# Alternatively (more efficient):
find /data -type f -name "*.tmp" -mtime +7 -delete
```

---

## 5. Troubleshooting Exercises

### Broken Configuration 1: Quoting Issue
**What is wrong here?**
```bash
DIR="My Documents"
cd $DIR
```
**Answer:** Because `$DIR` is not quoted, bash performs word splitting. It will attempt to execute `cd My` and look for an argument `Documents`, resulting in an error like `cd: too many arguments`.
**Corrected:**
```bash
cd "$DIR"
```

### Broken Configuration 2: Pipe Error Hiding
**Scenario:** A deployment script has this command:
`curl -s https://bad-url.com/install.sh | bash`
If the curl fails (e.g., DNS error), the script continues with exit code 0 because `bash` executed successfully (even though it received no input).
**Answer/Fix:** Add `set -o pipefail` at the top of the script. This ensures the exit code of a pipeline is the rightmost non-zero exit code.

---

## 6. Common Interview Questions

**Q: "What does `2>&1` mean?"**
*Answer:* It redirects file descriptor 2 (Standard Error - stderr) to file descriptor 1 (Standard Output - stdout). This allows you to capture both normal output and error messages into the same file or pipe. (e.g., `command > log.txt 2>&1`).

**Q: "How do you check if a port (e.g., 8080) is listening?"**
*Answer:* 
- Modern: `ss -tuln | grep 8080`
- Classic: `netstat -tuln | grep 8080`
- Port check from external: `nc -zv localhost 8080`

**Q: "What's the difference between `$@` and `$*`?"**
*Answer:* Both represent all command-line arguments. However, when wrapped in double quotes:
- `"$@"` expands to separate strings (`"$1" "$2" "$3"`). This is almost always what you want.
- `"$*"` expands to a single string (`"$1 $2 $3"`), separated by the first character of the `IFS` variable.

---

## 7. Cheat Sheet

| Command | Usage (4+ Yrs Experience Focus) |
| :--- | :--- |
| `awk '{print $3}'` | Print the 3rd column of text (space delimited by default). |
| `sed 's/old/new/g'` | Replace all occurrences of 'old' with 'new' in a stream. |
| `cut -d: -f1 /etc/passwd`| Print the 1st field of a file delimited by colons (gets all usernames). |
| `chmod +x script.sh` | Make a script executable. |
| `if [ -d "/dir" ]; then` | Check if a directory exists. |
| `if [ -z "$VAR" ]; then` | Check if a variable is empty (length is zero). |
| `${VAR:-default}` | Use "default" if `$VAR` is unset or null. |
