# Conexão de Teclado e Mouse Bluetooth via Terminal (bluetoothctl)

## Contexto

Este documento registra o processo de pareamento de um teclado Bluetooth e um mouse Bluetooth (BT5.2) no Raspberry Pi Zero 2W do cyberdeck, utilizando exclusivamente o terminal (`bluetoothctl`), sem interface gráfica.

## 1. Diagnóstico inicial: adaptador Bluetooth bloqueado

Ao tentar iniciar o `bluetoothctl` e ligar o rádio, os seguintes erros ocorreram:

```
[bluetoothctl]> power on
Failed to set power on: org.bluez.Error.Failed
[bluetoothctl]> scan on
SetDiscoveryFilter failed: org.bluez.Error.NotReady
Failed to start discovery: org.bluez.Error.NotReady
```

### Verificação com `rfkill`

```bash
rfkill list
```

Resultado:
```
0: hci0: Bluetooth
        Soft blocked: yes
        Hard blocked: no
1: phy0: Wireless LAN
        Soft blocked: no
        Hard blocked: no
```

O adaptador Bluetooth (`hci0`) estava **soft blocked**, impedindo a ativação do rádio.

### Correção

```bash
sudo rfkill unblock bluetooth
```

Após o desbloqueio, `hciconfig -a` confirmou o controlador ativo (`UP RUNNING`), com o firmware Broadcom BCM43430A1 carregado corretamente (`dmesg | grep -i bluetooth`).

Também foi verificado que o arquivo `config.txt` **não** continha a linha `dtoverlay=disable-bt`, descartando essa causa alternativa.

## 2. Pareamento do teclado Bluetooth

Com o adaptador liberado, o processo padrão foi retomado:

```bash
bluetoothctl
agent on
default-agent
power on
scan on
```

### Particularidade encontrada: endereço MAC rotativo (RPA)

O teclado (BLE) apresentou **endereços MAC diferentes a cada novo scan** (ex.: `54:46:6B:E3:A6:6C`, depois `54:46:6B:F3:8B:1D`, depois `54:46:6B:61:03:58`), causando falhas do tipo `Device ... not available` ao tentar parear com um endereço já expirado.

Isso é um comportamento comum em dispositivos **Bluetooth Low Energy (BLE)** que usam *Random Private Address* por privacidade — o endereço muda periodicamente enquanto o dispositivo não está pareado (com bond estabelecido).

**Solução:** parear imediatamente assim que o nome do dispositivo (`Bluetooth Keyboard`) aparecer no scan, sem esperar, usando o endereço mais recente exibido:

```bash
scan off
pair <endereço_mais_recente>
trust <endereço_mais_recente>
connect <endereço_mais_recente>
```

O pareamento foi bem-sucedido nessa tentativa mais rápida.

## 3. Pareamento do mouse Bluetooth (BT5.2 Mouse)

Processo análogo, com o mouse identificado no scan como `BT5.2 Mouse` (endereço `2A:09:79:00:01:10`):

```bash
pair 2A:09:79:00:01:10
trust 2A:09:79:00:01:10
connect 2A:09:79:00:01:10
```

O pareamento expôs o perfil completo de **HID sobre GATT (BLE HID)**, incluindo:

- Generic Access / Generic Attribute Profile
- Device Information (Manufacturer Name, PnP ID)
- Human Interface Device (Protocol Mode, Report, Boot Mouse Input Report, HID Information)
- Battery Service (Battery Level, HID Control Point)

Resultado: `Pairing successful`.

## 4. Verificação final dos dispositivos conectados

Confirmação via `/proc/bus/input/devices`:

```
N: Name="Bluetooth Keyboard"
...
H: Handlers=sysrq kbd leds event2

N: Name="BT5.2 Mouse"
...
H: Handlers=mouse0 event3

N: Name="BT5.2 Mouse Keyboard"
...
H: Handlers=sysrq kbd leds event4
```

Ambos os dispositivos foram reconhecidos corretamente pelo kernel, cada um com seu handler de evento (`event2`, `event3`, `event4`) e o mouse adicionalmente registrado como `mouse0`.

## Resumo do fluxo de comandos (referência rápida)

```bash
# Preparação
sudo systemctl start bluetooth
rfkill list
sudo rfkill unblock bluetooth
hciconfig -a

# Pareamento (dentro do bluetoothctl)
bluetoothctl
agent on
default-agent
power on
scan on
# aguardar dispositivo aparecer e parear imediatamente
pair <MAC>
trust <MAC>
connect <MAC>
exit

# Verificação
cat /proc/bus/input/devices
```

## Observações para o artigo

- O caso do endereço MAC rotativo do teclado BLE é um ponto interessante para documentar como desafio prático enfrentado durante a integração de periféricos no cyberdeck, já que foge do fluxo "padrão" normalmente descrito em tutoriais.
- O `trust` aplicado a ambos os dispositivos garante reconexão automática em boots futuros, sem necessidade de repetir o pareamento.
