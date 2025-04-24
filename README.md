# Guia de Instalação do Gentoo Linux

Este guia fornece instruções detalhadas para instalar o Gentoo Linux em máquinas com firmware BIOS tradicional e UEFI. Siga os passos cuidadosamente para uma instalação bem-sucedida.

## Índice
- [Pré-requisitos](#pré-requisitos)
- [Passo 1: Baixando e Preparando o Ambiente de Instalação](#passo-1-baixando-e-preparando-o-ambiente-de-instalação)
- [Passo 2: Configuração de Rede](#passo-2-configuração-de-rede)
- [Passo 3: Preparação do Disco](#passo-3-preparação-do-disco)
- [Passo 4: Instalando o Sistema Base](#passo-4-instalando-o-sistema-base)
- [Passo 5: Configurando o Sistema Base](#passo-5-configurando-o-sistema-base)
- [Passo 6: Compilando o Kernel](#passo-6-compilando-o-kernel)
- [Passo 7: Configurando o Sistema](#passo-7-configurando-o-sistema)
- [Passo 8: Instalando o Bootloader](#passo-8-instalando-o-bootloader)
- [Passo 9: Finalizando a Instalação](#passo-9-finalizando-a-instalação)
- [Solução de Problemas](#solução-de-problemas)

## Pré-requisitos

- Mídia de instalação do Gentoo (ISO ou USB)
- Computador com pelo menos 2GB de RAM (recomendado 4GB ou mais)
- Pelo menos 20GB de espaço livre em disco
- Conexão com a internet
- Paciência, pois a instalação do Gentoo pode levar algumas horas

## Passo 1: Baixando e Preparando o Ambiente de Instalação

1. Baixe a ISO mais recente do Gentoo do site oficial: https://www.gentoo.org/downloads/

2. Crie uma mídia bootável:
   ```bash
   # Para USB (substitua sdX pelo seu dispositivo USB)
   dd if=install-amd64-minimal-[version].iso of=/dev/sdX bs=4M status=progress
   ```

3. Inicie o computador com a mídia de instalação
   - Entre no BIOS/UEFI e configure para iniciar a partir do dispositivo USB
   - Selecione a opção de boot do Gentoo quando aparecer o menu

4. Uma vez iniciado, você verá o prompt do Gentoo LiveCD

## Passo 2: Configuração de Rede

1. Verifique se a interface de rede está disponível:
   ```bash
   ip link
   ```

2. Para redes cabeadas (normalmente automático):
   ```bash
   dhcpcd enp0s3  # Substitua enp0s3 pelo nome da sua interface
   ```

3. Para redes sem fio:
   ```bash
   # Carregue o firmware se necessário
   modprobe iwlwifi  # Para dispositivos Intel

   # Configure o WiFi
   wpa_passphrase "Nome_da_Rede" "Senha" > /etc/wpa_supplicant.conf
   wpa_supplicant -B -i wlp2s0 -c /etc/wpa_supplicant.conf
   dhcpcd wlp2s0
   ```

4. Teste a conexão:
   ```bash
   ping -c 3 www.gentoo.org
   ```

## Passo 3: Preparação do Disco

### Para sistemas BIOS:

1. Use fdisk para particionar o disco:
   ```bash
   fdisk /dev/sda
   ```

2. Crie um layout como este:
   - `/dev/sda1`: 2MB para BIOS boot partition (tipo: BIOS boot)
   - `/dev/sda2`: 128MB para /boot (tipo: Linux)
   - `/dev/sda3`: 4GB para swap (tipo: Linux swap)
   - `/dev/sda4`: Resto do disco para / (tipo: Linux)

3. Formate as partições:
   ```bash
   # Formatar a partição /boot
   mkfs.ext2 /dev/sda2
   
   # Configurar swap
   mkswap /dev/sda3
   swapon /dev/sda3
   
   # Formatar a partição raiz
   mkfs.ext4 /dev/sda4
   ```

### Para sistemas UEFI:

1. Use fdisk ou parted para criar um layout GPT:
   ```bash
   parted /dev/sda
   ```

2. Crie um layout como este:
   - `/dev/sda1`: 256MB para EFI System Partition (tipo: EFI)
   - `/dev/sda2`: 4GB para swap (tipo: Linux swap)
   - `/dev/sda3`: Resto do disco para / (tipo: Linux)

3. Formate as partições:
   ```bash
   # Formatar a partição EFI
   mkfs.fat -F32 /dev/sda1
   
   # Configurar swap
   mkswap /dev/sda2
   swapon /dev/sda2
   
   # Formatar a partição raiz
   mkfs.ext4 /dev/sda3
   ```

4. Monte as partições:
   ```bash
   # Para BIOS
   mount /dev/sda4 /mnt/gentoo
   mkdir /mnt/gentoo/boot
   mount /dev/sda2 /mnt/gentoo/boot
   
   # Para UEFI
   mount /dev/sda3 /mnt/gentoo
   mkdir -p /mnt/gentoo/boot/efi
   mount /dev/sda1 /mnt/gentoo/boot/efi
   ```

## Passo 4: Instalando o Sistema Base

1. Defina a data e hora:
   ```bash
   ntpd -q -g
   ```

2. Baixe o stage3 mais recente:
   ```bash
   cd /mnt/gentoo
   links https://www.gentoo.org/downloads/
   ```
   
   Alternativamente, use wget:
   ```bash
   wget https://bouncer.gentoo.org/fetch/root/all/releases/amd64/autobuilds/current-stage3-amd64-desktop-openrc/stage3-amd64-desktop-openrc-<date>.tar.xz
   ```

3. Extraia o stage3:
   ```bash
   tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
   ```

4. Configure o make.conf:
   ```bash
   nano -w /mnt/gentoo/etc/portage/make.conf
   ```
   
   Adicione/ajuste as seguintes linhas:
   ```
   COMMON_FLAGS="-O2 -pipe -march=native"
   MAKEOPTS="-j5"  # Número de cores + 1
   USE="X acl alsa bluetooth dbus elogind networkmanager pulseaudio systemd -gnome -kde"
   ACCEPT_LICENSE="*"
   VIDEO_CARDS="intel"  # ou "amdgpu radeonsi" para AMD, "nvidia" para NVIDIA
   ```

5. Selecione espelhos:
   ```bash
   mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf
   ```

6. Copie informações DNS:
   ```bash
   cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
   ```

## Passo 5: Configurando o Sistema Base

1. Monte sistemas de arquivos necessários:
   ```bash
   mount --types proc /proc /mnt/gentoo/proc
   mount --rbind /sys /mnt/gentoo/sys
   mount --make-rslave /mnt/gentoo/sys
   mount --rbind /dev /mnt/gentoo/dev
   mount --make-rslave /mnt/gentoo/dev
   mount --bind /run /mnt/gentoo/run
   mount --make-slave /mnt/gentoo/run
   ```

2. Entre no novo ambiente:
   ```bash
   chroot /mnt/gentoo /bin/bash
   source /etc/profile
   export PS1="(chroot) ${PS1}"
   ```

3. Monte a partição boot:
   ```bash
   mount /dev/sda2 /boot  # Para BIOS
   # OU
   mount /dev/sda1 /boot/efi  # Para UEFI
   ```

4. Atualize o repositório:
   ```bash
   emerge-webrsync
   emerge --sync
   ```

5. Escolha o perfil do sistema:
   ```bash
   eselect profile list
   eselect profile set <número>  # Ex: desktop/gnome or default/linux/amd64/17.1
   ```

6. Atualize o sistema base:
   ```bash
   emerge --ask --verbose --update --deep --newuse @world
   ```

7. Configure o fuso horário:
   ```bash
   echo "America/Sao_Paulo" > /etc/timezone
   emerge --config sys-libs/timezone-data
   ```

8. Configure a localização:
   ```bash
   nano -w /etc/locale.gen
   ```
   
   Descomente:
   ```
   en_US.UTF-8 UTF-8
   pt_BR.UTF-8 UTF-8
   ```
   
   Em seguida:
   ```bash
   locale-gen
   eselect locale list
   eselect locale set <número>  # Escolha pt_BR.utf8 ou en_US.utf8
   env-update && source /etc/profile
   ```

## Passo 6: Compilando o Kernel

1. Instale as fontes do kernel e as ferramentas necessárias:
   ```bash
   emerge --ask sys-kernel/gentoo-sources
   emerge --ask sys-kernel/genkernel
   emerge --ask sys-apps/pciutils
   ```

2. Configure e compile o kernel usando genkernel (método mais simples para iniciantes):
   ```bash
   genkernel all
   ```
   
   Ou compile manualmente (método avançado):
   ```bash
   cd /usr/src/linux
   make menuconfig
   make -j$(nproc)
   make modules_install
   make install
   ```

3. Instale o firmware necessário:
   ```bash
   emerge --ask sys-kernel/linux-firmware
   ```

## Passo 7: Configurando o Sistema

1. Configure o fstab:
   ```bash
   nano -w /etc/fstab
   ```
   
   Adicione linhas para suas partições, por exemplo:
   ```
   # Para BIOS
   /dev/sda2   /boot        ext2    defaults,noatime    0 2
   /dev/sda3   none         swap    sw                  0 0
   /dev/sda4   /            ext4    noatime             0 1
   
   # Para UEFI
   /dev/sda1   /boot/efi    vfat    defaults            0 2
   /dev/sda2   none         swap    sw                  0 0
   /dev/sda3   /            ext4    noatime             0 1
   ```

2. Configure o nome do host:
   ```bash
   echo "meu-gentoo" > /etc/hostname
   ```

3. Configure a rede:
   ```bash
   emerge --ask net-misc/dhcpcd
   emerge --ask net-wireless/wpa_supplicant  # Se precisar de WiFi
   emerge --ask net-misc/networkmanager      # Se preferir NetworkManager
   
   # Para iniciar automaticamente
   rc-update add dhcpcd default
   rc-update add NetworkManager default  # Se instalado
   ```

4. Defina a senha de root:
   ```bash
   passwd
   ```

5. Instale ferramentas de sistema:
   ```bash
   emerge --ask app-admin/sudo
   emerge --ask app-admin/sysklogd
   emerge --ask sys-process/cronie
   
   rc-update add sysklogd default
   rc-update add cronie default
   ```

## Passo 8: Instalando o Bootloader

### Para BIOS:

1. Instale o GRUB:
   ```bash
   emerge --ask sys-boot/grub
   grub-install /dev/sda
   grub-mkconfig -o /boot/grub/grub.cfg
   ```

### Para UEFI:

1. Instale o GRUB com suporte a UEFI:
   ```bash
   echo 'GRUB_PLATFORMS="efi-64"' >> /etc/portage/make.conf
   emerge --ask sys-boot/grub
   grub-install --target=x86_64-efi --efi-directory=/boot/efi
   grub-mkconfig -o /boot/grub/grub.cfg
   ```

## Passo 9: Finalizando a Instalação

1. Crie um usuário:
   ```bash
   useradd -m -G users,wheel,audio,video,usb,cdrom -s /bin/bash usuario
   passwd usuario
   ```

2. Configure o sudo:
   ```bash
   EDITOR=nano visudo
   ```
   
   Descomente a linha:
   ```
   %wheel ALL=(ALL) ALL
   ```

3. Instale um ambiente desktop (opcional):
   ```bash
   # Para XFCE
   emerge --ask xfce-base/xfce4-meta x11-terms/xfce4-terminal
   
   # Para GNOME
   emerge --ask gnome-base/gnome
   
   # Para KDE Plasma
   emerge --ask kde-plasma/plasma-meta
   ```

4. Instale um gerenciador de login (opcional):
   ```bash
   emerge --ask x11-misc/lightdm
   rc-update add lightdm default
   ```

5. Saia do ambiente chroot e reinicie:
   ```bash
   exit
   cd
   umount -l /mnt/gentoo/dev{/shm,/pts,}
   umount -R /mnt/gentoo
   reboot
   ```

## Solução de Problemas

### Problemas de rede após inicialização
Se você tiver problemas com a rede após a inicialização:
```bash
rc-service dhcpcd start
rc-service NetworkManager start
```

### Problemas com o X server
Se o servidor X não iniciar:
```bash
emerge --ask x11-drivers/xf86-video-<seu_driver>
```

### Problemas de som
Se não tiver áudio:
```bash
emerge --ask media-sound/alsa-utils
alsamixer  # Para ajustar o volume
```
