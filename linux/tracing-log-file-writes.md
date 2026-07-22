# Tracing Log File Writes

> **Company:** Bloomberg | **Difficulty:** Easy

---

#### **Scenario**

The `/var/log/messages` file has been growing unusually fast, filling up disk space within hours.

#### **Task**

Identify the process that is writing heavily to `/var/log/messages` by monitoring system activity in real time. Save the process details using `ps` and last `50` lines of logs at `/home/devops/excessive_log_process.txt`

#### **Example**

```
# Before (log file growing rapidly)

/var/log/messages: 15 GB and increasing
Disk usage: 92% and climbing
```

```
# After (responsible process identified)

Process identified: rsyslogd (PID 1234)
Confirmed active writes to /var/log/messages
```

---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/tracing-log-file-writes)

---
This is testing whether you know how to **identify which process is actively writing to a log file**, not just inspect the log contents.

### Step 1. Monitor writes to `/var/log/messages`

Use `iotop` (or `lsof`) to see which process is actively writing.

```bash
sudo iotop -o
```

The `-o` flag shows **only processes doing I/O**.

Alternatively, if the goal is specifically to see who has the file open:

```bash
sudo lsof /var/log/messages
```

Typical output:

```text
COMMAND    PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
rsyslogd  1234 root    4w   REG    ...            /var/log/messages
```

---

### Step 2. Save the process information

If the PID is `1234`:

```bash
ps -fp 1234 > /home/devops/excessive_log_process.txt
```

Example output:

```text
UID   PID  PPID CMD
root 1234     1 /usr/sbin/rsyslogd -n
```

---

### Step 3. Append the last 50 log lines

```bash
tail -50 /var/log/messages >> /home/devops/excessive_log_process.txt
```

---

## Final file

```bash
cat /home/devops/excessive_log_process.txt
```

Example:

```text
UID   PID  PPID CMD
root 1234     1 /usr/sbin/rsyslogd -n

Jul 22 10:41:01 ...
Jul 22 10:41:02 ...
...
(last 50 log entries)
```

---

## One-liner (after finding the PID)

```bash
ps -fp 1234 > /home/devops/excessive_log_process.txt && tail -50 /var/log/messages >> /home/devops/excessive_log_process.txt
```

---

## Interview explanation

> "Since `/var/log/messages` is growing rapidly, I'd first identify which process is actively writing to it using `lsof /var/log/messages` or monitor active disk writers with `iotop -o`. Once I identify the PID (commonly `rsyslogd`, though another process may be generating excessive logs), I'd capture its details with `ps -fp <PID>` and save the last 50 log entries using `tail -50 /var/log/messages` for further investigation. This helps distinguish the logging daemon from the application that is generating the excessive log messages."
