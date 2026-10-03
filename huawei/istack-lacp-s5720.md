# Huawei S5720 — iStack + LACP (Eth-Trunk)

## Objetivo

Aumentar a quantidade de portas disponíveis para o projeto Vanguardeira, empilhando dois switches Huawei S5720-52X-PWR-SI via iStack e agregando o link de fibra entre eles com LACP (Eth-Trunk).

## Hardware

- 2x Huawei S5720-52X-PWR-SI

## Topologia

```text
[Switch 0 - Master]                    [Switch 1 - Member]
  XGE 0/0/3 ──┐                          ┌── XGE 1/0/3
  XGE 0/0/4 ──┼── Stack-Port (iStack) ───┼── XGE 1/0/4
              │                          │
  XGE 0/0/1 ──┴── Eth-Trunk 1 (LACP) ────┴── XGE 1/0/1
              (link de fibra agregado)
```

As portas 3 e 4 foram reservadas para o Stack, deixando as portas 1 e 2 livres para o LACP.

## 1. Configuração do Stack (iSTACK)

No Switch 0 (Master):

```text
system-view

stack slot 0 priority 200
interface stack-port 0/1
 port interface xgigabitethernet 0/0/3 enable
 quit
interface stack-port 0/2
 port interface xgigabitethernet 0/0/4 enable
 quit
```

No Switch 1 (Member): renumerar o slot, reiniciar, e configurar a stack-port no novo slot.

```text
stack slot 0 renumber 1
```

Após o reboot, configurar `stack-port 1/1` e `1/2` nas portas XGE 1/0/3 e 1/0/4.

## 2. VLAN de gerência e IP

```text
vlan 10
 description Gerencia
 quit

interface Vlanif 10
 ip address 192.168.100.94 255.255.255.0
 quit
```

## 3. Usuário, acesso Web e SSH

```text
aaa
 local-user admin password irreversible-cipher SenhaExemplo123
 local-user admin privilege level 15
 local-user admin service-type http ssh terminal
 quit

http server enable
http secure-server enable

user-interface vty 0 4
 authentication-mode aaa
 protocol inbound all
 quit
```

## 4. LACP (Eth-Trunk) entre os switches

```text
interface Eth-Trunk 1
 description Link_Agregacao_Fibra
 mode lacp
 port link-type trunk
 port trunk allow-pass vlan 10
 quit

interface XGigabitEthernet 0/0/1
 eth-trunk 1
 quit

interface XGigabitEthernet 1/0/1
 eth-trunk 1
 quit
```

## 5. Salvar configuração

```text
quit
save
```

## Troubleshooting real

### Porta não subia no Eth-Trunk

**Causa:** a porta destinada ao LACP já estava configurada com um modo de link/VLAN conflitante, impedindo a negociação do protocolo.

**Diagnóstico:** feito via console (cabo serial + PuTTY), já que a interface Web não era confiável nesse momento.

**Comandos úteis para verificar:**

```text
display stack
display eth-trunk 1
display interface XGigabitEthernet 0/0/1
```

### Switch não salvava configuração via interface Web e não aceitava desabilitar a porta do trunk

**Sintoma:** ao tentar aplicar `save` ou desabilitar a porta pela interface Web de gerência, a operação não tinha efeito.

**Correção:** abandonar a interface Web para esse tipo de ajuste e aplicar as mudanças diretamente via **console serial (PuTTY)**, que se mostrou mais confiável que a Web para operações de configuração crítica nesse modelo/firmware.

**Lição aprendida:** em equipamentos Huawei de borda, preferir CLI via console/SSH a interface Web para mudanças estruturais (stack, trunk, save), e tratar a Web apenas como visualização.

## Observações de segurança

- IP, senha e nome de usuário neste documento são fictícios — não refletem a rede real de produção.
- Este laboratório foi realizado em ambiente de trabalho; nenhum dado institucional real (hostname, VLAN de produção, topologia completa) é exposto aqui.
