# **Task 3 – Nightly Report Failure Investigation**

**Server:** `office-srv`\
**Operating system:** Ubuntu 24.04.1 LTS\
**Investigation date:** 16 November 2026

## **1. Investigation transcript**

The investigation starts with the README to establish what evidence is available. I then follow the evidence from the failed services towards the actual execution failure, checking alternative explanations before deciding on the root cause.

| #  | Evidence file (command)                                                                             | What I expected                                                                        | What I found                                                                                                                                                         | What I concluded                                                                                                                                                                |
| -- | --------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1  | `00-README.txt`                                                                                     | A list of collected evidence and commands.                                             | The pack contains 22 command outputs covering system status, timers, logs, script contents, permissions and command history.                                         | I can investigate the problem using the supplied evidence and identify missing commands to run later.                                                                           |
| 2  | `05-systemctl-failed.txt` (`systemctl --failed`)                                                    | The nightly report service should appear as failed if it is not running successfully.  | Two failed units: `legacy-sync.service` and `nightly-report.service`.                                                                                                | Two leads exist. I need to establish which one is related to the ticket.                                                                                                        |
| 3  | `01-hostnamectl.txt` (`hostnamectl`)                                                                | Confirm the server identity and operating system.                                      | Hostname is `office-srv`; operating system is Ubuntu 24.04.1 LTS.                                                                                                    | This is the correct server, so I can continue investigating.                                                                                                                    |
| 4  | `02-uptime.txt` (`uptime`)                                                                          | Check whether the server has recently rebooted or is under unusual load.               | The server has been running for almost seven days, with very low load averages.                                                                                      | No indication of a system overload or recent reboot explaining the missing report.                                                                                              |
| 5  | `03-df-h.txt` (`df -h`)                                                                             | Check whether the disk is full and preventing log writes.                              | Root filesystem is 48% used, with 5.8 GB available.                                                                                                                  | Disk space is not the cause.                                                                                                                                                    |
| 6  | `04-free-h.txt` (`free -h`)                                                                         | Check whether memory exhaustion could prevent the script from running.                 | 1.5 GiB memory available and no swap usage.                                                                                                                          | Memory pressure is not the cause.                                                                                                                                               |
| 7  | `06-systemctl-list-timers.txt` (`systemctl list-timers`)                                            | Confirm whether the nightly report timer is running every two minutes.                 | `nightly-report.timer` is listed, activates `nightly-report.service`, and has a two-minute interval.                                                                 | The scheduler is running. The problem is more likely with service execution than scheduling.                                                                                    |
| 8  | `07-systemctl-status-nightly-report.txt` (`systemctl status nightly-report.service`)                | Identify why the service fails when triggered.                                         | Service has `Result: exit-code`, status `203/EXEC`, and reports that `/usr/local/bin/nightly-report.sh` cannot be executed because no such file or directory exists. | The failure occurs when systemd tries to execute the script. I need to inspect its configuration and file.                                                                      |
| 9  | `08-systemctl-cat-nightly-report.txt` (`systemctl cat nightly-report.service nightly-report.timer`) | Check the service executable path and timer configuration.                             | `ExecStart=/usr/local/bin/nightly-report.sh`. Timer uses `OnBootSec=1min` and `OnUnitActiveSec=2min`.                                                                | The configured script path needs to be checked. The timer configuration itself looks correct.                                                                                   |
| 10 | `09-journalctl-nightly-report.txt` (`journalctl -u nightly-report.service`)                         | Find when the failures started and whether the service previously worked.              | Successful runs occurred at 14:12, 14:14, 14:16 and 14:18 on 9 November. Failures began at 14:20, with repeated `203/EXEC` errors afterwards.                        | The fault began during the same period as the recent changes. This suggests a change to the executable or its environment.                                                      |
| 11 | `10-systemctl-status-legacy-sync.txt` (`systemctl status legacy-sync.service`)                      | Determine whether the other failed service affects the nightly report.                 | `legacy-sync.service` executes `/bin/false` and is described as an old, no-longer-used service.                                                                      | This is an unrelated failure and not the cause of the nightly report problem.                                                                                                   |
| 12 | `11-journalctl-legacy-sync.txt` (`journalctl -u legacy-sync.service`)                               | Check the history of the legacy service.                                               | It has failed repeatedly since September, across multiple boots.                                                                                                     | This is an existing unrelated failure, not a new problem caused by the nightly report update.                                                                                   |
| 13 | `12-journalctl-p-err-b.txt` (`journalctl -p err -b`)                                                | Look for other system errors around the time of failure.                               | Shows the legacy-sync failure and repeated nightly-report execution errors. There are also unrelated kernel messages.                                                | The relevant error remains the nightly-report `203/EXEC` failure.                                                                                                               |
| 14 | `13-ls-l-usr-local-bin.txt` (`ls -l /usr/local/bin`)                                                | Check whether the script exists and has execute permission.                            | `nightly-report.sh` exists and has permissions `-rwxr-xr-x`.                                                                                                         | The file exists and is executable. A simple missing-file or permissions problem is unlikely.                                                                                    |
| 15 | `14-ls-l-var-log.txt` (`ls -l /var/log`)                                                            | Check whether the report log exists and whether its permissions might prevent writing. | `nightly-report.log` exists, owned by root, with permissions `-rw-r--r--`.                                                                                           | The log exists and is writable by root. The script's failure to execute is a more immediate problem.                                                                            |
| 16 | `15-tail-nightly-report-log.txt` (`tail -n 20 /var/log/nightly-report.log`)                         | Check the latest successful report timestamp.                                          | The last report was written at 14:18 on 9 November.                                                                                                                  | This confirms that report generation stopped at the same time the journal records the execution failures.                                                                       |
| 17 | `16-cat-nightly-report-sh.txt` (`cat /usr/local/bin/nightly-report.sh`)                             | Inspect the script's contents for errors.                                              | The script contains `#!/bin/bash`, followed by commands to record the date, disk usage and memory usage in the log.                                                  | The script's logic looks correct. I need to inspect the actual characters in the file, not just its displayed contents.                                                         |
| 18 | `17-cat-A-nightly-report-sh.txt` (`cat -A /usr/local/bin/nightly-report.sh`)                        | Check for hidden characters, particularly in the shebang.                              | Every line ends with `^M$`, indicating Windows-style CRLF line endings.                                                                                              | This is a strong indication that the script has Windows-style line endings, potentially affecting execution.                                                                    |
| 19 | `18-head-1-od-c.txt` (`head -1 /usr/local/bin/nightly-report.sh \\| od -c`)                         | Verify the exact characters in the interpreter declaration.                            | The first line ends in `\r\n`, rather than the expected Unix `\n`.                                                                                                   | The shebang contains a carriage-return character. Linux may therefore look for an interpreter named `/bin/bash` followed by a carriage return.                                  |
| 20 | `19-run-by-hand.txt` (running the script manually)                                                  | Check whether the script can execute outside systemd.                                  | Both normal execution and execution with `sudo` fail with a cannot-execute/file-not-found error.                                                                     | The problem is not specific to systemd or its timer. The script itself cannot execute correctly.                                                                                |
| 21 | `20-ls-l-bin-bash.txt` (`which bash; ls -l /bin/bash`)                                              | Check whether Bash is missing or unavailable.                                          | Bash exists at `/bin/bash` and is executable.                                                                                                                        | The interpreter is installed. This supports the hidden carriage-return explanation rather than a missing Bash installation.                                                     |
| 22 | `21-apt-history.txt` (`tail -n 12 /var/log/apt/history.log`)                                        | Check whether system updates coincide with the beginning of the problem.               | An `apt full-upgrade` occurred on 9 November between 09:38 and 09:41.                                                                                                | This is useful timeline information, but the evidence does not prove that the upgrade caused the script's line-ending problem.                                                  |
| 23 | `22-bash-history-aigerim.txt` (`tail -n 15 ~/.bash_history`)                                        | Check what changes were made to the script recently.                                   | The history shows the script was copied from the user's home directory into `/usr/local/bin`, made executable and manually started.                                  | This provides a plausible route by which the script was introduced with incorrect line endings. It supports investigating the copied script rather than changing the scheduler. |

### **Additional investigation commands not included in the evidence pack**

These would be useful before making changes:

| #  | Command I would run                         | What I expect                                                       | Why I would run it                                                   |
| -- | ------------------------------------------- | ------------------------------------------------------------------- | -------------------------------------------------------------------- |
| 24 | `file /usr/local/bin/nightly-report.sh`     | The file is identified as a Bash script with CRLF line terminators. | Independently confirm the line-ending problem.                       |
| 25 | `systemctl is-enabled nightly-report.timer` | Ideally, `enabled`.                                                 | Establish whether the timer will start automatically after a reboot. |
| 26 | `systemctl is-active nightly-report.timer`  | `active`.                                                           | Confirm the timer is currently running.                              |
| 27 | `ls -l /var/log/nightly-report.log`         | Confirm the log file's current timestamp and size.                  | Establish a baseline before repairing the script.                    |

## **2. Root cause**

The root cause is incorrect Windows-style CRLF line endings in `/usr/local/bin/nightly-report.sh`. The carriage-return character at the end of the shebang (`#!/bin/bash`) causes Linux to look for an interpreter with an invalid filename, resulting in `203/EXEC`.

This is proven by `17-cat-A-nightly-report-sh.txt` and `18-head-1-od-c.txt`, which show the `^M` and `\r\n` characters. Evidence from `07-systemctl-status-nightly-report.txt` and `19-run-by-hand.txt` confirms the execution failure. Bash itself exists, as shown in `20-ls-l-bin-bash.txt`.

The timer is working correctly. The fault is the script's line endings, not the scheduler.

## **3. Fix**

I would repair the existing script rather than recreate it, because its contents are correct.

Run these commands on `office-srv` in order:

**Step 1 – Make a backup**

```
sudo cp -a /usr/local/bin/nightly-report.sh \
/usr/local/bin/nightly-report.sh.bak
```

**Step 2 – Convert Windows line endings to Unix line endings**

```
sudo sed -i 's/\r$//' /usr/local/bin/nightly-report.sh
```

**Step 3 – Confirm the correction**

```
file /usr/local/bin/nightly-report.sh
```

Expected: the file is identified as a Bash script with Unix-style line endings, not CRLF.

Check the first line:

```
head -1 /usr/local/bin/nightly-report.sh | od -c
```

Expected output should end with:

```
/ b a s h \n
```

There should be no `\r` character.

**Step 4 – Confirm executable permissions**

```
sudo chmod 755 /usr/local/bin/nightly-report.sh
ls -l /usr/local/bin/nightly-report.sh
```

Expected permissions:

```
-rwxr-xr-x
```

**Step 5 – Clear the previous failed state and run the service**

```
sudo systemctl reset-failed nightly-report.service
sudo systemctl start nightly-report.service
```

**Step 6 – Ensure the timer starts automatically after reboot**

```
sudo systemctl enable --now nightly-report.timer
```

This ensures the timer is enabled at boot and running immediately.

## **4. Verification**

I would verify the repair in three stages: successful manual execution, successful scheduled execution, and persistence after reboot.

### **A. Verify the service runs successfully**

Run:

```
sudo systemctl start nightly-report.service
systemctl status nightly-report.service
```

Expected:

- No `203/EXEC` error.
- The service completes successfully.
- The result is `success` (a oneshot service may subsequently show `inactive (dead)`, which is normal).

Check the service result directly:

```
systemctl show nightly-report.service \
-p Result -p ExecMainStatus
```

Expected:

```
Result=success
ExecMainStatus=0
```

Check whether the report has been written:

```
tail -n 10 /var/log/nightly-report.log
```

Expected:

- A new date and time.
- Disk usage information.
- Memory usage information.

The new timestamp must be later than the previous last entry, 9 November at 14:18.

### **B. Verify automatic scheduling**

Run:

```
systemctl is-active nightly-report.timer
systemctl is-enabled nightly-report.timer
systemctl list-timers
```

Expected:

```
active
enabled
```

The timer should appear in the timer list, activating `nightly-report.service`, with its next run approximately two minutes after the previous activation.

Wait at least two minutes and check:

```
sudo journalctl -u nightly-report.service -n 20 --no-pager
tail -n 20 /var/log/nightly-report.log
```

Expected:

- New service executions approximately every two minutes.
- Successful completion messages.
- New report entries with updated timestamps.
- No repeated `203/EXEC` errors.

### **C. Prove the fix survives a reboot**

A successful manual run is not enough. The timer must also start automatically when the server restarts.

First, reboot:

```
sudo reboot
```

Reconnect to `office-srv` using SSH.

Check that the timer started:

```
systemctl is-active nightly-report.timer
systemctl is-enabled nightly-report.timer
```

Expected:

```
active
enabled
```

Check the timer:

```
systemctl list-timers
```

Expected: `nightly-report.timer` is listed and has a next scheduled execution.

Because the timer uses `OnBootSec=1min`, allow at least one minute after boot for the first execution.

Check the service journal for this boot:

```
sudo journalctl -b -u nightly-report.service --no-pager
```

Expected:

- A new service execution after reboot.
- Successful completion.
- No `203/EXEC` errors.

Finally, check the report file:

```
ls -l /var/log/nightly-report.log
tail -n 20 /var/log/nightly-report.log
```

Expected:

- The log file has a newer modification timestamp.
- New report entries appear after the reboot.
- Date, disk usage and memory information are present.

## **5. Final conclusion**

The investigation identified a script formatting problem rather than a scheduling problem.

The `nightly-report.timer` was running every two minutes, but the service could not execute its script because the shebang contained a hidden carriage-return character from Windows-style line endings.

Converting the script to Unix line endings using `sed`, preserving executable permissions and ensuring the timer is enabled fixes the problem.

The repair is considered successful only when:

1. The service runs successfully without execution errors.
2. The report log receives new entries every two minutes.
3. The timer is enabled and active.
4. The service automatically runs again after a reboot.
5. The report continues to be written after the reboot.

**The most important proof is that the report continues updating automatically after a reboot, without anyone manually starting the service.**
