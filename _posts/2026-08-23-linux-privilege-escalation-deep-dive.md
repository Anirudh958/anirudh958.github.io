---
title: "Linux Privilege Escalation: From Dirty COW to RefluXFS — How Low-Privilege Shells Become Root"
date: 2026-08-23 10:00:00 +0530
categories: [Cybersecurity, Linux]
tags: [linux, privilege-escalation, lpe, kernel-exploitation, suid, capabilities, dirty-cow, dirty-pipe, pwnkit, cve]
description: "A hands-on tour of Linux privilege escalation — SUID, capabilities, race conditions, and kernel memory bugs explained through 16 famous LPE CVEs, from Dirty COW and Dirty Pipe to PwnKit, Januscape, and RefluXFS."
image:
  path: /assets/img/social/linux-lpe.svg
  alt: "Linux privilege escalation: from Dirty COW to RefluXFS, how low-privilege shells become root"
toc: true
comments: true
---

## Introduction

In cybersecurity, getting the initial shell is only half the work. That shell can come from anywhere — a web server vulnerability, phishing, credential theft, a sloppy misconfiguration. And almost every time, you land as some low-privileged user who can barely read `/etc/shadow`, let alone touch it.

A shell that can't compromise the system isn't really a compromise. To truly own a Linux box you need to be root.

In this article I want to walk through the famous Linux Privilege Escalation (LPE) vulnerabilities — including several very recent ones — and keep it simple enough that anyone can follow along. Before we dissect the CVEs themselves, we'll cover the terms and concepts they're built on, because these vulnerabilities make a lot more sense once you understand what they're actually breaking. The same way you've always wanted to understand your girlfriend.

Here's the full list of LPEs we'll cover:

1. Dirty COW
2. Dirty Pipe
3. Copy Fail
4. Dirty Frag
5. Fragnesia
6. PEdit-CoW
7. Januscape
8. PwnKit
9. OverlayFS
10. Netfilter Use-After-Free
11. sudo "Baron Samedit"
12. eBPF Verifier Bug
13. Netfilter x_tables Heap OOB
14. ptrace Exit-Race
15. RefluXFS

![Timeline of Linux LPE disclosures from Dirty COW in 2016 through the 2026 wave of bugs](/assets/img/lpe/timeline.svg)

*A decade of Linux privilege escalation — and the "Dirty" family just keeps coming back.*

---

## The Basics of Access

Local Privilege Escalation is the act of exploiting a bug, a misconfiguration in the OS or software, or an outright design flaw — with the whole and sole goal of getting access to a privileged user that would normally be protected from an application or user like you. On Linux, it usually means moving from a standard user to root.

Root (UID 0) is the superuser account on Linux. Root has unrestricted access over the entire system — getting root is like getting access to MrBeast's wallet.

There are two other important terms for this article: **UserLand** and **Kernel Space**.

**UserLand** is where your applications run — web servers, text editors, daemons, all of it, executing with restricted privileges outside of the kernel. If the exploitable bug lives here, it's a UserLand LPE. PwnKit is the classic example.

The **kernel**, meanwhile, is the core of the operating system. It manages hardware and memory, runs with unrestricted access to system memory, CPU instructions, and everything else. Kernel vulnerabilities are usually far more severe for exactly that reason — and most of the bugs in this article live there.

---

## How Linux Permissions Work

Before understanding how Linux privilege escalation works, we need to understand how Linux hands out permissions to programs in the first place.

Normally, every process on a Linux system runs with the same privileges as the user who started it. You start a program, the program runs as *you*. Nothing magical happens:

```text
User (anirudh)
      │
      ▼
Runs a program
      │
      ▼
Program executes with anirudh's permissions
```

This mechanism is what stops normal users from reading sensitive files, modifying system configuration, creating user accounts, loading kernel modules, or rewriting firewall rules.

But some legitimate system programs genuinely *need* privileged permissions when executed by normal users:

- When you change your own password, something has to write to `/etc/shadow` — which only root can modify.
- `sudo` must execute commands as root.
- `mount` needs elevated privileges to mount filesystems.

Linux provides two mechanisms to make this work safely: **SUID** and **Linux Capabilities**. Both allow processes to perform privileged actions, but they work completely differently — and both are prime hunting ground during privilege escalation.

### SUID

SUID stands for **Set User ID**. It's a special permission bit applied to executable files. When an executable has the SUID bit set, the program runs with the permissions of the file's *owner*, not the user who launched it.

Normally:

```text
User executes program
        │
        ▼
Program runs as the user
```

With SUID:

```text
User executes program
        │
        ▼
Program temporarily becomes
the owner of the executable
```

If the executable is owned by **root**, the program temporarily gains root privileges while running.

### Example

Suppose we have:

```console
-rwsr-xr-x 1 root root passwd
```

Notice the `s` in the owner's execute position:

```text
-rwsr-xr-x
    ^
    SUID
```

The owner is root. Now imagine a normal user runs `passwd`. Even though the user is not root, the program executes as root. Why? Because `passwd` needs to modify `/etc/shadow`, which is writable only by root. Without SUID, changing your own password simply wouldn't be possible.

### Why Attackers Love SUID

Because SUID programs execute with elevated privileges, any vulnerability inside them becomes extremely dangerous. Imagine a SUID program that executes another command insecurely:

```c
system("backup.sh");
```

If an attacker can manipulate the environment or replace `backup.sh`, they might force the program to execute their own commands as root. This is why so many LPE vulnerabilities target SUID binaries.

Some commonly abused SUID programs include:

- `sudo`
- `pkexec`
- `find`
- `vim`
- `less`
- `cp`
- `bash` (when misconfigured)

During a privilege escalation assessment, one of the first enumeration steps is:

```bash
find / -perm -4000 -type f 2>/dev/null
```

This lists every file with the SUID bit enabled.

---

## Linux Capabilities

Historically, Linux only had two privilege levels: root or not-root. If a program needed one privileged operation, it often had to run as full root. That violates the Principle of Least Privilege, which says:

> A program should receive only the permissions it actually needs — nothing more.

To fix this, Linux introduced capabilities. Instead of handing over complete root access, the kernel can grant individual privileged abilities.

Think of root as a collection of powers rather than one giant permission:

```text
Root Privileges
│
├── Bind to low ports
├── Change file ownership
├── Load kernel modules
├── Configure networking
├── Change system time
├── Kill arbitrary processes
├── Mount filesystems
└── ...
```

Capabilities split these powers into independent units, and a process can receive only the capability it needs without gaining all the others.

For example, suppose a web server wants to listen on port 80. Ports below 1024 require root, so without capabilities the web server must run as full root. With capabilities, the server gets exactly one thing:

```text
CAP_NET_BIND_SERVICE
```

Now it can bind to port 80 without ever becoming root — and if that web server gets popped, the attacker inherits far less than they would have otherwise.

| Capability             | Allows                                                   |
| ---------------------- | -------------------------------------------------------- |
| `CAP_NET_BIND_SERVICE` | Bind to ports below 1024                                 |
| `CAP_SYS_ADMIN`        | Perform many system administration tasks (very powerful) |
| `CAP_SETUID`           | Change user IDs                                          |

Attackers look for capabilities for the same reason they look for SUID bits: a binary with dangerous capabilities may never be marked SUID, yet it can still perform privileged operations. If a binary holds `CAP_SETUID` and contains a vulnerability, an attacker may be able to change its effective user ID to root. During assessments, the go-to enumeration command is:

```bash
getcap -r / 2>/dev/null
```

This recursively lists every file with capabilities assigned.

### Why This Matters for Privilege Escalation

When attackers land on a Linux machine, they rarely stop at a normal user shell — the next objective is always root, because root has unrestricted control over the system. To get there, attackers hunt for mechanisms that already possess elevated privileges. The two big ones are SUID binaries (which execute as their owner) and capabilities (which grant specific privileged operations without full root).

If either of those mechanisms is misconfigured or contains a software vulnerability, it can be abused to perform actions beyond the attacker's original permissions — which is precisely how Local Privilege Escalation happens. This is why almost every Linux privesc checklist starts by enumerating SUID binaries and assigned capabilities. They're usually the most promising path to root.

---

## Common Vulnerability Mechanisms

Most Linux privilege escalation vulnerabilities rely on a handful of recurring mechanisms. Before we look at real CVEs, let's nail down the four that matter most:

1. Race Conditions / Time-Of-Check-To-Time-Of-Use (TOCTOU)
2. Copy-On-Write (CoW)
3. Use-After-Free
4. Out-Of-Bounds Write

### Race Conditions and TOCTOU

Race conditions are famous for finding LPEs — most of the vulnerabilities in this article lean on them in some way.

Modern operating systems perform thousands of operations simultaneously. Multiple programs, processes, and threads may try to access the same resource at nearly the same time. A race condition occurs when the correctness of a program depends on the precise timing of two or more operations. If an attacker can squeeze an action in between two critical operations, they can trick the program into doing something it was never meant to do.

Consider a literal race: an operation trying to complete its task safely, and an attacker trying to interfere before the task finishes. Whoever wins the race decides what happens — hence "race condition."

TOCTOU (Time-Of-Check-To-Time-Of-Use) is the classic race condition pattern. Imagine a program that wants to delete a file only if it belongs to the current user:

```c
if (file_belongs_to_user("data.txt")) {
    delete_file("data.txt");
}
```

Between checking the file and deleting it, there is a small time gap:

```text
             CHECK                    USE
               ↓                       ↓
       ┌────────────────┐      ┌──────────────┐
       │ Is data.txt    │      │ Delete       │
       │ owned by me?   │      │ data.txt     │
       └────────────────┘      └──────────────┘
                 \                /
                  \              /
                      TIME GAP
```

An attacker exploits that gap:

```text
Program:  "data.txt belongs to me"
                 ↓
           [TIME GAP]
                 ↓
Attacker: swaps data.txt → /etc/passwd
                 ↓
Program:  deletes /etc/passwd
```

The program checked one object, but by the time it used it, it was a different object entirely.

A more realistic privileged-program version looks like this:

```c
if (access("/tmp/file", W_OK) == 0) {
    fd = open("/tmp/file", O_WRONLY);
}
```

`access()` checks whether the file is writable. But between `access()` and `open()`, an attacker can replace `/tmp/file` with a symlink pointing somewhere else entirely.

### Copy-On-Write

One of the most important places you encounter CoW is `fork()`:

```c
fork();
```

When a process calls `fork()`, Linux creates a child process. You might imagine it works like this:

```text
Parent
  ↓
Copy entire memory
  ↓
Child
```

That would be brutally expensive if the parent has hundreds of MB or even GB of memory. Instead, Linux uses Copy-on-Write: initially, both processes' virtual memory pages point at the same physical pages, and those pages are marked read-only:

```text
              Physical RAM
            ┌──────────────┐
Parent ────►│   Memory     │◄──── Child
            └──────────────┘
```

If both processes only read, everything stays shared and cheap:

```text
Parent ──READ──┐
               ├──► Same physical page
Child  ──READ──┘
```

The moment one side writes, the kernel steps in. Say the child executes `x = 100;`. Writing to a shared read-only page triggers a page fault, and the kernel creates a private copy:

```text
Child
  │ WRITE
  ▼
Page fault
  │
  ▼
Kernel creates a copy
  │
  ├──── Parent → Original page
  │
  └──── Child  → New page
```

Keep this picture in your head. Dirty COW, PEdit-CoW, and RefluXFS are all about breaking assumptions in this exact machinery.

### Use-After-Free

Use-After-Free is a memory-safety vulnerability that occurs when a program keeps using memory after it has already been freed. After `free()`, the program no longer owns that memory:

```c
char *data = malloc(100);

strcpy(data, "Hello");

free(data);

/* ❌ Use-After-Free */
printf("%s\n", data);
```

What's happening under the hood:

```text
malloc()
   ↓
┌──────────────┐
│ "Hello"      │
└──────────────┘
      ↑
     data

free(data)
      ↓
Memory is released
      ↓
┌──────────────┐
│  AVAILABLE   │
└──────────────┘
      ↑
     data
```

The pointer `data` still contains an address, but the memory at that address is no longer valid for the program to use. So `printf("%s", data);` after the free is textbook Use-After-Free.

Why do attackers care? Because freed memory doesn't vanish — it gets reused. If an attacker can control what gets allocated into that freed chunk, the old code path ends up operating on attacker-controlled data. We'll see this exact trick in the Netfilter section.

### Out-of-Bounds Write

An out-of-bounds write happens when a program writes outside the memory space it was allocated — whether that's a variable, an array, a buffer, or an object. In simple terms: the program writes past the boundary it's allowed to use.

Simple C example:

```c
int arr[5];

arr[0] = 10;
arr[1] = 20;
arr[2] = 30;
arr[3] = 40;
arr[4] = 50;

arr[5] = 60;   // ❌ Out-of-bounds write
```

`arr` has five elements:

```text
Index:    0    1    2    3    4
         ┌────┬────┬────┬────┬────┐
Memory:  │ 10 │ 20 │ 30 │ 40 │ 50 │
         └────┴────┴────┴────┴────┘
                                   ↑
                                boundary
```

But `arr[5] = 60;` tries to write past the end:

```text
         ┌────┬────┬────┬────┬────┐
         │ 10 │ 20 │ 30 │ 40 │ 50 │
         └────┴────┴────┴────┴────┘
                                     │
                                     ▼
                             ❌ Write outside
                                the array
```

The program is writing into memory that doesn't belong to `arr`. In userland that's already bad news; inside the kernel, corrupting an adjacent object can mean game over for the whole system.

---

## Now Let's Talk About the Real Stuff

We've covered the fundamentals. We know UserLand from Kernel Space, how SUID and capabilities grant privileges, and the four vulnerability mechanics that power most LPEs.

Time for the actual point of this article: **the CVEs**.

Knowing what a race condition is theoretically is fine, but seeing the same concept used to turn a normal Linux user into root is where things get interesting. And one more thing — I'm not going to just say *"this CVE is vulnerable, run this exploit and you're root."* That teaches you almost nothing. For every vulnerability we'll try to understand:

```text
What was wrong?
       ↓
Where was it wrong?
       ↓
What primitive did the attacker get?
       ↓
How did that primitive become root?
       ↓
How was it fixed?
```

That chain is what actually matters when you're doing Linux privilege escalation.

---

## 1) Dirty COW — CVE-2016-5195

Let's start with the vulnerability that made the word "Dirty" famous in Linux privilege escalation.

Dirty COW stands for Dirty Copy-On-Write, and yes — it attacks exactly the CoW concept we just discussed. The vulnerability lived in the kernel's memory management code: a race condition in the handling of Copy-on-Write memory mappings.

Normally, if a process has a read-only private mapping and tries to modify it, Linux creates a private copy of the page. The original read-only page should never be touched. Dirty COW raced the kernel's memory-management operations such that the attacker ended up with a writable view of a page that should have stayed read-only forever.

In simple terms, the expected flow:

```text
Read-only file
      ↓
Memory mapping
      ↓
Process tries to write
      ↓
Kernel creates private copy
      ↓
Original file remains untouched
```

Dirty COW breaks that assumption:

```text
Read-only mapping
       │
       ├──── attacker writes
       │
       └──── kernel performs COW
                │
                ▼
        Race condition wins
                │
                ▼
       Original page modified
```

Now the interesting question: what can an attacker do if they can modify something they're only supposed to read?

Quite a lot. If the attacker can manipulate the contents of a privileged executable or another sensitive file, that write primitive becomes a root execution path. Dirty COW didn't merely give "write access" — it broke a fundamental assumption of the memory management system: *a read-only private mapping must not allow modification of the underlying protected data.* It affected kernels across the 2.x through 4.x generations before the relevant fixes and was exploited in the wild.

The chain to remember:

```text
Dirty COW
    ↓
Race Condition
    ↓
Copy-On-Write handling
    ↓
Write to protected memory
    ↓
Modify privileged data
    ↓
Privilege Escalation
```

This is probably the best example of why understanding CoW plus race conditions pays off.

---

## 2) Dirty Pipe — CVE-2022-0847

Six years after Dirty COW came the next famous "Dirty" bug: Dirty Pipe. Unlike Dirty COW, it doesn't attack copy-on-write directly. It abuses something much more interesting:

```text
pipe + splice() + page cache
```

The root cause was incorrect initialization of flags in the Linux pipe buffer implementation. Those stale flags could make a pipe buffer be incorrectly treated as mergeable. Why does that matter? Because Linux uses the page cache as backing storage for file data. Dirty Pipe found a way to get a pipe buffer associated with a page belonging to a read-only file — then data written through the pipe could affect that page-cache page.

Sit with that for a second. The attacker has:

```text
READ access
```

but somehow obtains a:

```text
WRITE primitive
```

against the page cache of a protected file. The vulnerability let an unprivileged local user overwrite page-cache pages backed by read-only files — including files the attacker couldn't normally write. The kicker is that the attacker doesn't need filesystem write permission at all. The kernel itself performs the write:

```text
Attacker
   │
   ▼
pipe()
   │
   ▼
splice()
   │
   ▼
Page cache
   │
   ▼
Protected file's cached page
```

And this is where Dirty Pipe becomes an LPE: if the attacker can modify something security-sensitive through the page cache, that write can be converted into execution with higher privileges.

Dirty Pipe teaches an important lesson — **filesystem permissions are not necessarily the whole story**. A file can be `-r--r--r--` and you can still affect its cached representation if the kernel has a bug in the code path handling that data. That's exactly why kernel vulnerabilities are so nasty. The attacker doesn't convince `chmod` to hand them anything. They convince kernel code itself to perform an operation it shouldn't.

---

## 3) Copy Fail — CVE-2026-31431

Now we get to one of the newer ones, and it's interesting because it follows the same general family as Dirty Pipe even though the underlying bug is completely different.

Copy Fail affects the kernel's `algif_aead` cryptographic interface. The interesting combination:

```text
AF_ALG + splice() + page cache + incorrect in-place cryptographic operation
```

The bug eventually provides a controlled write into the page cache. The important difference from classic LPEs is that this isn't primarily about winning a race — it's a **logic flaw**. The attacker doesn't sit around hoping "maybe the race will land this time." The vulnerable kernel code follows a path that incorrectly allows attacker-controlled data to reach a page-cache page, deterministically. It was publicly disclosed in April 2026, and Red Hat describes it as a local privilege escalation in the kernel cryptographic interface.

Here's the relationship I really want you to notice:

```text
Dirty Pipe
     │  page-cache write
     ▼
Copy Fail
     │  page-cache write
     ▼
Dirty Frag
     │  page-cache write
     ▼
More bugs in the same general family
```

Attackers don't search for "new root exploit." They search for **reusable primitives**. Once researchers understand *how an unprivileged process can make the kernel write into the page cache*, they can start hunting for the same mistake in completely different kernel subsystems. That's how vuln research evolves.

---

## 4) Dirty Frag — CVE-2026-43284 and Related

Next up is Dirty Frag, and it continues the page-cache story.

Dirty Frag lives in Linux networking code dealing with shared page fragments. The affected paths include the XFRM ESP subsystem and RxRPC. The core problem: data associated with a page-cache page ends up being modified through a kernel path that assumed it owned a writable buffer. Familiar pattern, right?

```text
Protected file
      ↓
Page cache
      ↓
Kernel networking subsystem
      ↓
Unexpected write
      ↓
Protected page modified
```

Dirty Frag was disclosed in May 2026 as a local privilege escalation that could also allow container escape. Multiple CVEs are involved in the broader issue — CVE-2026-43284 and CVE-2026-43500 among them, with additional related fixes tracked by distributions.

The important thing isn't memorizing function names. It's understanding the primitive:

```text
Unprivileged user
       ↓
Networking primitive
       ↓
Kernel incorrectly handles shared fragment
       ↓
Page-cache write
       ↓
Modify privileged file
       ↓
Root
```

This is also why I put Dirty Pipe, Copy Fail, and Dirty Frag back to back. Different vulnerabilities, different subsystems — but the underlying idea rhymes: *a kernel subsystem writes somewhere the attacker should only be able to read.*

---

## 5) Fragnesia — CVE-2026-43503 / CVE-2026-46300

Fragnesia is another 2026 vulnerability family involving socket-buffer fragments. The name comes from the same neighborhood as Dirty Frag — shared fragment information being propagated through networking paths incorrectly.

For us, the important part is the flow:

```text
Network buffer
      ↓
Shared fragment
      ↓
Kernel assumes ownership
      ↓
Write occurs
      ↓
Page cache can be affected
```

Ubuntu describes Fragnesia as a flaw in the XFRM ESP-in-TCP path allowing a local attacker to escalate privileges or escape a container.

Notice how many recent vulnerabilities are starting to look alike. That's not an accident. Modern Linux is enormous — filesystems, networking, cryptography, virtualization, memory management, packet processing, namespaces, eBPF, IPC — and all of these subsystems interact. A vulnerability doesn't need to live in a "privileged program." Sometimes the dangerous part is simply:

```text
Subsystem A
      ↓ passes object to
Subsystem B
      ↓ B assumes something A never guaranteed
```

That assumption can become an LPE.

---

## 6) PEdit-CoW — CVE-2026-46331

This one is a beautiful example of why CoW matters.

PEdit-CoW lives in Linux traffic control, specifically `net/sched/act_pedit.c`. The `pedit` action lets packet headers be modified, and before touching a packet buffer the kernel must ensure it holds a private writable copy — that's literally what Copy-On-Write guarantees.

The buggy implementation calculated the writable range too early:

```text
Calculate writable range
        ↓
Assume this range is enough
        ↓
Process packet-editing keys
        ↓
Actual offset changes
        ↓
Write lands outside the COW'ed area
```

So part of the buffer could remain shared while the kernel believed it was private. That's the bug. The kernel writes believing *"this memory belongs to me"* when reality says *"this memory is still shared."* And if that shared memory ultimately refers to a page-cache page, the write can hit protected file data.

CVE-2026-46331's kernel fix moved the COW operation into the per-key processing path, so the actual write offset is known before making the buffer writable. Sensible.

The resulting chain:

```text
Bad COW assumption
       ↓
Shared buffer stays shared
       ↓
Attacker-controlled packet modification
       ↓
Unexpected memory write
       ↓
Page-cache corruption
       ↓
LPE
```

---

## 7) Same Primitive, Different Bugs

At this point you might be thinking: *"Bro, aren't all of these the same vulnerability?"*

No. And the distinction matters.

| Vulnerability | Main idea                                             |
| ------------- | ----------------------------------------------------- |
| Dirty COW     | Race in Copy-On-Write memory handling                 |
| Dirty Pipe    | Pipe buffer/page-cache bug                            |
| Copy Fail     | AF_ALG cryptographic path causes page-cache write     |
| PEdit-CoW     | Incorrect COW range in traffic-control packet editing |
| Dirty Frag    | Shared network fragments reach page-cache write paths |
| Fragnesia     | Fragment handling flaw in networking/XFRM paths       |

Different bug, different subsystem, different root cause. But sometimes — the same final primitive. And that overlap is exactly what makes vulnerability research interesting.

![Map of how fifteen LPE CVEs collapse into four exploitation families that all end at root](/assets/img/lpe/primitive-family.svg)

*Fifteen CVEs, four families, one destination.*

---

## 8) Januscape — CVE-2026-53359

Let's leave the page cache behind, because not every LPE is about files.

Januscape lives deep in the stack:

```text
Linux Kernel → KVM → x86 Shadow MMU
```

KVM is the Kernel-based Virtual Machine subsystem. So instead of the usual `normal user → root`, the boundary being attacked looks like this:

```text
Guest VM
   ↓
KVM
   ↓
Host kernel
```

The rule is supposed to be: guests stay in their box. Januscape breaks it. The bug is a Use-After-Free in KVM's x86 shadow MMU code — the component managing shadow page tables. A role mismatch could cause KVM to reuse a shadow page incorrectly, letting guest-controlled operations corrupt host kernel state. It hits both Intel and AMD x86 KVM environments, and reportedly saw action as a zero-day in Google's kvmCTF.

Think about the impact. If I compromise my guest VM, normally I control... my VM. But if the KVM boundary is broken:

```text
Attacker
   ↓
Guest
   ↓
KVM vulnerability
   ↓
Host kernel
   ↓
Host root
   ↓
Other guests
```

At that point we're no longer talking about a regular LPE — we're talking about a **VM escape**. Which is why hypervisor bugs get taken extremely seriously.

---

## 9) PwnKit — CVE-2021-4034

Back to UserLand. PwnKit is one of the best examples of userland privilege escalation out there.

The vulnerable component is `pkexec`, part of polkit, and it carries one crucial detail: pkexec ships installed as a **SUID-root executable**. So the setup is:

```text
Normal user
      ↓
pkexec
      ↓
SUID
      ↓
root privileges
```

The problem: pkexec mishandled its command-line arguments. An attacker could abuse argument and environment processing to make pkexec execute attacker-controlled code — and once arbitrary code runs inside a SUID-root process:

```text
Attacker-controlled code
        ↓
pkexec
        ↓
SUID root
        ↓
Root
```

Game over. Remember earlier when I said SUID programs are dangerous when they contain vulnerabilities? PwnKit is that sentence turned into an actual CVE.

---

## 10) OverlayFS — CVE-2023-0386

OverlayFS is interesting because it combines four things that shouldn't mix badly, yet did:

```text
Filesystem + User namespaces + SUID + Permission handling
```

OverlayFS stacks multiple filesystem layers into one merged view:

```text
Lower Layer + Upper Layer
        ↓
    OverlayFS
        ↓
 Merged filesystem
```

The vulnerability related to how OverlayFS handled ownership and capabilities while copying files between mounts — an incorrect UID-mapping/ownership assumption. An attacker could abuse the behavior to place a capable/SUID file somewhere it would execute with elevated privileges.

NVD describes CVE-2023-0386 as an OverlayFS flaw letting an unprivileged local user execute a setuid file with capabilities and escalate privileges. It also earned a spot on CISA's Known Exploited Vulnerabilities catalog.

The general idea:

```text
Unprivileged user
       ↓
User namespace
       ↓
OverlayFS
       ↓
Ownership confusion
       ↓
SUID/capable file
       ↓
Root
```

Another lesson here: Linux namespaces are not automatically a security boundary. They're extremely useful, sure — but every new boundary means new kernel code, and new kernel code means more places where assumptions can go wrong.

---

## 11) Netfilter Use-After-Free — CVE-2024-1085 / CVE-2024-1086

Time for Netfilter — one of those components nearly every Linux admin uses without thinking. It sits underneath iptables, nftables, firewalling, packet filtering, NAT... basically anything firewall-shaped. So when a memory-safety bug shows up in Netfilter, things get interesting fast.

CVE-2024-1085 is a use-after-free/double-free condition in the `nf_tables` subsystem, involving catch-all set elements handled across generations. Simplified:

```text
Object exists
    ↓
Kernel decides it should be freed
    ↓
Object state is not tracked correctly
    ↓
Object gets freed again
    ↓
Double free
    ↓
Memory corruption
```

CVE-2024-1086 is another nf_tables UAF, this time involving `nft_verdict_init()` and an invalid verdict/error interaction. Both were capable of local privilege escalation.

This is where our Use-After-Free groundwork pays off. Remember — `free(object);` doesn't erase the pointer, and the memory can be reused later:

```text
Old object
    ↓
free()
    ↓
Memory available
    ↓
Attacker-controlled object allocated
    ↓
Kernel still treats it as the old object
```

Now the attacker has a memory corruption primitive. And once you've got controlled kernel memory corruption, the usual goal looks like:

```text
Kernel memory corruption
        ↓
Control kernel execution/state
        ↓
Modify credentials
        ↓
UID 0
        ↓
Root
```

---

## 12) Baron Samedit — CVE-2021-3156

We cannot talk Linux privilege escalation without sudo — and more specifically, Baron Samedit.

The vulnerability was a heap-based buffer overflow in `sudo`, triggered by incorrect handling of escaped backslashes while processing command-line arguments in `sudoedit`.

Here's what makes it special: `sudo` is already a privileged program. Normally:

```text
User
  ↓
sudo
  ↓
Authorization check
  ↓
Run command as root
```

The attacker doesn't bypass the authorization logic at all. They skip straight past it:

```text
User
  ↓
sudo
  ↓
Memory corruption
  ↓
Code execution
  ↓
Root
```

That's a distinction worth internalizing: sometimes you don't exploit the authentication mechanism — you exploit the program implementing it. Qualys demonstrated working exploitation on systems including Ubuntu 20.04, Debian 10, and Fedora 33, though exact affected versions depend on the installed sudo build.

This is why, during enumeration, I always want to see:

```bash
sudo --version
```

Not because every old version is automatically vulnerable — but because version info tells you what attack surface you actually have.

---

## 13) eBPF Verifier Bug — CVE-2017-16995

I listed this one up top as "eBPF Map Update," which honestly undersells it. The actual CVE-2017-16995 root cause is a bug in the Linux eBPF verifier involving incorrect sign extension.

Quick context: eBPF allows user-supplied programs to be loaded into the kernel. Obviously Linux can't respond to that with *"sure bro, here's kernel execution, do whatever you want"* — so eBPF has a verifier:

```text
User
  ↓
eBPF program
  ↓
Verifier — is this safe?
  ↓
Kernel execution
```

That verifier is itself a security boundary. If it wrongly believes a malicious program is safe:

```text
Malicious BPF
      ↓
Verifier mistake
      ↓
Unsafe operations accepted
      ↓
Kernel memory corruption
      ↓
Potential arbitrary read/write
      ↓
Privilege escalation
```

CVE-2017-16995 involved incorrect sign extension in `check_alu_op()` and could lead to memory corruption and potentially arbitrary code execution. The general class is worth remembering: **a verifier bug becomes a kernel memory-corruption bug.** The verifier doesn't need to execute malicious code itself — it just needs to approve something that should have been rejected.

---

## 14) Netfilter x_tables Heap OOB — CVE-2021-22555

Another great example of the Out-of-Bounds Write concept from earlier.

The vulnerability existed in `net/netfilter/x_tables.c` and allowed a heap out-of-bounds write when processing certain compat rulesets. The basic problem was a wrong assumption about how much space was available for translated rule data:

```text
Allocated buffer
┌───────────────────────────────┐
│                               │
│          Valid data           │
│                               │
└───────────────────────────────┘
                                ↑
                              boundary

Kernel writes here
                                           ↓
                                    ❌ OOB WRITE
```

The attacker gets a write outside the intended heap object, corrupting neighboring kernel objects. From there, exploitation becomes the art of turning small memory corruption into a useful primitive:

```text
controlled pointer
      ↓
kernel read/write
      ↓
credential manipulation
      ↓
root
```

CVE-2021-22555 affected kernels going back to the 2.6.19-rc1 era and later landed on CISA's Known Exploited Vulnerabilities catalog. If you want to learn kernel exploitation, this one is particularly worth studying — it demonstrates how a seemingly tiny heap overflow becomes something much more powerful.

---

## 15) ptrace Exit-Race — CVE-2026-46333

A recent one, and it goes right back to our first topic: race conditions.

The vulnerable component is the Linux ptrace access-check path, and the fun happens while a privileged process is exiting:

```text
Privileged process
        │
        ▼
     do_exit()
        │
        ├── memory released
        │    ← race window
        └── file descriptors closed
```

There's a short window where `task->mm` has already been released while the process still holds its file descriptors. The buggy access-check logic allowed operations during exactly this state, and an attacker racing the condition with `pidfd_getfd()` can obtain file descriptors belonging to the privileged process.

So instead of `attacker → directly become root`, the chain looks like:

```text
Attacker
   ↓
Race privileged process exit
   ↓
Bypass ptrace access check
   ↓
Steal privileged file descriptor
   ↓
Read sensitive data / interact with privileged service
   ↓
Privilege escalation
```

Red Hat describes CVE-2026-46333 as a process-exit race allowing low-privileged users to access sensitive files or execute commands through stolen descriptors. It's a lovely modern TOCTOU-style problem: the kernel checks a process at one moment in its lifetime, but the process changes state underneath the check.

And this is exactly why race conditions are so nasty — the code can look perfectly reasonable line by line. The flaw only appears when you ask: *"what if another CPU mutates state between these two operations?"*

---

## 16) RefluXFS — CVE-2026-64600

Finally, the newest bug on our list: RefluXFS.

It lives inside XFS and — once again — involves Copy-On-Write plus a race condition. Full circle, as promised.

The vulnerability exists in XFS's reflink/CoW path. Reflink lets two files share the same underlying filesystem blocks instead of immediately copying all the data:

```text
File A ─────┐
             ├──► Same physical blocks
File B ─────┘
```

Great feature — copying huge files gets much cheaper. But sharing blocks creates a synchronization problem: if two operations touch the shared data at the wrong time, the kernel must separate the blocks before modification. RefluXFS races exactly that logic, and the result can be far worse than corrupting the attacker's own file — protected file contents can be overwritten on affected XFS filesystems. Which immediately gives us our LPE primitive:

```text
Normal user
     ↓
XFS reflink race
     ↓
Overwrite readable protected file
     ↓
Modify privileged target
     ↓
Execute privileged code
     ↓
Root
```

Qualys disclosed CVE-2026-64600 in July 2026 and reported successful exploitation against affected XFS filesystems with reflink enabled.

Note the requirement here: **XFS with reflink support**. Unlike some kernel bugs where `uname -r` is nearly the whole analysis, here the filesystem configuration matters too. On a target, check:

```bash
findmnt -T /
```

and if root is XFS:

```bash
xfs_info /
```

Then look for:

```text
reflink=1
```

Kernel version still matters — but so does the filesystem underneath it.

---

## What Did We Actually Learn From These CVEs?

Step back for a second. We've covered a lot of bugs, and there's a pattern hiding under all of them:

```text
Dirty COW          → Race + CoW
Dirty Pipe         → Pipe + Page Cache
Copy Fail          → AF_ALG + Page Cache
Dirty Frag         → Network fragments + Page Cache
PEdit-CoW          → Incorrect CoW
RefluXFS           → XFS + CoW + Race
Netfilter UAF      → Use-After-Free
Baron Samedit      → Heap Overflow
x_tables           → Heap OOB Write
PwnKit             → SUID + Argument handling
OverlayFS          → Namespace + Filesystem permission confusion
Januscape          → KVM + Use-After-Free
ptrace Exit-Race   → Race + Privileged process state
eBPF               → Verifier mistake + Kernel memory corruption
```

Which is why I don't recommend memorizing CVE names. Remember **the primitive**.

---

## The Real Privilege Escalation Mindset

When you get a shell on a Linux machine, don't immediately think *"which exploit should I run?"* Think like an investigator instead:

```text
What am I?
        ↓
Which kernel?
        ↓
Which distribution?
        ↓
Which architecture?
        ↓
What SUID binaries exist?
        ↓
What capabilities exist?
        ↓
Can I create user/network namespaces?
        ↓
What kernel modules are loaded?
        ↓
What services are running?
        ↓
What filesystem am I using?
        ↓
Are there known vulnerable components?
        ↓
Can I turn the vulnerability into a useful primitive?
```

Concretely:

```bash
uname -a
```

gives you kernel and architecture information. Then:

```bash
cat /etc/os-release
```

tells you the distribution. For SUID:

```bash
find / -perm -4000 -type f 2>/dev/null
```

For capabilities:

```bash
getcap -r / 2>/dev/null
```

For the filesystem:

```bash
findmnt
```

Then start asking questions. Not *"can I run Dirty Pipe?"* but *"is this system in the affected kernel range?"*, *"do the required kernel features exist?"*, and *"can the vulnerability actually give me the primitive I need?"*

That difference separates people who run exploits from people who understand privilege escalation.

---

## CVE ≠ Automatic Root

Probably the most important thing in this entire article: finding a vulnerable CVE does **not** automatically mean root.

```text
CVE found ≠ root
```

There are conditions:

```text
CVE
 │
 ├── Kernel version
 ├── Architecture
 ├── Configuration
 ├── Required module
 ├── Namespace support
 ├── Filesystem
 ├── Mitigations
 └── Exploit reliability
```

All of them matter. A vulnerability can exist in the kernel source while your specific machine remains unexploitable because the required module is disabled, user namespaces are turned off, the filesystem is different, or the vendor backported the patch.

That last one deserves emphasis: **do not blindly compare `uname -r` against a random CVE website.** Distributions backport security fixes. An Ubuntu kernel reporting `5.x` does not automatically mean "old vulnerable kernel" — the vendor may have backported the patch while keeping the versioning scheme intact. Always check the distribution's security advisory too.

---

## Final Thoughts

Linux privilege escalation isn't about collecting hundreds of exploit scripts. It's about understanding the relationship between:

```text
User → Process → Permissions → Kernel → Memory
     → Filesystems → Namespaces → Privileged Components
```

Once you understand that relationship, CVEs become dramatically easier to digest:

- Dirty COW teaches us **Copy-On-Write + race conditions**
- Dirty Pipe teaches us **pipes + page cache**
- Copy Fail teaches us that **kernel subsystem interactions can create unexpected primitives**
- PwnKit proves **a vulnerable SUID program is an instant privesc path**
- Baron Samedit shows why **memory corruption inside privileged utilities is lethal**
- Netfilter bugs demonstrate **kernel heap corruption crossing privilege boundaries**
- Januscape reminds us that **virtualization itself is a security boundary**
- RefluXFS brings us home: **race conditions + CoW can still break modern Linux**

So next time you get a shell on a Linux box, don't just ask *"how do I get root?"* Ask:

> **"What does this system trust that I shouldn't be able to control?"**

That's where real privilege escalation begins.

---

## References & Further Reading

I kept the write-ups above intentionally conceptual. If you want to go deeper than my summaries, start with these primary sources:

**Dirty COW — CVE-2016-5195**

* [dirtycow.ninja](https://dirtycow.ninja) — the original vulnerability site
* [NVD: CVE-2016-5195](https://nvd.nist.gov/vuln/detail/CVE-2016-5195)

**Dirty Pipe — CVE-2022-0847**

* [Max Kellermann's original disclosure](https://dirtypipe.cm4all.com/) — written by the researcher who found it; required reading if you want the full page-cache story
* [NVD: CVE-2022-0847](https://nvd.nist.gov/vuln/detail/CVE-2022-0847)

**Copy Fail — CVE-2026-31431**

* [Red Hat security tracker](https://access.redhat.com/security/cve/CVE-2026-31431)
* [NVD: CVE-2026-31431](https://nvd.nist.gov/vuln/detail/CVE-2026-31431)

**Dirty Frag — CVE-2026-43284 / CVE-2026-43500**

* [NVD: CVE-2026-43284](https://nvd.nist.gov/vuln/detail/CVE-2026-43284)
* [NVD: CVE-2026-43500](https://nvd.nist.gov/vuln/detail/CVE-2026-43500)

**Fragnesia — CVE-2026-43503 / CVE-2026-46300**

* [Ubuntu CVE tracker](https://ubuntu.com/security/CVE-2026-46300)
* [NVD: CVE-2026-43503](https://nvd.nist.gov/vuln/detail/CVE-2026-43503)
* [NVD: CVE-2026-46300](https://nvd.nist.gov/vuln/detail/CVE-2026-46300)

**PEdit-CoW — CVE-2026-46331**

* [NVD: CVE-2026-46331](https://nvd.nist.gov/vuln/detail/CVE-2026-46331)

**Januscape — CVE-2026-53359**

* [NVD: CVE-2026-53359](https://nvd.nist.gov/vuln/detail/CVE-2026-53359)

**PwnKit — CVE-2021-4034**

* [Qualys advisory: PwnKit](https://blog.qualys.com/vulnerabilities-threat-research/2022/01/25/pwnkit-local-privilege-escalation-vulnerability-cve-2021-4034) — from the team that discovered it
* [PoC exploit by blasty](https://github.com/blasty/CVE-2021-4034) — small enough to read line by line
* [NVD: CVE-2021-4034](https://nvd.nist.gov/vuln/detail/CVE-2021-4034)

**OverlayFS — CVE-2023-0386**

* [NVD: CVE-2023-0386](https://nvd.nist.gov/vuln/detail/CVE-2023-0386) — also listed in the CISA KEV catalog below

**Netfilter Use-After-Free — CVE-2024-1085 / CVE-2024-1086**

* [NVD: CVE-2024-1085](https://nvd.nist.gov/vuln/detail/CVE-2024-1085)
* [NVD: CVE-2024-1086](https://nvd.nist.gov/vuln/detail/CVE-2024-1086)
* [`cfreal/ten` — CVE-2024-1086 exploitation framework](https://github.com/cfreal/ten) — excellent study material on turning an nf_tables UAF into root

**Baron Samedit — CVE-2021-3156**

* [Qualys advisory: Baron Samedit](https://blog.qualys.com/vulnerabilities-threat-research/2021/01/26/cve-2021-3156-heap-based-buffer-overflow-in-sudo-baron-samedit)
* [NVD: CVE-2021-3156](https://nvd.nist.gov/vuln/detail/CVE-2021-3156)

**eBPF Verifier Bug — CVE-2017-16995**

* [NVD: CVE-2017-16995](https://nvd.nist.gov/vuln/detail/CVE-2017-16995)

**Netfilter x_tables Heap OOB — CVE-2021-22555**

* [Google Project Zero: "An iOS hacker tries Android"](https://googleprojectzero.blogspot.com/2021/06/an-ios-hacker-tries-android.html) — Andy Nguyen's full walkthrough of the bug and its exploitation chain
* [NVD: CVE-2021-22555](https://nvd.nist.gov/vuln/detail/CVE-2021-22555)

**ptrace Exit-Race — CVE-2026-46333**

* [Red Hat security tracker](https://access.redhat.com/security/cve/CVE-2026-46333)
* [NVD: CVE-2026-46333](https://nvd.nist.gov/vuln/detail/CVE-2026-46333)

**RefluXFS — CVE-2026-64600**

* [NVD: CVE-2026-64600](https://nvd.nist.gov/vuln/detail/CVE-2026-64600)

**General resources**

* [GTFOBins](https://gtfobins.github.io/) — abuse techniques for SUID binaries and sudo misconfigurations you will actually meet in the field
* [Linux capabilities man page](https://man7.org/linux/man-pages/man7/capabilities.7.html) — every capability flag explained by the kernel docs themselves
* [CISA Known Exploited Vulnerabilities catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) — check this before trusting any CVE as "actively exploited"

If any of the NVD or vendor links above look stale when you read this, search the CVE ID directly — advisories move around more often than the identifiers do.
