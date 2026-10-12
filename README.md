# CIP-B101 Lab 2A – Linux Command-Line Fundamentals: Case Workspace Setup, File Operations, Networking Basics and Evidence Packaging

**Student:** Fuseini Imoru Kantuogaa  
**Course:** CIP-B101  
**Lab:** Lab 2A – Directory Structure, File/Text Operations, grep Pattern Matching, Basic Networking Commands, Shell Scripting and Archive Creation

## Overview

This lab demonstrates the core Linux command-line skills used to set up and manage a forensic case workspace: creating a structured case directory, identifying the host operating system and environment, generating and inspecting evidence files (including file metadata via `stat`), copying/moving/renaming files and verifying results, writing and searching a multi-line activity log with `grep` (including extended regex and case-insensitive matching), using `find` to locate files by name, type and extension, checking basic network configuration and connectivity (`ip addr`, `ip route`, `ping`, `ss`, `nmap`), writing and running a reusable Bash shell script that generates a system summary report, adding that script to the shell `PATH` for use from any directory, and finally packaging the entire case workspace into both `.zip` and `.tar.gz` archives for evidence transfer. The goal was to build fluency with the everyday Linux toolkit a forensic analyst uses to build and manage a case folder, well before any specialized forensic tooling is introduced.

## Case Folder Structure

```
mkdir -p CIP-B101_Lab2A_Fuseini_Imoru
cd CIP-B101_Lab2A_Fuseini_Imoru
pwd
ls -la
```

| Check | Result |
|---|---|
| Working directory | /home/kali/CIP-B101_Lab2A_Fuseini_Imoru |
| Initial contents | empty (total 8, just `.` and `..`) |

Final structure (confirmed with `tree` in section 5 and again before archiving):

```
CIP-B101_Lab2A_Fuseini_Imoru/
├── evidence/
│   ├── activity_log.txt
│   ├── case_note1_renamed.txt
│   ├── lab2a_evidence.txt
│   └── notes_backup/
│       ├── case_note1.txt
│       └── case_note2.txt
├── notes/
│   ├── case_note1.txt
│   └── case_note2.txt
├── outputs/
│   └── system_summary.txt
└── scripts/
    └── lab2a_sys_summary.sh
```

## 1. Host and Root Filesystem Identification

```
ls /
ls -la /
```

The root filesystem listing confirmed a standard Kali Linux layout: `bin`, `boot`, `dev`, `etc`, `home`, `initrd.img`/`initrd.img.old` (symlinks to the installed kernel initramfs, 6.18.12+kali-amd64), `lib`/`lib32`/`lib64` (symlinks to their `usr/` counterparts), `lost+found`, `media`, `mnt`, `opt`, `proc`, `root`, `run`, `sbin` (symlink to `usr/sbin`), `srv`, `swap` (a 1,000,000,000-byte swapfile), `sys`, `tmp`, `usr`, `var`, and `vmlinuz`/`vmlinuz.old` (kernel image symlinks).

```
cat /etc/os-release
uname -a
whoami
id
```

| Field | Value |
|---|---|
| PRETTY_NAME | Kali GNU/Linux Rolling |
| NAME | Kali GNU/Linux |
| VERSION_ID | 2026.1 |
| VERSION | 2026.1 |
| VERSION_CODENAME | kali-rolling |
| ID / ID_LIKE | kali / debian |
| uname -a | Linux kali 6.18.12+kali-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.18.12-1kali1 (2026-02-25) x86_64 GNU/Linux |
| whoami | kali |
| id | uid=1000(kali) gid=1000(kali) groups=1000(kali),4(adm),20(dialout),24(cdrom),25(floppy),27(sudo),29(audio),30(dip),44(video),46(plugdev),100(users),101(netdev),102(scanner),104(bluetooth),113(lpadmin),122(wireshark),123(kaboxer) |

The `kali` user's group membership confirms it belongs to `sudo` (administrative privileges), `wireshark` (packet-capture access) and `adm`/`disk`-adjacent groups (`dialout`, `cdrom`, `floppy`) typical of a fully provisioned Kali desktop install.

## 2. Creating and Inspecting the First Evidence File

```
mkdir -p evidence
echo "This is Lab 2A evidence file created by Fuseini Imoru" > evidence/lab2a_evidence.txt
ls -li evidence/lab2a_evidence.txt
stat evidence/lab2a_evidence.txt
```

| Field | Value |
|---|---|
| Inode | 524452 |
| Permissions | -rw-rw-r-- |
| Size | 54 bytes |
| Owner/Group | kali/kali |
| Device | 8,1 |
| Links | 1 |
| Access / Modify / Change / Birth | 2026-06-27 07:05:30.152271715 -0400 (all four timestamps identical, confirming this was a freshly created, never-modified file) |

The `stat` command's four timestamps (Access, Modify, Change, Birth) all matching exactly is itself a useful forensic baseline: on a file that has since been read, edited, or had permissions changed, these would diverge — recording them here establishes a clean "time zero" for this evidence file.

## 3. Disk Space and Inode Usage

```
df -h
df -i
```

| Filesystem | Size | Used | Avail | Use% | Mounted on |
|---|---|---|---|---|---|
| udev | 879M | 0 | 879M | 0% | /dev |
| tmpfs | 197M | 1.3M | 196M | 1% | /run |
| /dev/sda1 | 79G | 62G | 13G | 84% | / |
| tmpfs | 982M | 4.0K | 982M | 1% | /dev/shm |
| tmpfs | 982M | 8.0K | 982M | 1% | /tmp |
| tmpfs | 197M | 108K | 197M | 1% | /run/user/1000 |

The root filesystem `/dev/sda1` is 84% used (62G of 79G) with 5,251,072 inodes total and only 10% of those consumed — confirming disk space, not inode exhaustion, would be the limiting factor if this VM's storage ever filled up.

## 4. Directory Navigation Practice

```
mkdir -p notes
cd evidence; pwd
cd ..; pwd
cd ./notes; pwd
cd ../evidence; pwd
readlink -f lab2a_evidence.txt
```

| Step | Result |
|---|---|
| cd evidence → pwd | /home/kali/CIP-B101_Lab2A_Fuseini_Imoru/evidence |
| cd .. → pwd | /home/kali/CIP-B101_Lab2A_Fuseini_Imoru |
| cd ./notes → pwd | /home/kali/CIP-B101_Lab2A_Fuseini_Imoru/notes |
| cd ../evidence → pwd | /home/kali/CIP-B101_Lab2A_Fuseini_Imoru/evidence |
| readlink -f lab2a_evidence.txt | /home/kali/CIP-B101_Lab2A_Fuseini_Imoru/evidence/lab2a_evidence.txt |

`readlink -f` resolved the file to its full canonical absolute path, confirming relative-path navigation (`.`, `..`) all ultimately resolves to the same case-workspace location regardless of the current working directory at the time.

## 5. Case Notes: Creation, Copying, Renaming

```
cd ~/CIP-B101_Lab2A_Fuseini_Imoru
echo "Case Note 1: Initial Linux command line practice" > notes/case_note1.txt
echo "Case Note 2: Evidence handling requires careful documentation" > notes/case_note2.txt
cat notes/case_note1.txt
cat notes/case_note2.txt
cp notes/case_note1.txt evidence/case_note1_copy.txt
cp -r notes evidence/notes_backup
mv evidence/case_note1_copy.txt evidence/case_note1_renamed.txt
```

```
ls -la evidence
```

| File | Size | Modified |
|---|---|---|
| case_note1_renamed.txt | 49 bytes | Jun 27 12:37 |
| lab2a_evidence.txt | 54 bytes | Jun 27 07:05 |
| notes_backup/ (directory) | — | Jun 27 12:37 |

This confirms the full copy → recursive-directory-copy → rename workflow completed correctly: `case_note1.txt` was copied into `evidence/` as `case_note1_copy.txt`, then renamed in place to `case_note1_renamed.txt` using `mv` (which both renames and moves files in Linux — there is no separate "rename" command), while the entire `notes/` directory was separately duplicated into `evidence/notes_backup/` with `cp -r`.

## 6. Creating and Safely Removing a Temporary File

```
echo "temporary data" > evidence/temp_delete_me.txt
ls -la evidence/temp_delete_me.txt
rm evidence/temp_delete_me.txt
ls -la evidence
```

| Step | Result |
|---|---|
| Before removal | evidence/temp_delete_me.txt, 15 bytes, confirmed present |
| After `rm` | File no longer listed; `evidence/` back to 3 items + notes_backup |

This demonstrates `rm`'s permanent, non-recoverable deletion behavior (no trash/recycle bin at the shell level) — an important reminder that `rm` should never be used on actual evidence files without a verified working copy, since Linux's `rm` does not move files to a recoverable location the way a GUI file manager's delete does.

## 7. Building and Querying a Multi-Line Activity Log

```
cd ~/CIP-B101_Lab2A_Fuseini_Imoru
cat > evidence/activity_log.txt <<'EOF'
2026-06-19 09:10 user=aminu action=login status=success ip=192.168.56.10
2026-06-19 09:25 user=guest action=file_copy status=success file=report.docx
2026-06-19 10:05 user=admin action=login status=failed ip=192.168.56.15
2026-06-19 10:20 user=aminu action=delete status=success file=temp.txt
2026-06-19 10:40 user=admin action=password_change status=success
EOF
cat evidence/activity_log.txt
```

The heredoc (`<<'EOF'`) correctly wrote all five log lines verbatim (quoting `'EOF'` prevents any shell variable expansion inside the block, which matters when a log line might otherwise contain a `$` or backtick that the shell would try to interpret).

```
grep "login" evidence/activity_log.txt
```

| Match | Line |
|---|---|
| 1 | 2026-06-19 09:10 user=aminu action=login status=success ip=192.168.56.10 |
| 2 | 2026-06-19 10:05 user=admin action=login status=failed ip=192.168.56.15 |

```
grep -n "failed" evidence/activity_log.txt
```

| Line # | Match |
|---|---|
| 3 | 2026-06-19 10:05 user=admin action=login status=failed ip=192.168.56.15 |

```
grep -Ei "delete|password" evidence/activity_log.txt
```

| Match | Line |
|---|---|
| 1 | 2026-06-19 10:20 user=aminu action=delete status=success file=temp.txt |
| 2 | 2026-06-19 10:40 user=admin action=password_change status=success |

`-E` enabled extended regex (allowing the `|` alternation operator without escaping) and `-i` made the match case-insensitive; this single command located every delete-related and password-related event in one pass.

```
grep -Ei "192\.168\.56\.[0-9]+" evidence/activity_log.txt
```

| Match | Line |
|---|---|
| 1 | 2026-06-19 09:10 user=aminu action=login status=success ip=192.168.56.10 |
| 2 | 2026-06-19 10:05 user=admin action=login status=failed ip=192.168.56.15 |

This IP-address regex isolated only the two lines carrying a source IP in the 192.168.56.x range, filtering out the three log lines that don't involve a network-originated action (file_copy, delete, password_change) — a basic version of the same IP-extraction technique used on full packet captures in later, more advanced labs.

## 8. Confirming the Final Directory Tree

```
cd ~/CIP-B101_Lab2A_Fuseini_Imoru
tree
```

```
.
├── evidence
│   ├── activity_log.txt
│   ├── case_note1_renamed.txt
│   ├── lab2a_evidence.txt
│   └── notes_backup
│       ├── case_note1.txt
│       └── case_note2.txt
└── notes
    ├── case_note1.txt
    └── case_note2.txt

4 directories, 7 files
```

## 9. Locating Files with find

```
find . -type f -name "*.txt"
```

Returned all 7 `.txt` files in the tree: `./notes/case_note2.txt`, `./notes/case_note1.txt`, `./evidence/notes_backup/case_note2.txt`, `./evidence/notes_backup/case_note1.txt`, `./evidence/case_note1_renamed.txt`, `./evidence/lab2a_evidence.txt`, `./evidence/activity_log.txt`.

```
find . -type f -iname "*log*"
```

| Match |
|---|
| ./evidence/activity_log.txt |

```
find . -type d -name "evidence"
```

| Match |
|---|
| ./evidence |

These three `find` invocations demonstrate filtering by extension (`-name "*.txt"`), case-insensitive substring match on filename (`-iname "*log*"`), and filtering by type (`-type d` for directories only) — the three most common `find` patterns for locating relevant files in a large, unfamiliar directory tree during an investigation.

## 10. Network Interface and Routing Information

```
ip addr
```

| Interface | State | Address |
|---|---|---|
| lo | UP (LOOPBACK) | 127.0.0.1/8, ::1/128 |
| eth0 | UP | 192.168.44.128/24 (brd 192.168.44.255), fe80::fcdb:733e:9de1:56cb/64, MAC 00:0c:29:87:06:29 |

```
ip route
```

| Route | Detail |
|---|---|
| default | via 192.168.44.2 dev eth0 proto dhcp src 192.168.44.128 metric 100 |
| 192.168.44.0/24 | dev eth0 proto kernel scope link src 192.168.44.128 metric 100 |

This confirms the VM is DHCP-addressed on the 192.168.44.0/24 network (the same VMware host-only/NAT range seen in other labs in this course) with a default gateway of 192.168.44.2.

## 11. Basic Connectivity and Port Scanning

```
ping -c 4 127.0.0.1
```

| Result | 4 packets transmitted, 4 received, 0% packet loss, rtt min/avg/max/mdev = 0.024/0.064/0.156/0.053 ms |

```
ping -c 4 localhost
```

| Result | Resolved to ::1 (IPv6 loopback); 4 packets transmitted, 4 received, 0% packet loss, rtt min/avg/max/mdev = 0.055/0.080/0.152/0.041 ms |

Notably, `ping localhost` resolved to the IPv6 loopback address (`::1`) rather than the IPv4 `127.0.0.1` used in the explicit-IP test — a reminder that `localhost` resolution order (IPv6-first vs. IPv4-first) depends on the host's `/etc/nsswitch.conf`/`getaddrinfo` configuration and can silently differ from an explicit IPv4 target.

```
ss -lnt
```

Returned only the column header (State, Recv-Q, Send-Q, Local Address:Port, Peer Address:Port) with no listening TCP sockets — confirming no local services were listening at the time of this check.

```
nmap localhost -p 22,80,443 2>/dev/null || echo "nmap unavailable or no services detected"
```

| Port | State | Service |
|---|---|---|
| 22/tcp | closed | ssh |
| 80/tcp | closed | http |
| 443/tcp | closed | https |

Nmap's scan of localhost independently confirmed the `ss -lnt` finding: all three commonly probed ports (SSH, HTTP, HTTPS) are closed, consistent with no services listening locally — two separate tools agreeing on the same negative result.

## 12. Writing and Running a Reusable System-Summary Shell Script

```
mkdir -p scripts outputs
cd scripts
nano lab2a_sys_summary.sh
chmod +x lab2a_sys_summary.sh
ls -l lab2a_sys_summary.sh
./lab2a_sys_summary.sh
```

| Field | Value |
|---|---|
| Script | scripts/lab2a_sys_summary.sh |
| Permissions after chmod | -rwxrwxr-x |
| Size | 558 bytes |
| Output | "Summary saved to /home/kali/CIP-B101_Lab2A_Fuseini_Imoru/outputs/system_summary.txt" |

```
cat ~/CIP-B101_Lab2A_Fuseini_Imoru/outputs/system_summary.txt | head -n 30
```

The generated report includes a header, timestamp, username, hostname, kernel version, current working folder, a full `df -h` disk-space table, a full `df -i` inode-usage table, and an "Evidence Workspace Tree" section — i.e., the script automates exactly the host-identification and disk-usage checks performed manually in sections 1–3 of this lab into a single repeatable report.

| Report field | Value |
|---|---|
| Date/Time | Sat Jun 27 01:05:21 PM EDT 2026 |
| User | kali |
| Hostname | kali |
| Kernel | Linux kali 6.18.12+kali-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.18.12-1kali1 (2026-02-25) x86_64 GNU/Linux |
| Current Folder (first run) | /home/kali/CIP-B101_Lab2A_Fuseini_Imoru/scripts |

## 13. Adding the Script to PATH and Running It From Elsewhere

```
export PATH=$HOME/CIP-B101_Lab2A_Fuseini_Imoru/scripts:$PATH
cd /tmp
lab2a_sys_summary.sh
```

| Result | "Summary saved to /home/kali/CIP-B101_Lab2A_Fuseini_Imoru/outputs/system_summary.txt" — script ran successfully by name alone, with no `./` prefix or full path, confirming PATH was updated correctly |

```
ls ~/CIP-B101_Lab2A_Fuseini_Imoru/scripts
chmod +x ~/CIP-B101_Lab2A_Fuseini_Imoru/scripts/lab2a_sys_summary.sh
```

```
cat ~/CIP-B101_Lab2A_Fuseini_Imoru/outputs/system_summary.txt | head -n 15
```

| Report field (second run) | Value |
|---|---|
| Date/Time | Sat Jun 27 01:10:55 PM EDT 2026 |
| Current Folder | /tmp |

Re-running the same script from `/tmp` (rather than from inside `scripts/`) correctly updated the "Current Folder" field to `/tmp` while still writing its output to the same absolute `outputs/` path — confirming the script uses absolute paths for its output destination rather than paths relative to wherever it happens to be invoked from, which is exactly the behavior needed for a script meant to be called from anywhere via `PATH`.

## 14. Packaging the Case Workspace for Evidence Transfer

```
cd ~
zip -r CIP-B101_Lab2A_Fuseini_Imoru.zip CIP-B101_Lab2A_Fuseini_Imoru
```

The zip archive added every file and directory in the tree (outputs/system_summary.txt deflated 62%, notes/, evidence/ including notes_backup/, case_note1_renamed.txt, lab2a_evidence.txt, activity_log.txt deflated 54%, and scripts/lab2a_sys_summary.sh deflated 48%).

```
tar -czvf CIP-B101_Lab2A_Fuseini_Imoru.tar.gz CIP-B101_Lab2A_Fuseini_Imoru
```

The gzip-compressed tarball listed the identical set of files and directories as the zip archive, confirming both archive formats captured the complete, identical case workspace.

```
ls -lh CIP-B101_Lab2A_Fuseini_Imoru.zip CIP-B101_Lab2A_Fuseini_Imoru.tar.gz
```

| Archive | Size |
|---|---|
| CIP-B101_Lab2A_Fuseini_Imoru.tar.gz | 1.7K |
| CIP-B101_Lab2A_Fuseini_Imoru.zip | 4.9K |

The `.tar.gz` archive is notably smaller than the `.zip` (1.7K vs. 4.9K) for the identical content — gzip's compression on the concatenated tar stream is more efficient here than zip's per-file DEFLATE compression, particularly given how many small, highly repetitive text files make up this case workspace.

## Conclusion

**This lab built a complete, reproducible Linux command-line workflow for standing up, populating, searching, scripting against, and finally packaging a forensic case workspace — the same basic skillset every later lab in this course (disk forensics, memory forensics, network forensics) depends on underneath its specialized tooling.** Every file operation (create, copy, rename, delete) was verified immediately afterward with `ls -la`, `stat`, or `cat`; every search operation (`grep`, `find`) was demonstrated with both a positive and a targeted/filtered case; networking basics were cross-checked with two independent tools (`ss` and `nmap`) that agreed on the same result; and the automation script (section 12–13) was proven to behave correctly both from its own directory and after being added to `PATH` and invoked from an unrelated directory (`/tmp`). The final workspace was archived in two independent formats (zip and tar.gz) with matching contents, ready for transfer or submission as evidence.

## Key Tools Used

| Tool | Purpose |
|---|---|
| mkdir / cd / pwd / ls | Directory structure creation and navigation |
| cat / echo / heredoc (`<<'EOF'`) | File content creation, multi-line log generation, content display |
| stat | Detailed file metadata (size, permissions, inode, access/modify/change/birth timestamps) |
| cp / cp -r / mv / rm | File and directory copying, renaming, safe deletion |
| grep (-n, -i, -E) | Line-numbered, case-insensitive, and extended-regex pattern search in the activity log |
| find (-type, -name, -iname) | Locating files by extension, case-insensitive substring, and type |
| tree | Visual directory-tree confirmation |
| ip addr / ip route | Network interface and routing table inspection |
| ping | ICMP connectivity testing (IPv4 and IPv6) |
| ss -lnt | Listening TCP socket enumeration |
| nmap | Independent port-state cross-check |
| nano / chmod +x | Script authoring and making it executable |
| export PATH= | Adding the scripts directory to the shell's command search path |
| zip -r / tar -czvf | Archiving the completed case workspace in two formats |

## Forensic Notes

- `stat`'s four matching timestamps (Access/Modify/Change/Birth, all 2026-06-27 07:05:30.152271715 -0400) on the very first evidence file created in this lab establishes a clean baseline; any divergence between these four values on a file encountered in a real investigation is itself a forensic signal (e.g., a Modify time newer than Birth time indicates the file's content was edited after creation).
- The heredoc output for `activity_log.txt` displayed as "EOFEOF" in one terminal capture during this session — this is a rendering/display duplication artifact from the terminal capture tool, not a data-integrity issue; the subsequent `cat evidence/activity_log.txt` confirms the file's actual five-line content is correct and clean.
- `ping localhost` resolving to the IPv6 loopback (`::1`) while `ping 127.0.0.1` explicitly used IPv4 is a reminder that hostname resolution order is configuration-dependent and should not be assumed — an investigator relying on `localhost` in a script could silently test the wrong protocol stack on a differently configured host.
- `rm` in Linux performs immediate, non-recoverable deletion with no built-in recycle bin; this was demonstrated deliberately on a disposable temp file (`temp_delete_me.txt`) specifically so the behavior could be shown safely, separate from any actual case evidence.
- The `lab2a_sys_summary.sh` script writing its output to an absolute path (`~/CIP-B101_Lab2A_Fuseini_Imoru/outputs/system_summary.txt`) regardless of the directory it was invoked from is good scripting practice for any tool meant to be added to `PATH` and run from arbitrary locations — relative-path output would otherwise scatter report files across whatever directory happened to be current at invocation time.
- Both archive formats (zip and tar.gz) were verified to contain the identical file set via their respective verbose listings before being size-compared; the gzip-compressed tarball's smaller size (1.7K vs. 4.9K) reflects gzip's more effective compression of this particular file set, not any difference in content completeness.
- `export PATH=...` in section 13 only persists for the current shell session; a durable PATH addition for future sessions would require appending the line to `~/.bashrc` or `~/.zshrc`, which was not done in this lab and is worth noting for anyone reproducing this workflow outside a single sitting.
