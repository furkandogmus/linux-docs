# Linux Container Internals

Bu repo, Linux container teknolojilerinin temel bileşenlerini ve nasıl çalıştıklarını anlamak için bir rehber niteliğindedir.

## İçindekiler
- [Namespace'ler](#namespaceler)
  - [PID Namespace](#pid-namespace)
  - [Network Namespace](#network-namespace)
  - [Mount Namespace](#mount-namespace)
  - [IPC Namespace](#ipc-namespace)
  - [UTS Namespace](#uts-namespace)
  - [User Namespace](#user-namespace)
- [Control Groups (cgroups)](#control-groups-cgroups)
- [Örnek Uygulamalar](#örnek-uygulamalar)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

## Namespace'ler

Namespace'ler, Linux çekirdeğinin sistem kaynaklarını izole etmesini sağlayan bir özelliktir. Her namespace türü farklı bir sistem kaynağını izole eder.

### PID Namespace

Process ID'lerini izole eder. Her container kendi process ağacına sahip olur.

```bash
# Yeni PID namespace oluştur
unshare --pid --fork --mount-proc /bin/bash

# Process listesini kontrol et
ps aux
```

Kullanım Alanları:
- Container runtime'lar
- Process monitoring tools
- Sandbox environments

Dikkat Edilmesi Gerekenler:
- Init process (PID 1) yönetimi
- Zombie process'lerin temizlenmesi
- Signal handling

### Network Namespace

Network stack'i izole eder. Her container kendi network interface'lerine sahip olur.

```bash
# Network namespace oluştur
ip netns add container1

# Interface yapılandır
ip link add veth0 type veth peer name ceth0
ip link set ceth0 netns container1
```

Kullanım Alanları:
- Container networking
- Network virtualization
- Security isolation

Network Tipleri:
1. Bridge Network
```bash
# Bridge oluştur
ip link add name mybridge type bridge
ip link set mybridge up
```

2. NAT Network
```bash
# NAT ayarları
iptables -t nat -A POSTROUTING -s 172.17.0.0/16 -j MASQUERADE
```

### Mount Namespace

Filesystem mount noktalarını izole eder.

```bash
# Mount namespace oluştur
unshare --mount /bin/bash

# Filesystem mount et
mount -t proc proc /proc
```

Kullanım Alanları:
- Container filesystem isolation
- Chroot environments
- Secure application sandboxing

Best Practices:
- Read-only root filesystem
- Volume management
- Temporary filesystem (tmpfs) kullanımı

### IPC Namespace

Inter-Process Communication mekanizmalarını izole eder.

```bash
# IPC namespace oluştur
unshare --ipc /bin/bash

# IPC kaynaklarını listele
ipcs
```

IPC Mekanizmaları:
1. Shared Memory
```c
shmget(key, size, 0666 | IPC_CREAT);
```

2. Message Queues
```c
msgget(key, 0666 | IPC_CREAT);
```

3. Semaphores
```c
semget(key, 1, 0666 | IPC_CREAT);
```

### UTS Namespace

Hostname ve NIS domain name'i izole eder.

```bash
# UTS namespace oluştur
unshare --uts /bin/bash

# Hostname değiştir
hostname container1
```

### User Namespace

User ve group ID'lerini izole eder.

```bash
# User namespace oluştur
unshare --user /bin/bash

# UID/GID mapping
echo "0 1000 65536" > /proc/$$/uid_map
```

Security Considerations:
- Root privilege separation
- UID/GID mapping
- Capability management

## Control Groups (cgroups)

cgroups, container'ların sistem kaynak kullanımını sınırlandırır ve izler.

### CPU Limitleri

```bash
# CPU kullanımını %50 ile sınırla
echo "50000 100000" > /sys/fs/cgroup/mycontainer/cpu.max

# CPU core'larını sınırla
echo "0-1" > /sys/fs/cgroup/mycontainer/cpuset.cpus
```

### Memory Limitleri

```bash
# RAM limiti (512MB)
echo "536870912" > /sys/fs/cgroup/mycontainer/memory.max

# Swap limiti
echo "0" > /sys/fs/cgroup/mycontainer/memory.swap.max
```

### I/O Limitleri

```bash
# Disk I/O limiti (10MB/s)
echo "10485760" > /sys/fs/cgroup/mycontainer/io.max
```

### Process Limitleri

```bash
# Maximum process sayısı
echo "100" > /sys/fs/cgroup/mycontainer/pids.max
```

## Örnek Uygulamalar

### 1. Basit Container İmplementasyonu

```bash
#!/bin/bash
unshare --pid --fork --mount-proc \
        --mount --uts --ipc --net \
        chroot container_root /bin/bash
```

### 2. Network İzolasyonu ve Bridge Network

```bash
# Bridge oluştur
ip link add name mybridge type bridge
ip link set mybridge up
ip addr add 172.17.0.1/16 dev mybridge

# Container network ayarları
ip link add veth0 type veth peer name ceth0
ip link set ceth0 netns container1
ip netns exec container1 ip addr add 172.17.0.2/16 dev ceth0
```

### 3. Resource Limitleri

```bash
# cgroup oluştur ve limitle
mkdir -p /sys/fs/cgroup/myapp
echo "512M" > /sys/fs/cgroup/myapp/memory.max
echo "50000 100000" > /sys/fs/cgroup/myapp/cpu.max
```

## Best Practices

1. Security
- User namespace kullan
- Root filesystem'i read-only yap
- Capability'leri sınırla
- Seccomp profilleri kullan

2. Networking
- Bridge networking tercih et
- Port range'leri sınırla
- Network izolasyonunu doğru yapılandır
- DNS ayarlarını düzgün yap

3. Resource Management
- Her container için resource limitleri belirle
- Monitoring sistemi kur
- OOM killer ayarlarını yapılandır
- CPU pinning kullan

4. Storage
- Volume mounting güvenliğine dikkat et
- tmpfs kullan
- Disk I/O limitleri koy
- Logging stratejisi belirle

## Troubleshooting

### Namespace Issues

1. PID Namespace
```bash
# Process'leri kontrol et
pstree -p
# Namespace'leri listele
lsns -t pid
```

2. Network Namespace
```bash
# Network interface'leri kontrol et
ip netns exec container1 ip addr
# Routing table'ı kontrol et
ip netns exec container1 route -n
```

3. Mount Namespace
```bash
# Mount pointleri kontrol et
findmnt
# Namespace mount'ları listele
cat /proc/self/mountinfo
```

### cgroups Issues

1. Resource Usage
```bash
# Memory kullanımı
cat /sys/fs/cgroup/myapp/memory.current
# CPU stats
cat /sys/fs/cgroup/myapp/cpu.stat
```

2. Limits
```bash
# Limitleri kontrol et
cat /sys/fs/cgroup/myapp/memory.max
cat /sys/fs/cgroup/myapp/cpu.max
```

## Kaynaklar

- [Linux Kernel Documentation](https://www.kernel.org/doc/html/latest/)
- [Docker Documentation](https://docs.docker.com)
- [Control Groups Documentation](https://www.kernel.org/doc/Documentation/cgroup-v2.txt)


