# Troubleshooting AIC8800D80 — Guia Técnico Passo a Passo

Registro técnico completo de um caso real: Bluetooth morto no Linux Mint 22.3,
kernel 6.14.0-37-generic, adaptador AIC8800D80 (AX900+BT, IDs USB 1111:1111).

## Fluxo de Diagnóstico (visão geral)

Bluetooth não funciona │ ▼ systemctl status bluetooth ──── inactive (dead) │ ▼ dmesg | grep -i bluetooth ───── nada encontrado │ ▼ lsusb ────────────────────────── 1111:1111 "Pandadia 88M80" ✓ presente │ ▼ lsblk ────────────────────────── /dev/sdb de 1,9MB "WIFI driver" │ └── adaptador travado em MODO STORAGE ▼ CAUSA RAIZ: kernel nunca viu o rádio — só um pendrive de drivers Windows

## Fase 1 — Diagnóstico

### Comandos e interpretação

```bash
# 1. Estado do serviço Bluetooth
sudo systemctl status bluetooth
# Sintoma observado: Active: inactive (dead)
# Causa: bluetoothd não inicia sem interface HCI para gerenciar.

# 2. Mensagens do kernel
sudo dmesg | grep -iE "usb|aic|bluetooth|hci"
# Achado-chave:
#   usb 2-4: New USB device found, idVendor=1111, idProduct=1111
#   usb 2-4: Product: 88M80, Manufacturer: AIC
#   usb-storage 2-4:1.0: USB Mass Storage device detected

# 3. Confirmação do "pendrive fantasma"
lsblk
#   sdb  1,9M └─sdb1  /media/ricardo/WIFI driver   ← não é um HD: é o dongle
Lição de diagnóstico

bluetooth.service morto NÃO significa problema do serviço — significa que não existe hardware Bluetooth visível para ele. Sempre desça a pilha: serviço → dmesg → USB → modo do dispositivo.
Fase 2 — Comutação do Modo Storage → Rádio

O chip AIC8800D80 conecta em modo "CD de drivers" (Zero-CD) e precisa ser ejetado para reenumerar como dispositivo de rádio.
# Backup dos drivers embutidos (contêm o SDK para Linux)
cp -r "/media/ricardo/WIFI driver" ~/AIC8800_backup

# Comutação manual
sudo umount "/media/ricardo/WIFI driver"
sudo eject /dev/sdb

# Evidência no dmesg:
#   sdb: detected capacity change from 3968 to 0
# Mas atenção: SEM driver instalado, o chip reenumera órfão.

    ⚠️ O eject é necessário, mas não suficiente. Sem os módulos do kernel (aic_load_fw, aic8800_fdrv), o rádio conecta sem nenhum driver para assumi-lo.

Fase 3 — Pré-requisitos e Secure Boot
sudo apt install build-essential linux-headers-$(uname -r) dkms git
O detalhe do Secure Boot

Módulos compilados fora da árvore do kernel NÃO são assinados pela Canonical. Com Secure Boot ativo, o kernel os rejeita:
module verification failed: signature and/or required key missing - tainting kernel

Verificação e solução:
mokutil --sb-state     # SecureBoot enabled → problema
# Opção A: desabilitar Secure Boot na UEFI (mais simples, usado neste caso)
# Opção B: assinar o módulo com chave própria via MOK (mais elaborado, mantém SB)
Fase 4 — Instalação do Driver
# Limpar firmwares antigos (firmware misturado pode congelar o sistema!)
sudo rm -rf /lib/firmware/aic8800*

# Clonar o driver mantido pela comunidade
git clone https://github.com/gungdr/aic8800d80.git
cd aic8800d80

# Regra udev: comuta o modo storage automaticamente a cada conexão
sudo cp aic.rules /usr/lib/udev/rules.d/
sudo udevadm control --reload-rules

# Compilar e instalar
sudo ./install.sh          # ou: make && sudo make install, conforme o fork
sudo depmod -a
sudo modprobe aic_load_fw
sudo modprobe aic8800_fdrv
A regra udev (o que ela faz)
KERNEL=="sd*", ATTRS{idVendor}=="a69c", ATTRS{idProduct}=="5721",
  RUN+="/usr/bin/eject /dev/%k"

Toda vez que o dongle conectar em modo storage, o udev o ejeta automaticamente. IDs cobertos nas variantes: a69c:5721, a69c:5722, a69c:572f.
Fase 5 — A Pegadinha do chip_mcu_id (Bluetooth morto com WiFi vivo)

Existem revisões de hardware distintas do AIC8800D80. Verifique no log:
sudo dmesg | grep -iE "chip_id|aic"
Log do driver	Branch correto	Consequência do branch errado
chip_id=7, chip_mcu_id=1	legacy-mcu1	Timeout determinístico no upload de firmware (0x170400); BT nunca inicializa
chip_mcu_id=0	main	—
# Se seu hardware é MCU1:
git checkout legacy-mcu1
sudo ./install.sh && sudo depmod -a

Nesse design, o driver AIC sobe o firmware e o btusb padrão do kernel assume a interface HCI — não existe módulo Bluetooth proprietário separado.
Fase 6 — Validação e Automação (DKMS)
dkms status
#   aic8800/1.0.0, 6.14.0-37-generic, x86_64: installed

DKMS = recompilação automática do driver a cada atualização de kernel (via hook do apt). Elimina o clássico "atualizou o kernel, morreu o WiFi".
Verificações finais
sudo systemctl is-enabled bluetooth    # → enabled
bluetoothctl show                      # → Controller ... Powered: yes
sudo dmesg | grep -i aic               # → módulos carregados sem erros
Cuidados

    Entradas DKMS duplicadas conflitam: sudo dkms remove aic8800/<versão> --all
    Regras em /usr/lib/udev/rules.d/ podem ser sobrescritas por updates; prefira /etc/udev/rules.d/ para persistência.
    Ao bootar em kernel novo (7.x), validar: sudo dkms autoinstall -k <versão>

Timeline do Caso Real
#	Ação	Resultado
1	systemctl status bluetooth	inactive (dead)
2	dmesg filtrado	Nenhum registro de rádio
3	lsusb + lsblk	Dongle em modo storage 1,9MB
4	eject /dev/sdb	Mídia ejetada, chip sem driver
5	Secure Boot desabilitado	Módulos não assinados liberados
6	Driver compilado (branch mcu1)	Interfaces de rádio ativas
7	DKMS + udev	Solução permanente e automática
8	Pareamento de fone	✔ Resolvido
Referências

    Driver: github.com/gungdr/aic8800d80 (branch legacy-mcu1)
    Driver original: github.com/goecho/aic8800_linux_drvier
    Variação DKMS: github.com/ronnyf/AIC8800-Linux-Driver


---