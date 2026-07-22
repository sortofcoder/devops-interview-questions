# Managing High I/O Processes

> **Company:** Revolut | **Difficulty:** Easy

---

#### **Scenario**
Users are complaining about slow file access. System metrics show high disk utilization.

#### **Task**
Reduce `I/O` activity of top offender using I/O priorities to `idle`. Keep critical jobs (databases, message queues, applications) at high priority.


#### **Example**

```
 # Before (high I/O contention)

 Total DISK READ:      45.67 M/s | Total DISK WRITE:     123.45 M/s

   TID  PRIO  USER      DISK READ  DISK WRITE  COMMAND
  5678  be/4  backup     2.34 M/s   98.76 M/s  rsync /data /backup
  3421  be/4  postgres  15.23 M/s   12.45 M/s  postgres: vacuum
  8234  be/4  appuser    8.45 M/s    3.21 M/s  /usr/bin/log-processor

 # After (non-critical jobs throttled)

 Total DISK READ:      18.34 M/s | Total DISK WRITE:      35.21 M/s

   TID  PRIO  USER      DISK READ  DISK WRITE  COMMAND
  5678  idle  backup     0.50 M/s    2.10 M/s  rsync /data /backup
  3421  be/4  postgres  15.23 M/s   12.45 M/s  postgres: vacuum
  8234  idle  appuser    1.20 M/s    0.80 M/s  /usr/bin/log-processor

```

---
📹 [Video Solution](https://prepare.sh/interview/devops/terminal/managing-high-io-processes)

---
## GPT answer

---

# Step 1: What is happening?

Imagine one disk.

Only one thing can physically read/write from the disk at a time.

Think of it like one checkout lane in a supermarket.

People in line:

* PostgreSQL
* Redis
* rsync backup
* Log processor
* Application

Everyone wants to use the disk.

If one process writes massive amounts of data, everyone else waits.

That waiting causes:

* slow applications
* slow databases
* slow file access
* increased latency

So the problem is **not necessarily CPU**.

It's that **too many processes want the disk simultaneously.**

---

# Step 2: What does "high disk utilization" mean?

Suppose Linux reports

```
Disk Utilization: 100%
```

This does **NOT** mean

> The disk is full.

It means

> The disk is busy almost all the time.

Imagine a printer.

If someone is constantly printing,

the printer utilization is 100%.

The printer still has paper.

It's just never idle.

Exactly the same idea.

---

# Step 3: What is disk I/O?

I/O = Input / Output

Disk I/O means

Reading

```
Read file
Read database page
Read logs
```

Writing

```
Write logs
Backup files
Save database
Create temp files
```

Everything that touches storage is disk I/O.

---

# Step 4: Why are users complaining?

Imagine:

```
Database needs file

↓

Backup is copying 5TB

↓

Disk is busy

↓

Database waits

↓

Application waits

↓

Users complain
```

Nothing is broken.

The disk is just busy serving someone else.

---

# Step 5: What does the example show?

Before

```
backup (rsync)

98 MB/s write
```

It is absolutely hammering the disk.

Meanwhile

```
postgres
```

needs disk too.

So they compete.

---

# Step 6: What is I/O priority?

Linux lets you tell the scheduler

> "This process is important."

or

> "This process can wait."

Exactly like CPU priorities (`nice`),

except for disks.

CPU

```
nice
renice
```

Disk

```
ionice
```

---

# Step 7: Priority classes

Linux has three major classes.

## Real Time

```
RT
```

Gets disk first.

Almost never used.

Can starve everything else.

---

## Best Effort

```
be
```

Default.

Most processes use this.

Example

```
postgres

nginx

python

java

```

---

## Idle

```
idle
```

This means

> Only use the disk when nobody else needs it.

Perfect for

* backups
* indexing
* virus scans
* log processing
* batch jobs

---

# Step 8: What does the interviewer want?

They want you to identify

Which workload is critical?

Which isn't?

Example

Critical

```
postgres

mysql

redis

rabbitmq

web application
```

Non-critical

```
backup

find /

rsync

log processor

updatedb

large archive
```

Instead of stopping backups,

we simply tell Linux

> Let everyone else go first.

---

# Step 9: How?

Command

```
ionice
```

View priority

```
ionice -p 5678
```

Change process

```
sudo ionice -c3 -p 5678
```

Meaning

```
-c3

class 3

Idle
```

Now

```
backup

```

only gets disk when nobody important needs it.

---

# Step 10: Looking at the example

Before

```
backup

98 MB/s
```

Very aggressive.

After

```
backup

2 MB/s
```

Linux pauses it whenever

* postgres
* applications

need disk.

Users become happy again.

The backup still finishes.

Just later.

---

# Step 11: How would you investigate?

This is where interviewers really care.

Don't immediately type commands.

Think like an SRE.

### 1. Confirm symptoms

Users report

```
Slow file access
```

---

### 2. Check disk utilisation

```
iostat -x 1
```

Look for

```
%util

await

svctm

avgqu-sz
```

If

```
%util

≈100%
```

or

```
await

very high
```

the disk is saturated.

---

### 3. Find the offending process

Use

```
iotop
```

Example

```
PID

5678

rsync

100 MB/s write
```

Immediately obvious.

---

### 4. Decide whether it's important

Ask

Is this

```
database?
```

No.

```
message queue?
```

No.

```
application?
```

No.

It's a backup.

Perfect candidate.

---

### 5. Lower priority

```
sudo ionice -c3 -p 5678
```

---

### 6. Observe

Run

```
iotop
```

again.

Verify

```
database

same speed

backup

much slower
```

Also watch

```
iostat
```

Users should notice lower latency because critical workloads are no longer waiting behind the backup.

---

# Easy way to remember `ionice`

Think of CPU scheduling:

```
nice

"I'm nice."

"I'll let others use CPU first."
```

Disk scheduling:

```
ionice

"I'm nice."

"I'll let others use the disk first."
```

---

# Interview Answer (Mid-Level SRE)

> "First I'd confirm that disk I/O is the bottleneck rather than CPU or memory by checking metrics with `iostat`, looking for high `%util` and increased `await` times. I'd then use `iotop` to identify which processes are generating the most disk reads and writes. If the top consumer is a non-critical workload like a backup, log processor, or indexing job, I'd lower its I/O priority using `ionice -c3 -p <PID>`, which moves it to the idle class so it only uses the disk when higher-priority processes don't need it. I would avoid throttling critical services such as PostgreSQL, MySQL, Redis, RabbitMQ, or the application itself. After making the change, I'd monitor disk utilization and application latency to verify that users' file access performance has improved while ensuring the background job continues to make progress."

---

## The mental model to remember

Whenever you see a question like this, think of this flow:

```
Users are slow
        │
        ▼
Is disk busy?
        │
        ▼
Find top I/O process (iotop)
        │
        ▼
Critical service?
   │              │
  Yes            No
   │              │
Leave it      Lower I/O priority (ionice)
   │              │
        ▼
Monitor disk latency and user experience
```

One additional interview insight: **`ionice` changes scheduling priority, not a hard bandwidth limit.** If the disk is otherwise idle, an `idle`-class process can still use nearly all of the available bandwidth. As soon as a higher-priority process needs disk I/O, the idle process backs off. That's why it's ideal for backups and batch jobs—you don't waste disk capacity when nothing else is using it, but you protect latency-sensitive services when contention occurs.

