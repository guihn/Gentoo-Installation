# Guia de Instalação do Gentoo Linux
### Arch Linux → Gentoo — AMD Ryzen 9 9900X + GTX 750 Ti + Radeon 760M

---

> **Filosofia do guia:** cada passo vem acompanhado do *porquê* e do *como funciona internamente*. Nenhuma decisão é tomada por você — alternativas são apresentadas com trade-offs explícitos.

---

## Sumário

1. [Conceitos fundamentais do Gentoo](#1-conceitos-fundamentais-do-gentoo)
2. [Preparação do ambiente live](#2-preparação-do-ambiente-live)
3. [Particionamento](#3-particionamento)
4. [Stage3 — a semente do sistema](#4-stage3--a-semente-do-sistema)
5. [Portage — o sistema de pacotes](#5-portage--o-sistema-de-pacotes)
6. [make.conf — configuração central](#6-makeconf--configuração-central)
7. [Compilação do kernel](#7-compilação-do-kernel)
8. [Bootloader](#8-bootloader)
9. [Sistema base pós-boot](#9-sistema-base-pós-boot)
10. [Drivers de GPU — situação dual-GPU](#10-drivers-de-gpu--situação-dual-gpu)
11. [Wayland + Hyprland](#11-wayland--hyprland)
12. [Periféricos de alta taxa de polling](#12-periféricos-de-alta-taxa-de-polling)
13. [Otimizações pós-instalação](#13-otimizações-pós-instalação)
14. [Referência rápida](#14-referência-rápida)

---

## 1. Conceitos Fundamentais do Gentoo

### Portage e ebuilds

O Portage é o gerenciador de pacotes do Gentoo. Diferente do pacman (binários pré-compilados), o Portage por padrão **baixa o código-fonte e compila na sua máquina**. O motivo é simples: compilação local permite passar flags de compilação específicas para o seu CPU, habilitando instruções que o mantenedor do pacote não pode assumir como disponíveis universalmente (AVX2, AVX-512, etc.).

Um **ebuild** é um script shell com metadados sobre como baixar, compilar e instalar um pacote. O equivalente ao PKGBUILD do Arch.

O repositório principal (`::gentoo`) fica em `/var/db/repos/gentoo/`.

### USE flags

USE flags são o mecanismo central de controle do Gentoo. Elas definem **quais features são compiladas** em cada pacote. Exemplos:

- `USE="pulseaudio"` → compila suporte a PulseAudio dentro dos pacotes que o suportam.
- `USE="-bluetooth"` → remove suporte a Bluetooth de todos os pacotes que o incluiriam por padrão.

Isso reduz dependências, tamanho de binários e superfície de ataque. Um pacote compilado sem uma feature não carrega aquele código em memória.

### ACCEPT_KEYWORDS e estabilidade

O Gentoo possui dois níveis de estabilidade por arquitetura:

| Keyword | Significado |
|---|---|
| `amd64` | Estável — testado pelos maintainers |
| `~amd64` | Testing — funciona, mas menos testado |

Para um desktop moderno com hardware recente (Zen 5, Wayland, Hyprland), você vai precisar de pacotes `~amd64`. Isso não significa instável no sentido de quebrado — significa menos garantias formais. O Arch testing é comparável.

### Slots

Pacotes no Gentoo podem coexistir em **slots** diferentes. Exemplo: Python 3.11 e 3.13 podem ser instalados simultaneamente. O gerenciamento é feito automaticamente pelo Portage.

---

## 2. Preparação do Ambiente Live

### ISO recomendada

Use o **Gentoo minimal installation CD** (x86_64). Não use live DVDs cheios de software que você não precisa.

Download: `https://www.gentoo.org/downloads/`

Verifique a integridade:

```bash
# Baixe o .iso e o .iso.sha256sum
sha256sum --check install-amd64-minimal-*.iso.sha256sum
```

Grave em pendrive:

```bash
dd if=install-amd64-minimal-*.iso of=/dev/sdX bs=4M status=progress conv=fsync
```

### Boot

Na BIOS/UEFI da Asus TUF X670-E Plus:
- Confirme que **Secure Boot está desativado** (o kernel Gentoo customizado não terá assinatura).
- Boot mode: **UEFI**.
- Desative **CSM** se ativo.

### Rede no ambiente live

O live CD usa `net-setup` ou configuração manual. Sua placa-mãe não tem WiFi, então é Ethernet diretamente:

```bash
# Identifica a interface
ip link

# Se não tiver endereço DHCP automático:
dhcpcd <interface>  # ex: dhcpcd enp4s0

# Testa
ping -c 3 gentoo.org
```

### SSH opcional mas recomendado

Instalar do seu Arch via SSH no live CD é mais confortável — você pode copiar/colar comandos longos:

```bash
# No live CD, defina senha root:
passwd

# Inicie o sshd:
rc-service sshd start

# Descubra o IP:
ip addr

# Do Arch, conecte:
ssh root@<ip-do-live>
```

---

## 3. Particionamento

Seu layout atual no Arch: EFI (1 GB) + swap (16 GB) + raiz unificada. Você pode manter exatamente esse esquema.

### Identificando os discos

```bash
lsblk
# ou
fdisk -l
```

Seu NVMe provavelmente aparece como `/dev/nvme0n1`. As partições como `nvme0n1p1`, `nvme0n1p2`, `nvme0n1p3`.

### Opção A — Reusar as partições existentes (wipando apenas root)

Se quiser preservar a EFI e o swap, formate apenas a partição raiz:

```bash
# Formate root como ext4 (padrão sólido) ou btrfs (se quiser snapshots)
mkfs.ext4 /dev/nvme0n1p3

# A EFI já existe (FAT32), não reformate
# O swap já existe:
swapon /dev/nvme0n1p2
```

### Opção B — Recriar tudo do zero

```bash
gdisk /dev/nvme0n1
```

Estrutura:
| Partição | Tipo | Tamanho | Código gdisk |
|---|---|---|---|
| p1 EFI | EFI System | 1 GiB | ef00 |
| p2 swap | Linux swap | 16 GiB | 8200 |
| p3 root | Linux filesystem | restante | 8300 |

```bash
# Formate
mkfs.vfat -F32 /dev/nvme0n1p1
mkswap /dev/nvme0n1p2
mkfs.ext4 /dev/nvme0n1p3

swapon /dev/nvme0n1p2
```

### Sobre ext4 vs btrfs

| | ext4 | btrfs |
|---|---|---|
| Maturidade | Muito madura | Madura para uso desktop |
| Snapshots | Não | Sim (via subvolumes) |
| Compressão transparente | Não | Sim (zstd) |
| Performance em NVMe | Excelente | Comparável, overhead em writes |
| Complexidade | Simples | Adiciona abstrações |

Para sua filosofia de controle e simplicidade sem abstrações desnecessárias: **ext4 é a escolha mais limpa**. Btrfs faz sentido se você quer snapshots antes de atualizações do sistema — mas isso é uma abstração sobre o que seria `rsync` ou partições separadas.

### Montagem

```bash
mount /dev/nvme0n1p3 /mnt/gentoo
mkdir -p /mnt/gentoo/efi
mount /dev/nvme0n1p1 /mnt/gentoo/efi
```

---

## 4. Stage3 — A Semente do Sistema

O **stage3** é um tarball com o sistema base mínimo: libc (glibc ou musl), coreutils, shell, Portage. É o ponto de partida de toda instalação Gentoo — não existe installer no sentido tradicional.

### Escolhendo o stage3

Acesse `https://www.gentoo.org/downloads/` ou use um mirror:

```bash
cd /mnt/gentoo
links https://mirrors.kernel.org/gentoo/releases/amd64/autobuilds/current-stage3-amd64-systemd/
```

Variantes relevantes:

| Variante | Quando usar |
|---|---|
| `stage3-amd64-openrc` | OpenRC como init — simples, scripts shell, sem dependências extras |
| `stage3-amd64-systemd` | systemd — familiar se vem do Arch, mais integração com alguns softwares |
| `stage3-amd64-musl` | musl libc — menor, mais rígido, alguns softwares não compilam |

**Recomendação técnica:** OpenRC é nativo do Gentoo, mais simples de entender internamente, e é um script shell — você pode ler e entender cada init script sem uma camada de DSL. systemd é o que você usa no Arch, logo já conhece.

**A decisão é sua.** Este guia usa **OpenRC** por ser mais alinhado com sua filosofia de entendimento sobre automação.

```bash
# Exemplo com OpenRC:
wget https://mirrors.kernel.org/gentoo/releases/amd64/autobuilds/current-stage3-amd64-openrc/stage3-amd64-openrc-<data>.tar.xz
wget https://mirrors.kernel.org/gentoo/releases/amd64/autobuilds/current-stage3-amd64-openrc/stage3-amd64-openrc-<data>.tar.xz.asc

# Verificação criptográfica:
gpg --import /usr/share/openpgp-keys/gentoo-release.asc
gpg --verify stage3-amd64-openrc-*.tar.xz.asc
```

### Extração

```bash
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner -C /mnt/gentoo
```

Flags importantes:
- `--xattrs-include='*.*'` → preserva atributos estendidos (capabilities, SELinux labels se usados).
- `--numeric-owner` → preserva UIDs/GIDs numéricos ao invés de nomes — evita mapeamentos incorretos se o live CD tiver usuários diferentes.
- `-C /mnt/gentoo` → extrai no destino correto.

---

## 5. Portage — O Sistema de Pacotes

### Estrutura interna

```
/var/db/repos/gentoo/     ← repositório de ebuilds (texto, ~700 MB)
/var/db/pkg/              ← banco de dados de pacotes instalados
/var/cache/distfiles/     ← tarballs de código-fonte baixados
/var/cache/binpkgs/       ← pacotes binários locais (se usar FEATURES=buildpkg)
/etc/portage/             ← configuração do usuário
```

O Portage resolve dependências, baixa fontes, aplica patches, compila e instala. O DAG de dependências é calculado a partir dos ebuilds.

### Sincronização do repositório

```bash
# Dentro do chroot (próxima seção), após instalar o Portage:
emerge --sync
# ou com git (mais rápido, apenas deltas):
# (configurado via /etc/portage/repos.conf)
```

---

## 6. make.conf — Configuração Central

Este é o arquivo mais importante da instalação. Fica em `/etc/portage/make.conf`.

### Chroot primeiro

```bash
# Copie resolv.conf para ter DNS dentro do chroot:
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/

# Monte os filesystems virtuais:
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run

# Entre no chroot:
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"

# Monte a EFI:
mount /dev/nvme0n1p1 /efi
```

### make.conf para seu hardware

```bash
nano /etc/portage/make.conf
```

```make
# =============================================
# CFLAGS — flags de compilação C
# =============================================
# -O2: otimização padrão. Seguro e agressivo o suficiente.
# -O3: pode gerar código mais rápido em loops numéricos, mas aumenta tamanho
#      de binários e raramente beneficia software de sistema. Benchmarke antes de usar.
# -march=znver5: gera instruções específicas para Zen 5 (Ryzen 9000).
#   Alternativa: -march=native (detecta em tempo de compilação, não portável).
#   znver5 é explícito e documentado. Use znver5 para Ryzen 9000.
# -pipe: usa pipes ao invés de arquivos temporários entre estágios do compilador.
#   Mais RAM, menos I/O. Com 32 GB, irrelevante — ative sem pensar.
# -fomit-frame-pointer: remove o frame pointer do stack, liberando um registrador.
#   Padrão em -O2. Debug fica levemente mais difícil.

CFLAGS="-march=znver5 -O2 -pipe"
CXXFLAGS="${CFLAGS}"
FCFLAGS="${CFLAGS}"
FFLAGS="${CFLAGS}"

# =============================================
# MAKEOPTS — paralelismo de compilação
# =============================================
# Ryzen 9 9900X: 12 cores / 24 threads.
# Regra geral: -jN onde N = threads. Deixe 1-2 livres se quiser usar o sistema
# durante compilações pesadas. Com 32 GB de RAM, memória não é gargalo.
# --load-average=20: não dispara novos jobs se a carga já estiver acima de 20.

MAKEOPTS="-j22 --load-average=20"

# =============================================
# PORTAGE_NICENESS — prioridade de compilação
# =============================================
# 19 = prioridade mínima. O compilador não compete com seu trabalho interativo.
PORTAGE_NICENESS=19

# =============================================
# USE flags globais
# =============================================
# Configure conforme suas necessidades. Exemplos comentados:

USE="
  -bluetooth        # desative se não usar Bluetooth
  -modemmanager     # sem modem 3G/4G
  -policykit        # desative se quiser controle manual (alguns SDs dependem dele)
  wayland           # habilita suporte Wayland nos pacotes que o têm
  -X                # desative X11 se for 100% Wayland (cuidado: alguns pacotes exigem)
  vulkan            # para GPU/jogos
  pipewire          # áudio moderno
  -pulseaudio       # pipewire substitui
  nvptx             # NVIDIA compute (opcional)
  opencl            # compute GPU (AMD/NVIDIA)
"

# =============================================
# VIDEO_CARDS — drivers de GPU
# =============================================
# amdgpu: driver open source AMD (iGPU Radeon 760M — RDNA2)
# nvidia: driver proprietário NVIDIA (GTX 750 Ti — Maxwell)
# Ambos são necessários para sua configuração dual-GPU.

VIDEO_CARDS="amdgpu nvidia"

# =============================================
# INPUT_DEVICES
# =============================================
INPUT_DEVICES="libinput"

# =============================================
# ACCEPT_KEYWORDS
# =============================================
# ~amd64 para hardware/software moderno. Sem isso, Hyprland e pacotes Wayland
# recentes ficam indisponíveis ou em versões antigas.
ACCEPT_KEYWORDS="~amd64"

# =============================================
# ACCEPT_LICENSE
# =============================================
# @FREE: apenas software livre. Adicione licenças específicas conforme necessário.
# Para NVIDIA proprietário:
ACCEPT_LICENSE="@FREE @BINARY-REDISTRIBUTABLE"

# =============================================
# FEATURES
# =============================================
# parallel-fetch: baixa próximo tarball enquanto compila o atual.
# buildpkg: cria pacotes binários locais. Útil para reinstalações ou múltiplas máquinas.
# ccache: cache de compilação. Reinstalações de pacotes iguais ficam rápidas.
#   Requer: emerge dev-util/ccache
FEATURES="parallel-fetch buildpkg"

# =============================================
# GRUB_PLATFORMS
# =============================================
GRUB_PLATFORMS="efi-64"

# =============================================
# L10N — localização
# =============================================
L10N="pt-BR pt en"
LINGUAS="pt_BR pt en"
```

### Por que não `-O3` por padrão?

`-O3` habilita otimizações como unroll de loops, vetorização agressiva e inlining extensivo. Em benchmarks sintéticos melhora. Em software real:

- Binários maiores → mais cache misses → pode ser *mais lento*.
- Alguns softwares têm UB (Undefined Behavior) que `-O3` expõe onde `-O2` mascara.
- GCC/Clang com `-O3` ocasionalmente gera código incorreto em edge cases.

Se quiser explorar: compile pacotes específicos com `-O3` via `/etc/portage/package.env`, meça, compare. Não aplique globalmente sem evidência.

---

## 7. Compilação do Kernel

Esta é a etapa mais importante para controle real do sistema. Você vai configurar e compilar seu próprio kernel.

### Instalando o código-fonte

```bash
emerge --ask sys-kernel/gentoo-sources
# Instala em /usr/src/linux-<versão>/

# Cria symlink /usr/src/linux:
eselect kernel list
eselect kernel set 1
```

### genkernel vs configuração manual

| | Manual (`make menuconfig`) | genkernel |
|---|---|---|
| Controle | Total | Limitado |
| Tempo inicial | Alto (horas para aprender) | Baixo |
| Kernel resultante | Mínimo e específico | Genérico, maior |
| Aprendizado | Profundo | Superficial |

Para sua filosofia: **configuração manual**. O genkernel produz um kernel genérico comparável ao da maioria das distros — negando o principal benefício do Gentoo.

### Configuração

```bash
cd /usr/src/linux

# Ponto de partida: configuração do kernel atual do Arch (se acessível):
# zcat /proc/config.gz > .config  (no sistema Arch antes de migrar)
# Ou use a configuração padrão do gentoo-sources:
make defconfig

# Interface de configuração:
make menuconfig
```

#### Itens críticos para seu hardware

**CPU:**
```
Processor type and features →
  Processor family → Opteron/Athlon64/Hammer/K8  [ou Family 17h/19h para Zen]
  Multi-core scheduler support → Y
  SMT (Hyperthreading) scheduler support → Y

General setup →
  Preemption Model → Voluntary ou Full (Full = menor latência, útil para desktop)
```

**AMD iGPU (Radeon 760M — RDNA2):**
```
Device Drivers →
  Graphics support →
    Direct Rendering Manager (DRM) → Y (built-in, não módulo)
      AMD GPU → M (módulo) ou Y
        Enable amdgpu support for SI parts → N (GCN 1, não relevante)
        Enable amdgpu support for CIK parts → N (GCN 2, não relevante)
        [RDNA2 é suportado pelo driver principal amdgpu]
      Enable HSA kernel driver for AMD GPU → Y (compute)
```

**NVIDIA GTX 750 Ti (Maxwell):**
```
# O driver proprietário da NVIDIA substitui o driver open-source Nouveau.
# Nouveau deve ser desabilitado ou colocado em lista negra.

Device Drivers →
  Graphics support →
    Nouveau (NVIDIA) → N  ← desative ou deixe como módulo para blacklist
```

O driver proprietário NVIDIA fornece seu próprio módulo de kernel (`nvidia.ko`), compilado via `emerge x11-drivers/nvidia-drivers`. Ele usa o framework DKMS ou é recompilado a cada atualização do kernel pelo Portage.

**NVMe:**
```
Device Drivers →
  NVME Support →
    NVM Express block device → Y (built-in, você precisa disso para bootar)
```

**Filesystem:**
```
File systems →
  The Extended 4 (ext4) filesystem → Y (built-in se for sua root)
  EFI Variable filesystem → Y
```

**Wayland/DRM:**
```
Device Drivers →
  Graphics support →
    DRM → Y
    DRM KMS helper → Y
    Framebuffer Console → Y (opcional, para TTY)
```

**Input de alta polling rate (mouse/keyboard 8000 Hz):**
```
Device Drivers →
  HID support →
    USB HID transport layer → Y ou M
    Generic HID driver → Y
  USB support →
    USB device filesystem → Y
    xHCI HCD (USB 3.0) support → Y
```

> Nota: 8000 Hz polling via USB precisa de suporte a `usbhid` com parametrização. Detalhe na seção 12.

**Rede (placa Ethernet da Asus TUF X670-E):**

A Asus TUF X670-E usa um Intel I225-V ou Realtek chip. Verifique:

```bash
# No Arch, antes de migrar:
lspci | grep -i ethernet
```

Habilite o driver correspondente no kernel:
```
Device Drivers →
  Network device support →
    Ethernet driver support →
      Intel → Intel(R) Ethernet Controller I225-LM/I225-V  [se Intel]
      Realtek → Realtek 8169 gigabit ethernet  [se Realtek]
```

### Compilação

```bash
# Número de jobs = threads do CPU:
make -j22

# Instala módulos em /lib/modules/<versão>/:
make modules_install

# Instala vmlinuz e System.map em /boot/:
make install
```

Tempo estimado no Ryzen 9 9900X com defconfig: ~3-5 minutos. Com configuração completa: ~5-10 minutos.

### initramfs

Você precisa de um initramfs se o driver do seu filesystem raiz estiver como módulo (não built-in). Com ext4 como built-in no kernel, é possível bootar sem initramfs — vantagem de configurar o kernel manualmente.

Se precisar de initramfs:

```bash
emerge --ask sys-kernel/dracut
# ou
emerge --ask sys-kernel/genkernel  # apenas para gerar o initramfs, não o kernel

dracut --force --kver $(make -s kernelrelease)
```

---

## 8. Bootloader

### GRUB2 (opção padrão, funcional e bem documentado)

```bash
emerge --ask sys-boot/grub

# Instala o GRUB na partição EFI:
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=Gentoo

# Gera grub.cfg:
grub-mkconfig -o /boot/grub/grub.cfg
```

### systemd-boot (opção mais leve)

Se preferir algo sem camada extra de abstração e com configuração em texto plano:

```bash
# systemd-boot já está disponível se você usar systemd.
# Com OpenRC, instale manualmente:
bootctl install

# Crie /efi/loader/loader.conf:
cat > /efi/loader/loader.conf << 'EOF'
default gentoo.conf
timeout 3
editor no
EOF

# Crie /efi/loader/entries/gentoo.conf:
cat > /efi/loader/entries/gentoo.conf << 'EOF'
title   Gentoo Linux
linux   /vmlinuz-<versão>
initrd  /initramfs-<versão>.img  # se tiver initramfs
options root=/dev/nvme0n1p3 rw quiet
EOF
```

**Trade-off:** GRUB suporta mais casos (multi-boot, LVM, LUKS), detecta kernels automaticamente, e tem mais documentação. systemd-boot é mais simples mas requer configuração manual a cada atualização do kernel (automatizável via hooks).

### Kernel parameters relevantes

```
# Para configuração dual-GPU com Wayland:
root=/dev/nvme0n1p3 rw
amdgpu.runpm=0        # evita problemas de power management no iGPU
nvidia-drm.modeset=1  # necessário para Wayland com NVIDIA
```

---

## 9. Sistema Base Pós-Boot

### Configurações antes do primeiro boot

```bash
# fstab:
# Use UUIDs, não nomes de dispositivo (mais estável):
blkid  # anote os UUIDs

cat > /etc/fstab << 'EOF'
UUID=<uuid-efi>   /efi   vfat   defaults,noatime   0 2
UUID=<uuid-swap>  none   swap   sw                 0 0
UUID=<uuid-root>  /      ext4   defaults,noatime   0 1
EOF

# Hostname:
echo "gentoo" > /etc/hostname

# Hosts:
cat > /etc/hosts << 'EOF'
127.0.0.1   localhost
::1         localhost
127.0.1.1   gentoo.localdomain gentoo
EOF

# Timezone:
ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime

# Locale:
cat > /etc/locale.gen << 'EOF'
en_US.UTF-8 UTF-8
pt_BR.UTF-8 UTF-8
EOF
locale-gen

# Selecione o locale:
eselect locale list
eselect locale set <número do pt_BR.UTF-8>
env-update && source /etc/profile

# Senha root:
passwd
```

### Rede com OpenRC

```bash
emerge --ask net-misc/dhcpcd

# Edite /etc/conf.d/net:
cat > /etc/conf.d/net << 'EOF'
config_enp4s0="dhcp"
EOF

# Crie symlink para iniciar na boot:
ln -s /etc/init.d/net.lo /etc/init.d/net.enp4s0
rc-update add net.enp4s0 default

rc-update add dhcpcd default
```

### Usuário não-root

```bash
useradd -m -G wheel,audio,video,usb,input -s /bin/bash <usuario>
passwd <usuario>

# sudo:
emerge --ask app-admin/sudo
visudo  # descomente: %wheel ALL=(ALL:ALL) ALL
```

---

## 10. Drivers de GPU — Situação Dual-GPU

Sua configuração é incomum: dois GPUs de fabricantes diferentes, conectados a monitores diferentes.

### AMD (Radeon 760M — iGPU Zen 5)

O driver `amdgpu` é open-source e parte do kernel mainline (DRM). Para firmware:

```bash
emerge --ask sys-kernel/linux-firmware
# O firmware da Radeon 760M (RDNA2/gfx1103) está em linux-firmware.
```

O Radeon 760M usa arquitetura **GFX11** (gfx1103). O firmware necessário: `amdgpu/gc_11_0_3_*`, `amdgpu/sdma_6_0_3_*`, etc. Tudo incluído no `linux-firmware`.

### NVIDIA (GTX 750 Ti — Maxwell GM107)

A GTX 750 Ti (Maxwell) é suportada pelo driver proprietário atual da NVIDIA. Instale:

```bash
emerge --ask x11-drivers/nvidia-drivers
```

O Portage configura automaticamente o DKMS-like rebuild quando o kernel muda (via `linux-mod-rebuild`).

**Blacklist do Nouveau:**

```bash
cat > /etc/modprobe.d/blacklist-nouveau.conf << 'EOF'
blacklist nouveau
options nouveau modeset=0
EOF
```

### Configuração Wayland dual-GPU

Com Hyprland, você pode:

**Opção A — Monitor NVIDIA renderizado pela NVIDIA, monitor AMD pela AMD:**
Hyprland suporta múltiplos backends. Com `WLR_DRM_DEVICES` você controla qual GPU renderiza qual output:

```bash
# Na sessão Hyprland (environment variables):
WLR_DRM_DEVICES=/dev/dri/card1:/dev/dri/card0
# A ordem importa — teste qual card corresponde a qual GPU:
ls -la /dev/dri/
```

**Opção B — iGPU AMD renderiza tudo (offload NVIDIA):**
Possível mas complexo. A NVIDIA GTX 750 Ti não suporta bem reverse PRIME no Wayland sem configuração cuidadosa.

**Opção C — Cada monitor roda no GPU conectado:**
Esta é a opção mais natural para seu setup. Hyprland detecta automaticamente ambos os DRM devices. Verifique com:

```bash
hyprctl monitors
```

---

## 11. Wayland + Hyprland

### Dependências base

```bash
# Bibliotecas Wayland:
emerge --ask dev-libs/wayland dev-libs/wayland-protocols

# wlroots (compositor library base do Hyprland):
emerge --ask gui-libs/wlroots

# Hyprland:
emerge --ask gui-wm/hyprland
```

> **Nota:** Hyprland e suas dependências exigem `~amd64`. Se você definiu `ACCEPT_KEYWORDS="~amd64"` globalmente no `make.conf`, isso já está coberto.

### Configuração inicial

```bash
mkdir -p ~/.config/hypr
# Copie sua config do Arch diretamente — ela é portável.
```

### Wayland-specific para NVIDIA

A GTX 750 Ti com driver proprietário precisa de:

```bash
# /etc/environment ou no script de início da sessão:
WLR_NO_HARDWARE_CURSORS=1   # cursors de hardware falham com NVIDIA no wlroots
LIBVA_DRIVER_NAME=nvidia    # VA-API via NVIDIA (se usar)
GBM_BACKEND=nvidia-drm      # GBM backend
__GLX_VENDOR_LIBRARY_NAME=nvidia
```

### Seat e permissões

Com OpenRC, use `seatd` (leve, sem dependências systemd) ou `elogind`:

```bash
# seatd — mais leve, sem PAM/systemd
emerge --ask sys-auth/seatd
rc-update add seatd default
usermod -aG seat <usuario>

# OU elogind — mais compatível com software que espera logind API:
emerge --ask sys-auth/elogind
rc-update add elogind default
```

Hyprland funciona com ambos via `libseat`.

---

## 12. Periféricos de Alta Taxa de Polling

Seu mouse (ATK Blazing Sky Z1 Ultra, 8000 Hz) e teclado (MonsGeek FUN60 Ultra, 8000 Hz) operam em polling rates que o kernel Linux trata de forma especial.

### O problema

O subsistema `usbhid` por padrão usa a polling rate reportada pelo dispositivo, mas o scheduler de interrupções do kernel pode throttlear em rates muito altas se não configurado.

Verifique a polling rate atual:

```bash
# Instale evtest ou use diretamente:
cat /sys/bus/usb/devices/<id>/bcdDevice
# Ou:
sudo evhz  # (pacote separate)
```

### Parâmetro do kernel usbhid

```
# Em /etc/modprobe.d/usbhid.conf:
options usbhid mousepoll=1   # força 1ms polling interval (1000 Hz via usbhid)
# Para 8000 Hz: o dispositivo precisa de USB 2.0 High Speed e driver que suporte
# microframe polling (intervalo de 0.125ms = 8000 Hz)
```

> **Detalhe técnico:** USB Full Speed (12 Mbps) suporta no máximo 1000 Hz. USB High Speed (480 Mbps) suporta até 8000 Hz com microframes. Seu receptor 2.4 GHz provavelmente opera em High Speed — verifique com `lsusb -v | grep bInterval`.

### IRQ affinity para USB

Para polling de alta frequência sem latência:

```bash
# Descubra o IRQ do controlador USB que seu mouse está conectado:
cat /proc/interrupts | grep xhci

# Fixe o IRQ em um core específico (ex: core 11, evitando cores com outros IRQs pesados):
echo 800 > /proc/irq/<número>/smp_affinity  # core 11 em hex = 800
```

Isso evita que a interrupção do mouse migre entre cores, reduzindo latência de input.

---

## 13. Otimizações Pós-Instalação

### LTO (Link-Time Optimization)

LTO permite que o compilador otimize *entre* arquivos de objeto — visibilidade global do programa em vez de por unidade de compilação. Pode melhorar performance em 5-15% em alguns workloads:

```bash
# Em make.conf, adicione nas CFLAGS:
CFLAGS="-march=znver5 -O2 -pipe -flto=thin"
# thin LTO: paralelizável, menos memória que fat LTO, ganhos comparáveis.

# Requer que o linker suporte LTO:
emerge --ask sys-devel/llvm  # para lld, ou use gold
```

> **Aviso:** LTO aumenta tempo de compilação e pode quebrar pacotes com UB. Aplique incrementalmente, monitore.

### PGO (Profile-Guided Optimization)

Compilação em duas fases: primeiro gera código instrumentado, executa workload real, depois recompila usando os dados de profile. Melhoras reais de 10-20% em software como Firefox, compiladores.

Complexidade alta — pesquise por pacote específico quando quiser extrair máximo de um binário crítico.

### ccache

```bash
emerge --ask dev-util/ccache

# Em make.conf:
# FEATURES="... ccache"

# Configure:
ccache --max-size=10G
ccache --set-config=compression=true
```

Na segunda compilação de qualquer pacote (após atualização menor, por exemplo), hits de cache fazem a recompilação ser em segundos.

### Portage binhost local

Se você tiver mais de uma máquina Gentoo, o `buildpkg` FEATURE já gera binários locais. Configure `/etc/portage/binrepos.conf` para sincronizar entre máquinas.

### Gentoo prefix para software isolado

Para testar software sem afetar o sistema base, use o Prefix — uma instalação Gentoo em diretório arbitrário.

### emerge -av world periodicamente

```bash
emerge --ask --verbose --update --deep --newuse @world
```

- `--update`: atualiza pacotes com novas versões.
- `--deep`: verifica dependências de dependências (necessário para pegar atualizações transitivas).
- `--newuse`: recompila pacotes se USE flags mudaram.

Após: `emerge --depclean` remove dependências órfãs.

---

## 14. Referência Rápida

### Comandos Portage essenciais

```bash
# Instalar pacote:
emerge --ask <categoria/pacote>

# Remover pacote:
emerge --ask --depclean <categoria/pacote>

# Atualizar sistema:
emerge --ask --update --deep --newuse @world

# Buscar pacote:
emerge --search <nome>
# ou com mais detalhes:
emerge --searchdesc <nome>

# Ver USE flags de um pacote:
emerge --ask --verbose <pacote>
# ou:
equery uses <pacote>

# Ver dependências:
equery depends <pacote>
equery depgraph <pacote>

# Qual pacote provê um arquivo:
equery belongs /usr/bin/python

# Verificar integridade de arquivos instalados:
equery check <pacote>
```

### Diretórios /etc/portage/ importantes

```
/etc/portage/make.conf          ← configuração global
/etc/portage/package.use        ← USE flags por pacote
/etc/portage/package.accept_keywords  ← keywords por pacote
/etc/portage/package.mask       ← bloquear versões específicas
/etc/portage/package.env        ← variáveis de ambiente por pacote
/etc/portage/repos.conf/        ← repositórios adicionais (overlays)
```

### Exemplo: USE flag por pacote

```bash
# Habilitar suporte LTO apenas no Firefox:
echo "www-client/firefox lto" >> /etc/portage/package.use/firefox
```

### Overlays (repositórios adicionais)

```bash
emerge --ask app-eselect/eselect-repository
emerge --ask dev-vcs/git

# Listar overlays disponíveis:
eselect repository list | grep <nome>

# Adicionar overlay:
eselect repository enable guru  # overlay community mantido pelos usuários
emerge --sync
```

---

## Checklist de Instalação

```
[ ] Boot do live CD
[ ] Particionamento e formatação
[ ] Extração do stage3
[ ] make.conf configurado (CFLAGS, MAKEOPTS, USE, VIDEO_CARDS)
[ ] Chroot e sincronização do Portage
[ ] Fuso horário e locale
[ ] Configuração e compilação do kernel
[ ] Bootloader instalado
[ ] fstab correto com UUIDs
[ ] Rede configurada (dhcpcd + OpenRC)
[ ] Usuário criado com grupos corretos (wheel, video, audio, input, seat)
[ ] Driver NVIDIA instalado + Nouveau blacklistado
[ ] linux-firmware instalado (AMD iGPU)
[ ] seatd ou elogind configurado
[ ] Wayland env vars para NVIDIA
[ ] Hyprland instalado e config copiada
[ ] Primeiro boot bem-sucedido
[ ] Periféricos testados
```

---

## Leituras Complementares

- **Gentoo Handbook (AMD64):** `https://wiki.gentoo.org/wiki/Handbook:AMD64` — referência definitiva, mais completa que qualquer guia.
- **Kernel configuration:** `https://wiki.gentoo.org/wiki/Kernel/Gentoo_Kernel_Configuration_Guide`
- **NVIDIA no Gentoo:** `https://wiki.gentoo.org/wiki/NVIDIA/nvidia-drivers`
- **AMD GPU:** `https://wiki.gentoo.org/wiki/AMDGPU`
- **Hyprland no Gentoo:** `https://wiki.gentoo.org/wiki/Hyprland`
- **USE flags reference:** `https://www.gentoo.org/support/use-flags/`

---

*Guia gerado para: AMD Ryzen 9 9900X | GTX 750 Ti + Radeon 760M | Arch Linux → Gentoo com OpenRC*
