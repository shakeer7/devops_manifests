# Python for DevOps Interviews (4+ Years Experience)

## 1. Basic Syntax

**File Handling, OS/System Operations, and JSON**

```python
import os
import json

# 1. Basic OS operations
def ensure_directory(path):
    if not os.path.exists(path):
        os.makedirs(path)
        print(f"Created directory: {path}")

# 2. File handling and JSON
def update_config(filepath, key, new_value):
    ensure_directory(os.path.dirname(filepath))
    
    # Read existing or create empty dict
    data = {}
    if os.path.exists(filepath):
        with open(filepath, 'r') as f:
            data = json.load(f)
    
    # Update and write
    data[key] = new_value
    with open(filepath, 'w') as f:
        json.dump(data, f, indent=4)
        
update_config("/tmp/app/config.json", "environment", "production")
```

---

## 2. Intermediate Examples

**Subprocess, Regex, and Exception Handling**

```python
import subprocess
import re
import sys

def check_service_status(service_name):
    try:
        # Run a shell command
        result = subprocess.run(
            ['systemctl', 'status', service_name],
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True,
            check=True # Raises CalledProcessError if exit code != 0
        )
        
        # Parse output with Regex
        # Looking for something like: "Active: active (running)"
        match = re.search(r'Active:\s+(.*?)\s+\(', result.stdout)
        if match:
            status = match.group(1)
            print(f"{service_name} status: {status}")
            return status == "active"
            
    except subprocess.CalledProcessError as e:
        print(f"Error checking service: {e.stderr.strip()}", file=sys.stderr)
        return False
    except Exception as e:
        print(f"Unexpected error: {e}", file=sys.stderr)
        return False

# Usage
# is_running = check_service_status("docker")
```

---

## 3. Advanced Examples

**API Interaction (Requests) and AWS Automation (Boto3)**

```python
import boto3
import requests
from botocore.exceptions import ClientError

# 1. Cloud Automation: Stop untagged EC2 instances
def cleanup_untagged_instances(region="us-east-1"):
    ec2 = boto3.client('ec2', region_name=region)
    
    try:
        # Find running instances without 'Owner' tag
        instances = ec2.describe_instances(
            Filters=[
                {'Name': 'instance-state-name', 'Values': ['running']}
            ]
        )
        
        to_stop = []
        for reservation in instances['Reservations']:
            for instance in reservation['Instances']:
                tags = {t['Key']: t['Value'] for t in instance.get('Tags', [])}
                if 'Owner' not in tags:
                    to_stop.append(instance['InstanceId'])
        
        if to_stop:
            print(f"Stopping instances: {to_stop}")
            ec2.stop_instances(InstanceIds=to_stop)
            
    except ClientError as e:
        print(f"AWS Error: {e}")

# 2. API Interaction: Trigger Jenkins Build
def trigger_jenkins_job(jenkins_url, job_name, user, token):
    url = f"{jenkins_url}/job/{job_name}/build"
    response = requests.post(url, auth=(user, token))
    
    if response.status_code == 201:
        print("Build triggered successfully")
    else:
        print(f"Failed to trigger build: {response.status_code}")
```

---

## 4. Interview Coding Exercises

### Problem 1: `tail -f` Log Monitor
**Task:** Write a Python script that continuously reads a log file (like `tail -f`) and alerts if it finds the word "ERROR".

**Solution:**
```python
import time

def monitor_log(filepath):
    with open(filepath, 'r') as file:
        # Move pointer to the end of the file
        file.seek(0, 2)
        
        while True:
            line = file.readline()
            if not line:
                time.sleep(0.5) # Wait for new data
                continue
                
            if "ERROR" in line:
                print(f"ALERT FOUND: {line.strip()}")

# monitor_log("/var/log/syslog")
```
**Explanation:** `file.seek(0, 2)` goes to the EOF. The `while True` loop tries to read; if there's no new line, it sleeps briefly. If there is, it processes it.

### Problem 2: Disk Space Monitor
**Task:** Write a script that checks disk usage of `/` and prints a warning if it is over 80%.
**Solution:**
```python
import shutil

def check_disk_usage(threshold_percent=80.0):
    total, used, free = shutil.disk_usage("/")
    
    # Calculate percentage
    percent_used = (used / total) * 100
    
    if percent_used > threshold_percent:
        print(f"WARNING: Disk usage is high at {percent_used:.2f}%")
    else:
        print(f"Disk usage is healthy at {percent_used:.2f}%")
```

---

## 5. Troubleshooting Exercises

### Broken Configuration 1: Mutable Default Arguments
**What is wrong here?**
```python
def add_server(server_ip, server_list=[]):
    server_list.append(server_ip)
    return server_list

print(add_server("10.0.0.1"))
print(add_server("10.0.0.2"))
```
**Answer:** In Python, default arguments are evaluated only *once* when the function is defined. The `server_list` will persist across calls. The second print will output `["10.0.0.1", "10.0.0.2"]`, which is usually a bug.
**Corrected:**
```python
def add_server(server_ip, server_list=None):
    if server_list is None:
        server_list = []
    server_list.append(server_ip)
    return server_list
```

---

## 6. Common Interview Questions

**Q: "How do you parse arguments in a Python CLI script?"**
*Answer:* The standard library `argparse` is best. For simple scripts, `sys.argv` can be used. For advanced/modern CLIs, third-party libraries like `click` or `typer` are highly favored in the DevOps community.

**Q: "Explain generators and why they are useful in DevOps."**
*Answer:* Generators use the `yield` keyword to return data one item at a time instead of storing a whole list in memory. They are crucial for DevOps tasks like parsing massive log files (e.g., a 10GB access log) without crashing the server due to out-of-memory (OOM) errors.

**Q: "What's the difference between `os.system()` and `subprocess.run()`?"**
*Answer:* `os.system()` is older, less secure (vulnerable to shell injection), and doesn't let you easily capture `stdout`/`stderr`. `subprocess.run()` is the modern standard, offering tight control over input/output pipes, timeout handling, and exit code checking (`check=True`).

---

## 7. Cheat Sheet

| Snippet | Explanation |
| :--- | :--- |
| `os.environ.get('AWS_REGION', 'us-east-1')` | Safely get an environment variable with a fallback. |
| `json.loads(string_data)` | Convert a JSON string into a Python dictionary. |
| `subprocess.run([...], capture_output=True)` | Modern way to run commands and grab output. |
| `with open('file.txt') as f:` | Context manager; safely ensures the file is closed automatically. |
| `re.findall(r'\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}', text)` | Regex to find all IP addresses in a string. |
