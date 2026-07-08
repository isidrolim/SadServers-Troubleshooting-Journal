# Venice: Am I in a Container?

## Scenario

The SadServers lab **"Venice"** asks whether the current environment is a container or a virtual machine.

The goal is not to fix a broken service, but to investigate the runtime environment and prove the answer with evidence.

The lab is intentionally tricky because the environment looks like a normal virtual machine at first.

## Requirement

Determine whether the SadServers instance is running inside:

```text
A container
```

or:

```text
A virtual machine
```

## Initial State

The system appeared to be a Debian 11 environment.

Initial checks made it look like a normal VM because:

```text
PID 1 was systemd
/proc/1/cgroup did not show obvious Docker markers
No /.dockerenv file was present
systemd-detect-virt returned kvm
```

This made the first conclusion look like:

```text
Virtual Machine
```

However, the SadServers solution revealed that the environment was actually a Podman container.

## Troubleshooting Path

```text
Check operating system information
↓
Check PID 1 process
↓
Check cgroup information
↓
Check common container marker files
↓
Run virtualization detection
↓
Check PID 1 environment
↓
Look for kernel threads
↓
Compare evidence
↓
Final conclusion
```

## Verification Before Conclusion

Check OS release:

```bash
cat /etc/os-release
```

Result showed:

```text
Debian GNU/Linux 11 (bullseye)
```

This only proves the userspace distribution.

It does not prove whether the environment is a container or a VM.

Check PID 1:

```bash
ps -p 1 -o pid,ppid,comm,args
```

Result:

```text
PID  PPID  COMMAND  COMMAND
1    0     systemd  /sbin/init
```

This initially suggested a VM because normal full Linux systems often run `systemd` as PID 1.

Check cgroup information:

```bash
cat /proc/1/cgroup
```

Result:

```text
0::/init.scope
```

This did not show obvious container markers such as:

```text
docker
containerd
kubepods
lxc
libpod
```

Check common container marker files:

```bash
ls -la /.dockerenv /run/.containerenv 2>/dev/null
```

Result:

```text
No output
```

This means common Docker or Podman marker files were not present.

Check virtualization detection:

```bash
systemd-detect-virt
```

Result:

```text
kvm
```

Check container detection:

```bash
systemd-detect-virt --container
```

Result:

```text
none
```

Check VM detection:

```bash
systemd-detect-virt --vm
```

Result:

```text
kvm
```

At this point, the evidence appeared to point toward a VM.

## First Finding

The first set of checks was misleading.

The environment looked like a VM because:

```text
PID 1 was systemd
systemd-detect-virt reported kvm
No obvious container marker files existed
```

But this was not enough evidence.

The lab was designed to be tricky.

## Additional Investigation

The official clue suggested checking the environment of PID 1:

```bash
cat /proc/1/environ | tr '\0' '\n' | grep -i container
```

In a typical Podman container, this may show a variable such as:

```text
container=podman
```

However, in this lab, the variable value was changed to make detection less obvious.

Another important clue was to check running processes and look for kernel threads.

Check for kernel threads:

```bash
ps -ef | grep -E 'kthreadd|kworker|ksoftirqd|rcu|migration'
```

On a normal VM, kernel threads such as these are usually visible:

```text
kthreadd
kworker
ksoftirqd
rcu_sched
migration
```

In a container, these host kernel threads are usually not visible inside the container's PID namespace.

This absence is a strong clue that the environment is a container.

## First Failure in My Initial Reasoning

The mistake was trusting `systemd-detect-virt` and PID 1 too much.

The first conclusion was:

```text
This is a VM.
```

But the better conclusion required correlating more signals.

The important missing check was:

```text
Are kernel threads visible?
```

## Final Conclusion

The environment is actually:

```text
A Podman container
```

not a virtual machine.

## Why This Was Tricky

This container was made to look like a VM.

It had:

```text
systemd as PID 1
No obvious /.dockerenv marker
No obvious cgroup container name
Misleading virtualization detection output
Changed container environment variable
```

Because of that, common quick checks were not enough.

## Correct Reasoning

A better investigation uses multiple independent signals:

```text
PID 1 process
↓
cgroup data
↓
container marker files
↓
systemd-detect-virt
↓
PID 1 environment
↓
kernel thread visibility
↓
process namespace behavior
```

If the system looks like a VM but does not show normal kernel threads, that is an important container clue.

## Validation

There is no automated validation for this SadServers scenario.

The final answer is based on evidence and the SadServers solution:

```text
This instance is a Podman container.
```

## Lessons Learned

- Containers can be configured to look like virtual machines.
- `systemd` as PID 1 does not always prove the system is a VM.
- `systemd-detect-virt` can be misleading in tricky or modified environments.
- The absence of `/.dockerenv` does not prove the system is not a container.
- `/proc/1/environ` can reveal container-related environment variables.
- Kernel threads such as `kthreadd` and `kworker` are normally visible on real VMs, but usually not inside containers.
- Do not rely on only one signal when identifying the runtime environment.
- Use multiple independent checks before making a conclusion.

## Knowledge Check

### Question 1

Why was the initial conclusion of "VM" misleading?

A. Because Docker was not installed  
B. Because PID 1 was `systemd` and `systemd-detect-virt` returned `kvm`  
C. Because `/etc/os-release` showed Debian  
D. Because the hostname was short  

**Answer:** B

### Question 2

Which check can help reveal that an environment is actually a container even when it looks like a VM?

A. Checking for missing kernel threads such as `kthreadd` and `kworker`  
B. Running `pwd`  
C. Checking the current date  
D. Listing `/tmp`  

**Answer:** A

### Question 3

Which command checks the environment variables of PID 1 for container-related hints?

A. `cat /etc/passwd`  
B. `cat /proc/1/environ | tr '\0' '\n' | grep -i container`  
C. `docker ps`  
D. `hostnamectl set-hostname container`  

**Answer:** B
