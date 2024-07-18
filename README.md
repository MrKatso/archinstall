# (PRÉ-INSTALAÇÃO) ANTES DE PROSSEGUIRMOS COM A INSTALAÇÃO

### DEFINIR O LAYOUT DO TECLADO E FONTE DO CONSOLE

**$bash:** `loadkeys br-abnt2`  
**$bash:** `setfont ter-116b`

### DEFINIR O IDIOMA DO AMBIENTE LIVE

**$bash:** `vim /etc/locale.gen`

_• Dentro do arquivo locale.gen descomente essa linha e gere-os com (locale-gen):_

```gen
#pt_BR.UTF-8 UTF-8
```

**$bash:** `locale-gen`  
**$bash:** `export LANG=pt_BR.UTF-8`

### CONECTAR À INTERNET

**$bash:** `iwctl --passphrase ($passphrase) station ($device) connect ($SSID)`

> 📌 Para verificar se está conectado à rede

**$bash:** `ping -c 4 archlinux.org`

### ATUALIZAR O RELÓGIO DO SISTEMA

**$bash:** `timedatectl`

### CLONE O REPOSITÓRIO DESTE GUIA

**$bash:** `git clone https://www.github.com/MrKatso/myarch`  
**$bash:** `cd /myarch`

### PARTICIONAMENTO DOS DISCOS

> 📌 Particionamento manual do disco usando o cfdisk

**$bash:** `cfdisk`

_• ESTRUTURA DE PARTIÇÕES:_

| _VOLUME_  | _RÓTULO_  | _TIPO_           | _SISTEMA DE ARQUIVOS_ | _TAMANHO_                                                        |
| --------- | --------- | ---------------- | --------------------- | ---------------------------------------------------------------- |
|           |           |                  |                       |                                                                  |
| /dev/sdXX | /boot/efi | EFI System       | fat-32                | 1GBs no mínimo, de acordo com a quantidade de Kernels instalados |
|           |           |                  |                       |                                                                  |
| /dev/sdXX | /         | Linux Filesystem | ext4                  | 30GBs no mínimo                                                  |
|           |           |                  |                       |                                                                  |
| /dev/sdXX | /home     | Linux Filesystem | ext4                  | 60GBs no mínimo                                                  |
|           |           |                  |                       |                                                                  |
| /dev/sdXX | swap      | Linux Swap       | none                  | 2GBs no mínimo, de acordo com a quantidade de ram instalada      |
|           |           |                  |                       |                                                                  |

### FORMATAR AS PARTIÇÕES

> 📌 Visuálizar as unidades e listar seus respectivos volumes

**$bash:** `lsblk`

> 📌 Formatar as unidades

**$bash:** `mkfs.vfat -F32 /dev/sdXX`  
**$bash:** `mkfs.ext4 /dev/sdXX`  
**$bash:** `mkfs.ext4 /dev/sdXX`  
**$bash:** `mkswap /dev/sdXX`

### MONTAR OS SISTEMAS DE ARQUIVOS

> 📌 Formatar as unidades

**$bash:** `mount -t ext4 /dev/sdXX /mnt`  
**$bash:** `mkdir -p /mnt/boot/efi`  
**$bash:** `mount -t vfat /dev/sdXX /mnt/boot/efi`  
**$bash:** `mkdir -p /mnt/home`  
**$bash:** `mount -t ext4 /dev/sdXX /mnt/home`  
**$bash:** `swapon /dev/sdXX`

# INSTALAÇÃO DO SISTEMA

### SELECIONAR OS ESPELHOS

> 📌 Faça um _backup_ do /etc/pacman.d/mirrorlist existente

**$bash:** `cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup`

> 📌 Copie o arquivo mirrorlist dentro da pasta "myarch" clonada do git para o diretório /etc/pacman.d/

**$bash:** `cp mirrorlist /etc/pacman.d/mirrorlist`

> 📌 Classifique os espelhos, aqui com o operando (-n 10) para emitir apenas os 10 espelhos mais rápidos

**$bash:** `rankmirrors -n 10 /etc/pacman.d/mirrorlist`

> 📌 Force a atualização dos mirrors do pacman

**$bash:** `pacman -Syyu`

### INSTALAR OS PACOTES ESSENCIAIS

**$bash:** `pacstrap -K /mnt base base-devel linux linux-firmware linux-headers bash-completion`

# CONFIGURAR O SISTEMA

### FSTAB

> 📌 Gere o arquivo fstab

**$bash:** `genfstab -U /mnt >> /mnt/etc/fstab`

### CHROOT

> 📌 Mude a raiz para o novo sistema

**$bash:** `arch-chroot /mnt`

### HORÁRIO

> 📌 Defina o fuso horário

**$bash:** `ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime`

> 📌 Execute "hwclock" para gerar /etc/adjtime

**$bash:** `hwclock --systohc --utc`

### LOCALIZAÇÃO

> 📌 Edite /etc/locale.gen e descomente pt_BR.UTF-8 UTF-8

**$bash:** `vim /etc/locale.gen`

_• Dentro do arquivo locale.gen descomente essa linha:_

```gen
pt_BR.UTF-8 UTF-8
```

> 📌 Gere os locales executando

**$bash:** `locale-gen`

> 📌 Crie o arquivo locale.conf e defina a variável LANG adequadamente

**$bash:** `echo LANG=pt_BR.UTF-8 > /etc/locale.conf`  
**$bash:** `export LANG=pt_BR.UTF-8`

> 📌 Defina o esquema do teclado do console como persistentes em vconsole.conf

**$bash:** `pacman -S terminus-font --needed --noconfirm`  
**$bash:** `vim /etc/vconsole.conf`

_• Dentro do arquivo vconsole.conf:_

```conf
KEYMAP=br-abnt2
FONT=ter-v16n
FONT_MAP=
```

### CONFIGURAÇÃO DE REDE

> 📌 Crie o arquivo hostname para setar o nome da máquina

**$bash:** `echo ($HOSTNAME) > /etc/hostname`

> 📌 Edite o arquivo /etc/hosts para setar o localhost

**$bash:** `vim /etc/hosts`

_• Dentro do arquivo hosts:_

```hosts
127.0.0.1     localhost.localdomain     localhost
::1           localhost.localdomain     localhost
127.0.0.1     ($HOSTNAME).localdomain   ($HOSTNAME)
```

### SENHA ROOT

> 📌 Defina a senha do root (conhecido como "superusuário")

**$bash:** `passwd`

### CRIANDO O USUÁRIO USUAL

> 📌 Cria o usuário katso no grupo wheel (acesso ao sudo) e define a senha padrão do user

**$bash:** `useradd -m -g users -G wheel joao`  
**$bash:** `passwd joao`

> 📌 Altere o arquivo sudoers

**$bash:** `vim /etc/sudoers`

_• Dentro do arquivo sudoers:_

```sudoers
$wheel ALL(ALL:ALL) NOPASSWD: ALL, !/usr/bin/passwd, !/usr/bin/passwd root
$wheel ALL(ALL:ALL) PASSWD: /usr/sbin/visudo

## NO FINAL DO ARQUIVO

Defaults editor=/usr/bin/vim
```

### PACMAN.CONF

> 📌 Modifique o arquivo /etc/pacman.conf

**$bash:** `vim /etc/pacman.conf`

_• Dentro do arquivo pacman.conf faça o seguinte:_

```conf
## ADICIONE

[options]
ILoveCandy

## DESCOMENTE

[options]
Color

[core]
SigLevel = PackageRequired
Include = /etc/pacman.d/mirrorlist

[extra]
SigLevel = PackageRequired
Include = /etc/pacman.d/mirrorlist

[community]
SigLevel = PackageRequired
Include = /etc/pacman.d/mirrorlist

# Para poder executar aplicações de 32 bits
[multilib]
#SigLevel = PackageRequired
Include = /etc/pacman.d/mirrorlist

## ALTERE
ParallelDownloads=10
```

### GERENCIADOR DE BOOT

> 📌 Instalando o grub como gerenciador de boot

**$bash:** `pacman -S grub efibootmgr os-prober`  
**$bash:** `grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=arch-grub --root-directory=/ /dev/sdXX --recheck`  
**$bash:** `grub-mkconfig -o /boot/grub/grub.cfg`

> 📌 Modifique o arquivo /etc/default/grub para que o os-prober identifique outros sistemas na inicialização

**$bash:** `vim /etc/default/grub`

_• Dentro do arquivo grub descomente essa linha:_

```grub
GRUB_DISABLE_OS_PROBER=false
```

> 📌 Gere o arquivo grub novamente para que as alterações tenham efeito

**$bash:** `grub-mkconfig -o /boot/grub/grub.cfg`

# FINALIZANDO A CONFIGURAÇÃO DO SISTEMA

### SCRIPT ARCHDI

> 📌 Baixe o script archdi dos repositórios oficiais e instale os pacotes necessários para o funcionamento completo do sistema, depois saia do script

**$bash:** `curl -LO archdi.sf.net/archdi > archdi`  
**$bash:** `sh archdi`

### FINAL

> 📌 Após usar o script remova-o

**$bash:** `rm -rf archdi`

> 📌 Saia do ambiente chroot

**$bash:** `exit`

> 📌 Desmonte as unidades recursivamente

**$bash:** `umount -R /mnt`  
**$bash:** `swapoff /dev/sdXX`

> 📌 Desligue a máquina e remova a unidade de instalação live

**$bash:** `shutdown -h now`