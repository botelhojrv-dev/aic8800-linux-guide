# AIC8800D80 no Linux — Driver, Bluetooth e Automação Completa

> Guia definitivo para fazer o adaptador WiFi 6 + Bluetooth (chipset AIC8800D80,
> IDs USB 1111:1111 / a69c:xxxx) funcionar no Linux Mint / Ubuntu com kernels 6.14+,
> com suporte automático via DKMS e udev.

![Status](https://img.shields.io/badge/status-resolvido-success)
![Kernel](https://img.shields.io/badge/kernel-6.14%20%E2%88%92%207.x-blue)
![Distros](https://img.shields.io/badge/testado-Linux%20Mint%2022.3%20%2F%20Ubuntu-informational)

## O Problema

Adaptadores baseados no chipset AIC8800D80 (vendidos como "AX900+", CUDY WU900,
Tenda U11 e similares) apresentam três armadilhas no Linux:

1. **Modo pendrive fantasma**: o dongle conecta como dispositivo de armazenamento USB
   de ~2MB (com os drivers Windows embutidos), e nunca comuta para modo rádio.
2. **Sem suporte no kernel mainline**: o driver precisa ser compilado fora da árvore.
3. **Bluetooth instável**: variantes de hardware com `chip_mcu_id=1` exigem firmware
   específico, senão o BT nunca inicializa (o WiFi funciona, o BT não).

## A Solução

Este repositório documenta o processo completo, do diagnóstico à automação:

- ✅ Diagnóstico sistemático (dmesg, lsusb, lsblk, systemctl)
- ✅ Comutação do modo storage via `eject` + regra udev
- ✅ Compilação e instalação do driver (branch correto por revisão de hardware)
- ✅ Registro no DKMS — recompila automaticamente em novos kernels
- ✅ Secure Boot: análise do impacto em módulos não assinados

## Resultado Final

| Item | Estado |
|------|--------|
| `aic8800/1.0.0` no DKMS | `installed` |
| `bluetooth.service` | `enabled`, sobe no boot |
| Comutação storage → rádio | Automática via udev |
| Fone Bluetooth | Pareado e funcionando |

## Documentação

📄 **[Guia completo passo a passo](TROUBLESHOOTING.md)** — diagnóstico detalhado,
comandos por fase, análise de logs e explicação de cada decisão técnica.

## Ambiente Testado

- Linux Mint 22.3 (base Ubuntu Noble 24.04)
- Kernel 6.14.0-37-generic (coexiste com 7.x)
- CPU AMD FX-8370E, Secure Boot desabilitado
- Driver: fork `gungdr/aic8800d80`, branch `legacy-mcu1`

## Contribuições

Encontrou uma variação de hardware diferente, outro erro de firmware ou testou
em outra distro? Abra uma issue ou PR — quanto mais variantes documentadas, melhor.

## Licença

MIT
