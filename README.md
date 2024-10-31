# GUIA COMPLETO INSTALAÇÃO ARCH COM BTRFS E SNAPPER

# TÓPICOS DA INSTALAÇÃO

- [INTRODUÇÃO](#INTRODUÇÃO)
- [ANTES DA INSTALAÇÃO](#ANTES-DA-INSTALAÇÃO)
-  [INSTALANDO O SISTEMA](#INSTALANDO-O-SISTEMA)
	- [Particionamento dos discos](#Particionamento-dos-discos)
	- [Formatação dos discos](#Formatação-dos-discos)
	- [Monte as partições](#Monte-as-partições)
	- [Selecionando os espelhos](#Selecionando-os-espelhos)
	- [Instalar o sistema base](#Instalar-o-sistema-base)
	- [Gerar o FSTAB](#Gerar-o-FSTAB)
	- [CHROOT no sistema](#CHROOT-no-sistema)
	- [Configurando o sistema](#Configurando-o-sistema)
		- [Língua do sistema](#Língua-do-sistema)
		- [Teclado do sistema](#Teclado-do-sistema)
		- [Local e hora](#Local-e-hora)
		- [Nome da máquina na rede](#Nome-da-máquina-na-rede)
		- [Módulos do kernel](#Módulos-do-kernel)
		- [Crie um ambiente ramdisk inicial](#Crie-um-ambiente-ramdisk-inicial)
		- [Usuários](#Usuários)
		- [Configurando o pacman](#Configurando-o-pacman)
		- [Instalando o grub](#Intalando-o-grub)
		- [Finalizando](#Finalizando)
	- [Final](#Final)
- [PÓS INSTALAÇÃO](PÓS-INSTALAÇÃO)

# INTRODUÇÃO

Processo de instalação do Projeto "MrArch", sistema de trabalho principal do ano de 2025. A seguir siga os passos com cautela e atenção para não ter de repetir todo o processo. A instalação será feita em modo UEFI e com o sistema de arquivos BTRFS com snapshots snapper.

# ANTES DA INSTALAÇÃO

Primeiro configure o layout do teclado e o idioma do ambiente LiveUSB.

```Zsh
# Para listar os teclados disponíveis
localectl list-keymaps

# Novo padrão us com acentos (Eua)
localectl set-x11-keymap us acentos

# Ou
loadkeys us-acentos

# Novo padrão br-abnt2 (Brasil)
localectl set-x11-keymap br abnt2

# Ou
loadkeys br-abnt2
```

<br>

**OPCIONAL:** Por padrão, a língua padrão do ambiente LiveUSB é o Inglês. Se quiser modificar, edite o arquivo "locale.gen" e descomente a língua preferível.

```Zsh
# Para editar o locale.gen
vim /etc/locale.gen

# Depois execute estes comandos
locale-gen
export LANG=pt_BR.UTF-8 ou $(OUTROLOCALE)
```

<br>

Cheque se está no modo UEFI.

```Zsh
# Se este comando retornar 64 ou 32, estás em UEFI
cat /sys/firmware/efi/fw_platform_size
```

<br>

Se por acaso o LiveUSB do Arch ISO estiver em modo Legacy, recrie a unidade USB bootável como GPT e UEFI, se não pode prosseguir com a instalação.

<br>

Verifique a conexão com a internet.

```Zsh
# Para verificar se tem internet
ping -c 5 archlinux.org

# Caso dê erro rode estes comandos (ETHERNET) que no caso deve conectar-se automaticamente
dhcpcd

# WIFI
iwctl

# Listar os dispositivos disponíveis
device list

# Para scanear as redes
station device scan

# Listar as redes disponíveis
station device get-networks

# Finalmente conecte-se ao SSID
station device connect SSID

# Sair do iwd
exit
```

<br>

Para mudar a fonte do ambiente LiveUSB.

```Zsh
# Caso a fonte terminus não estiver instalada (Note que deve estar conectado à internet)
pacman -Sy
pacman -S terminus-font

# Para setar uma fonte terminus de melhor visualização
setfont ter-v22n
```

<br>

Cheque o relógio do ambiente LiveUSB.

```Zsh
# Cheque se o ntp está ativado e se a hora está correta
timedatectl

# Se o ntp estiver desabilitado
timedatectl set-ntp true

# Ou isso
systemctl enable systemd-timesyncd.service
```

<br>

# INTALAÇÃO PRINCIPAL

## Particionamento dos discos

Para listar os discos disponíveis

```Zsh
# Lista os discos do X (Unidade, ex: sda1, nvme) desejados(as)
lsblk -f /dev/X
```

<br>

Configurando as partições com o CFDISK, lembrando de selecionar o formato GPT

```Zsh
# Listar as partições, "X" é a unidade alvo, ex: sda1, sdb1, nvme1
cfdisk /dev/X
# 
```

<br>

O esquema de partição para está instalação deve ser dessa forma:

| _Número_ | _Tipo_ | _Sistema de Arquivos_ | _Tamanho_ |
| --- | --- | --- | --- |
| 1 | Swap | SWAP | 2GBs no mínimo, de acordo com a quantidade de ram instalada |
| --- | --- | --- |--- |
| 2 | EFI System | FAT-32 | 1GBs no mínimo, de acordo com a quantidade de Kernels instalados |
| --- | --- | --- | --- |
| 3 | Linux Filesystem | BTRFS | Todo o restante do armazenamento |

<br>

## Formatação dos discos

Uma vez que as partições forem criadas, cada uma deve ser formatada com um sistema de arquivo adequado, exceto para partições SWAP.

```Zsh
# Para visualizar o particionamento dos discos
lsblk -l /dev/X

# SWAP (Note que em "X" deve se colocar a partição do SWAP)
mkswap -L SWAP /dev/X

# EFI System (Note que em "X" deve se colocar a partição do EFI System)
mkfs.vfat -F32 -n LINUXEFI /dev/X

# Linux Filesystem (Note que em "X" deve se colocar a partição do Linux Filesystem)
mkfs.btrfs -L LINUX /dev/X
```

<br>

## Monte as partições

Agora monte todas as partições criadas anteriormente

```Zsh
# Monte a partição raiz na pasta /mnt (Note que em "X" deve se colocar a partição do Linux Filesystem)
mount /dev/X /mnt
```

<br>

Agora crie os subvolumes BTRFS nesse esquema, adequado para esta instalação:

@ -> O subvolume raiz do sistema (contendo o sistema operacional e todos os arquivos do sistema).  
@home -> Este é o diretório inicial. Isso consiste na maioria dos seus dados, incluindo área de trabalho e downloads.  
@var -> útil para gerenciar logs e caches.  
@opt -> Para softwares adicionais ou aplicativos que podem ter configurações próprias.  
@.snapshots -> Diretório para armazenar instantâneos para o pacote snapper (Este será adicionado após a instalação).

```Zsh
# Partições essenciais
btrfs su cr /mnt/@
btrfs su cr /mnt/@home
btrfs su cr /mnt/@var
btrfs su cr /mnt/@opt

# Agora desmonte o /mnt
umount /mnt
```

<br>

**OPCIONAL:** Caso queira comprimir os subvolumes siga estes passos, senão pode pular esta parte. Caso seja um ssd, não esqueça de especificar depois do "relatime".

```Zsh
# Raiz (Note que em "X" deve se colocar a partição do Linux Filesystem)
mount -o rw,relatime,subvol=@ /dev/sdaX /mnt
mkdir -p /mnt/{boot/efi,home,var,opt}

# Home
mount -o rw,relatime,subvol=@home /dev/X /mnt/home

# Var
mount -o rw,relatime,subvol=@var /dev/X /mnt/var

# Opt
mount -o rw,relatime,subvol=@opt /dev/sda2 /mnt/opt

# Agora habilite a compressão dos volumes
btrfs property set /mnt/@ compression zstd
btrfs property set /mnt/@home compression zstd
btrfs property set /mnt/@var compression zstd
btrfs property set /mnt/@opt compression zstd
```

<br>

Agora vamos montar as partições. Caso seja um ssd, não esqueça de especificar depois do "relatime".

```Zsh
# Raiz (Note que em "X" deve se colocar a partição do Linux Filesystem)
mount -o rw,relatime,subvol=@ /dev/sdaX /mnt
mkdir -p /mnt/{boot/efi,home,var,opt}

# Home
mount -o rw,relatime,subvol=@home /dev/X /mnt/home

# Var
mount -o rw,relatime,subvol=@var /dev/X /mnt/var

# Opt
mount -o rw,relatime,subvol=@opt /dev/sda2 /mnt/opt
```

<br>

Agora as partições que são a Linux EFI e o SWAP.

```Zsh
# A partição Linux EFI (Note que em "X" deve se colocar a partição do Linux EFI)
mount /dev/X /mnt/boot/efi

# A partição SWAP (Note que em "X" deve se colocar a partição do SWAP)
swapon /dev/X
```

## Selecionando os espelhos

Primeiro tens de ativar a função de downloads paralelos do pacman. Para isso descomente no arquivo /etc/pacman.conf o "ParallelDownloads=5", se quiser pode aumentar de 5 para 10.

```Zsh
# Abra o arquivo
vim /etc/pacman.conf

# Descomente e altere de 5 para 10
ParallelDownloads=5
```

<br>

O ambiente LiveUSB do arch já vem com o reflector instalado por padrão, para usa-lo neste caso, siga estes passos.

-c -> Para selecionar o país  
-f -> Busca um número de espelhos mais rápidos  
--save -> Onde salvar o arquivo mirrorlist

```Zsh
# Usando o reflector para selecionar os 5 espelhos mais rápidos no país Brasil
reflector -c brazil -f 5 --save /etc/pacman.d/mirrorlist

# Caso queira visualizar o mirrorlist
cat /etc/pacman.d/mirrorlist
```

<br>

## Instalar o sistema base

Um sistema mínimo exige o pacote do grupo "base", também a instalação do grupo de pacote "base-devel" neste momento é altamente recomendado.

```Zsh
# Usando o pacstrap para instalar o sistema base e algumas ferramentas de antemão
pacstrap -K /mnt base base-devel linux linux-firmware linux-headers btrfs-progs amd-ucode git wget curl man man-db vim nano sudo
```

<br>

## Gerar o FSTAB

Gere o fstab com o script genfstab (Adicione a opção -U (UUIDs) ou -L (Labels), respectivamente).

```Zsh
# Gerar o fstab, que marca as partições montadas no sistema
genfstab -U /mnt >> /mnt/etc/fstab

# Verifique com o cat
cat /mnt/etc/fstab

# Opcional: Apenas a partição root(/) precisa de 1 no último campo. Todo o resto deve ser 2 ou 0. Além disso, data=ordered devem ser removidos.
vim /mnt/etc/fstab
```

<br>

## CHROOT no sistema

```Zsh
# Para entrar no ambiente do sistema
arch-chroot /mnt
```

<br>

## Configurando o sistema

Primeiramente realize novamente [estes passos](##Selecionando-os-espelhos) caso o mirrorlist não tenha sido herdado do ambiente LiveUSB, depois rode pacman -Sy para atualizar os espelhos.

### Língua do sistema

Primeiro configure a língua do sistema.

```Zsh
# Edite o arquivo /etc/locale.gen e decomente essas linhas
pt_BR.UTF-8 UTF-8
pt_BR ISO-8859-1

# Agora o arquivo /etc/locale.conf
echo LANG=pt_BR.UTF-8 > /etc/locale.conf

# Por fim
locale-gen
export LANG=pt_BR.UTF-8
```

<br>

### Teclado do sistema

Depois configure o teclado do sistema (TTY).

```Zsh
# Primeiro instale o pacote de fontes terminus
pacman -S terminus-font --needed --noconfirm

# Edite o arquivo /etc/vconsole.conf e coloque isso conforme o tipo de teclado (BR=br-abnt2, USACT=us-acentos)
KEYMAP=br-abnt2
FONT=ter-v22n
FONT_MAP=
```

<br>

### Local e hora

Agora configure o local e a hora.

```Zsh
# Para listar as timezones
timedatectl list-timezones

# Para America/Sao_Paulo, devemos criar um link simbólico
ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime

# Para atualizar a hora e ativar o formato UTC
hwclock --systohc --utc
```

<br>

### Nome da máquina na rede

Agora configure o nome da máquina e hosts

```Zsh
# Nome da máquina (Onde $(HOSTNAME) é o nome da máquina desejado que irá aparecer na rede)
echo ($HOSTNAME) > /etc/hostname

# Edite o arquivo /etc/hosts para configurar o localhost e coloque isso (Onde $(HOSTNAME) é o nome da máquina)
127.0.0.1     localhost.localdomain     localhost
::1           localhost.localdomain     localhost
127.0.1.1     ($HOSTNAME).localdomain   ($HOSTNAME)
```

<br>

### Módulos do kernel

Para um módulo relacionado ao hardware siga estes passos.

```Zsh
# Crie o arquivo com o nome do módulo, nesse caso o virtio
touch /etc/modules-load.d/virtio-net.conf

## Abra o arquivo com o vim e insira isso
# Load virtio-net.ko at boot
virtio-net

# Dont run the pcspkr module at boot
blacklist pcspkr
```

<br>

### Crie um ambiente ramdisk inicial

Isso é necessário para atualizar e aplicar as alterações feitas nos módulos do kernel.

```Zsh
# ramdisk
mkinitcpio -p linux
```

<br>

### Usuários

Senha para o root e criação do usuário principal do sistema

```Zsh
# Senha do usuário root
passwd

# Criando o usuário principal e definindo sua senha
useradd -m -g users -G wheel -s /bin/zsh mr
passwd mr

# Agora altere o arquivo /etc/sudoers para permitir que o usuário criado possa usar o sudo, com o editor vim e descomente e altere essas linhas
$wheel ALL(ALL:ALL) NOPASSWD: ALL, !/usr/bin/passwd, !/usr/bin/passwd root
$wheel ALL(ALL:ALL) PASSWD: /usr/sbin/visudo

# No final do arquivo coloque isso, para padronizar o editor ao usar o visudo
Defaults editor=/usr/bin/vim
```

<br>

### Configurando o pacman

Agora precisamos fazer algumas pequenas alterações no arquivo /etc/pacman.conf, siga os passos.

```Zsh
# Entre no arquivo pacman.conf
vim /etc/pacman.conf

# Realize essas modificações
## Adicione
[options]
ILoveCandy

- Repositórios -
[community]
SigLevel = PackageRequired
Include = /etc/pacman.d/mirrorlist
[multilib]
#SigLevel = PackageRequired
Include = /etc/pacman.d/mirrorlist

## Descomente
[options]
Color

- Repositórios -
[core]
SigLevel = PackageRequired
Include = /etc/pacman.d/mirrorlist

[extra]
SigLevel = PackageRequired
Include = /etc/pacman.d/mirrorlist
```

<br>

### Instalando o grub

Para iniciar o sistema, precisamos de um carregador boot do sistema.

```Zsh
# Primeiro instale o grub e o efibootmgr para sistemas em EFI
pacman -S grub efibootmgr ntfs-3g fuse3 --needed --noconfirm

# Caso esteja em dual boot, intale isso também
pacman -S os-prober --needed --noconfirm

# Agora instale o grub no sistema
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB --recheck

# Depois verifique se o arquivo foi instalado na pasta grub ou arch, o que deverá aparecer algum arquivo chamado grubx64.efi
ls /boot/efi/EFI/GRUB/

# Caso esteja em dual boot, siga estes passos
## Modifique o arquivo /etc/default/grub usando o vim para que o os-prober identifique outros sistemas na inicialização
## Dentro do arquivo grub descomente essas linhas
GRUB_DISABLE_OS_PROBER=false

# Se tudo estiver em seu devido lugar e nenhum erro reportado crie o arquivo de configuração do grub
grub-mkconfig -o /boot/grub/grub.cfg
```

<br>

### Finalizando

Instale mais alguns pacotes necessários para um sistema mais funcional

```Zsh
# Pacotes de compactação e descompactação de arquivos
pacman -S unace p7zip --needed --noconfirm

# Bluetooth
pacman -S bluez bluez-libs bluez-utils --needed --noconfirm

# Rede
pacman -S networkmanager network-manager-applet --needed --noconfirm

# Ferramentas do sistema
pacman -S xdg-user-dirs htop neofetch --needed --noconfirm

# Fontes do sistema e temas
pacman -S ttf-dejavu --needed --noconfirm

# Drivers de vídeo (INTEL) que são necessários
pacman -S mesa lib32-mesa vulkan-intel vulkan-icd-loader ocl-icd intel-media-driver --needed --noconfirm

# Drivers de vídeo (INTEL) que possívelmente não são necessários para essa configuração
pacman -S lib32-vulkan-intel --needed --noconfirm

# Drivers de vídeo (AMD) que são necessários
pacman -S mesa lib32-mesa vulkan-radeon vulkan-icd-loader ocl-icd libva-mesa-driver --needed --noconfirm

# Drivers de vídeo (AMD) que possívelmente não são necessários para essa configuração
pacman -S lib32-vulkan-radeon lib32-amdvlk lib32-libva-mesa-driver --needed --noconfirm
```

<br>

Para finalizar ative alguns serviços para melhor funcionamento do sistema

```Zsh
# Ative o serviço de rede
systemctl enable NetworkManager

# Ative o bluetooth (opcional)
systemctl enable bluetooth.service
```

<br>

## FINAL

Após terminar a configuração do sistema.

```Zsh
# Saia do ambiente chroot
exit

# Desmonte as partições
umount -R /mnt
swapoff /dev/X

# Reinicie a máquina
reboot -h now
```

<br>

# PÓS INSTALAÇÃO
