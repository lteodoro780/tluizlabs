# Kubuntu Enterprise Deployment — 300+ Estações

## Objetivo

Substituir instalações de Windows não licenciado em um parque de mais de 300 máquinas por Kubuntu, com integração completa ao domínio Active Directory (autenticação, GPO, antivírus, atualizações e impressoras).

## Escala

- 300+ estações de trabalho.
- Implantação padronizada via imagem clonada.

## 1. Implantação em massa (Rescuezilla)

Uma imagem de referência do Kubuntu, já configurada e testada, foi clonada para as estações via **Rescuezilla**, permitindo replicar a instalação em volume sem precisar reinstalar manualmente máquina por máquina.

Fluxo geral:

```text
1. Instalar e configurar uma máquina de referência (golden image)
2. Capturar a imagem com Rescuezilla
3. Restaurar a imagem nas demais estações via boot USB/rede
4. Rodar script de pós-implantação (hostname + join no domínio)
```

## 2. Integração com Active Directory (CID)

A integração com o domínio AD — incluindo suporte a GPO no Linux — foi feita usando a ferramenta **CID** (disponível no SourceForge), que permite que estações Linux recebam políticas de grupo de forma similar a estações Windows.

Com isso foi possível aplicar centralmente:

- Instalação e atualização de antivírus (Kaspersky);
- Atualização de sistema via mirror apt interno;
- Configuração de impressoras via CUPS.

## 3. Script de hostname — resolvendo conflito de DHCP

### Problema

A imagem clonada replicava o **mesmo hostname** em todas as máquinas restauradas, causando conflito no servidor DHCP (múltiplos hosts tentando se registrar com o mesmo nome).

### Solução

Um script Bash, disparado via GPO (CID) logo após a finalização da instalação, gerava um **hostname único por máquina**, baseado na VLAN do setor + final do IP atribuído.

```bash
#!/bin/bash
# gerar-hostname.sh
# Executado via GPO (CID) após a implantação da imagem

IP=$(hostname -I | awk '{print $1}')
VLAN_SETOR=$(echo "$IP" | cut -d. -f3)       # Exemplo: octeto que identifica o setor/VLAN
HOST_SUFFIX=$(echo "$IP" | cut -d. -f4)      # Último octeto do IP

NOVO_HOSTNAME="kb-vlan${VLAN_SETOR}-pc${HOST_SUFFIX}"

hostnamectl set-hostname "$NOVO_HOSTNAME"
echo "Hostname definido: $NOVO_HOSTNAME"

# Reaplica o registro no DNS/DHCP, se necessário
systemctl restart systemd-networkd 2>/dev/null
```

Esse script resolveu o conflito, já que cada estação passou a ter um hostname derivado da sua própria VLAN e IP, eliminando duplicidade.

## 4. Antivírus via GPO (Kaspersky)

A instalação e as políticas do Kaspersky foram distribuídas via GPO (CID), aplicando configuração padrão a todas as estações do domínio sem intervenção manual por máquina.

## 5. Atualização de sistema via mirror apt interno

As estações foram configuradas para apontar para um **mirror apt interno**, em vez do repositório público, permitindo:

- Atualizações mais rápidas (rede local);
- Controle centralizado de quais pacotes/versões são distribuídos;
- Redução de uso de banda externa.

```bash
# Exemplo de sources.list apontando para mirror interno (exemplo fictício)
deb http://mirror.example.local/ubuntu focal main restricted universe multiverse
```

## 6. Impressoras via CUPS + GPO

A configuração de impressoras seguiu uma lógica parecida com a do hostname: um script disparado via GPO identificava o setor/VLAN da máquina e configurava automaticamente a impressora correta via **CUPS**, sem necessidade de configuração manual pelo usuário final.

## Resultado

- Eliminação do uso de Windows não licenciado em mais de 300 estações.
- Antivírus, atualizações e impressoras centralizados e automatizados via GPO.
- Conflito de hostname/DHCP identificado e resolvido via automação.

## Observações de segurança

- Hostnames, IPs, VLANs e nomes de domínio neste documento são ilustrativos — não refletem a rede real da instituição.
- Scripts foram adaptados para fins de portfólio, mantendo a lógica real sem expor dados institucionais.
