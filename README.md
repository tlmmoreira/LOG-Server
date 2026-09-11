# Amoreiratech IPFIX Flow Collector

Servidor de coleta de **Traffic Flow via IPFIX** para equipamentos de rede, com foco inicial em MikroTik RouterOS v7.

Este README atende aos dois instaladores:

- `instalar_ipfix_amoreiratech.sh` — Debian 11
- `instalar_ipfix_amoreiratech_debian12.sh` — Debian 12

---

## 1. Arquitetura

```text
MikroTik / Equipamento de Rede
          |
          | Traffic Flow / IPFIX
          | UDP 2055
          v
+-----------------------------+
| Debian 11 ou Debian 12      |
| nfcapd / nfdump             |
+-------------+---------------+
              |
              v
       /var/lib/ipfix/
              |
              +-- Exportador
                    +-- Ano
                        +-- Mes
                            +-- Dia
                                +-- Hora
                                    +-- nfcapd.*
```

O projeto utiliza **IPFIX para coleta de fluxos**. Não utiliza rsyslog para registrar o tráfego.

---

## 2. Configuração padrão

| Item | Debian 11 | Debian 12 |
|---|---|---|
| Collector | nfcapd | nfcapd |
| Consulta | nfdump | nfdump |
| Protocolo | IPFIX | IPFIX |
| Transporte | UDP | UDP |
| Porta | 2055 | 2055 |
| Diretório | `/var/lib/ipfix` | `/var/lib/ipfix` |
| Rotação | 5 minutos | 5 minutos |
| Compressão | LZO | LZO |
| Retenção local | 7 dias | 7 dias |
| Warning de disco | 80% | 80% |
| Limpeza automática | 85% | 85% |
| Meta após limpeza | 78% | 78% |
| Nível crítico | 92% | 92% |

Os valores podem ser alterados nas variáveis do início do respectivo instalador antes da execução.

---

# 3. Escolha do instalador

Confira a versão:

```bash
cat /etc/debian_version
```

ou:

```bash
cat /etc/os-release
```

### Debian 11

Use:

```bash
instalar_ipfix_amoreiratech.sh
```

### Debian 12

Use:

```bash
instalar_ipfix_amoreiratech_debian12.sh
```

Não é necessário executar os dois scripts.

---

# 4. Instalação — Debian 11

Dê permissão:

```bash
chmod +x instalar_ipfix_amoreiratech.sh
```

Entre como root:

```bash
su -
```

Execute:

```bash
./instalar_ipfix_amoreiratech.sh
```

---

# 5. Instalação — Debian 12

Dê permissão:

```bash
chmod +x instalar_ipfix_amoreiratech_debian12.sh
```

Entre como root:

```bash
su -
```

Execute:

```bash
./instalar_ipfix_amoreiratech_debian12.sh
```

---

# 6. O que o instalador configura

O instalador correspondente à versão do Debian:

1. instala `nfdump/nfcapd`;
2. cria o usuário de serviço `ipfix`;
3. cria `/var/lib/ipfix`;
4. configura buffers UDP do Linux;
5. cria o serviço `ipfix-collector`;
6. habilita inicialização automática;
7. configura rotação dos flow-files;
8. configura retenção local;
9. instala proteção contra disco cheio;
10. cria comandos de diagnóstico;
11. valida a inicialização do collector.

O instalador do Debian 12 também instala `tcpdump` explicitamente para facilitar a validação da chegada dos datagramas IPFIX.

---

# 7. Verificar o collector

Execute:

```bash
ipfix-status
```

O serviço deve aparecer como:

```text
active (running)
```

Validação manual:

```bash
systemctl status ipfix-collector
```

Verifique a porta:

```bash
ss -lunp | grep 2055
```

Acompanhe o serviço:

```bash
journalctl -u ipfix-collector -f
```

---

# 8. Configuração MikroTik RouterOS v7

Substitua `IP_DO_SERVIDOR` pelo IP real do collector.

Habilite Traffic Flow:

```routeros
/ip traffic-flow
set enabled=yes
```

Adicione o destino IPFIX:

```routeros
/ip traffic-flow target
add dst-address=IP_DO_SERVIDOR port=2055 version=ipfix
```

Confira:

```routeros
/ip traffic-flow print
```

e:

```routeros
/ip traffic-flow target print detail
```

## Recomendação

Antes de habilitar em uma CCR de produção com grande volume de tráfego, revise:

- interfaces monitoradas;
- active-flow-timeout;
- inactive-flow-timeout;
- sampling, quando aplicável;
- CPU da CCR;
- quantidade de flows por segundo;
- capacidade do collector.

---

# 9. Confirmar chegada dos pacotes IPFIX

No Debian:

```bash
tcpdump -ni any udp port 2055
```

Se `tcpdump` não estiver instalado no Debian 11:

```bash
apt update
apt install tcpdump
```

Você deverá observar pacotes UDP do roteador chegando ao collector.

Exemplo conceitual:

```text
IP 172.30.0.1.49123 > IP_DO_SERVIDOR.2055: UDP
```

Se os pacotes chegam, mas os flow-files não são criados, investigue o `nfcapd`.

---

# 10. Estrutura de armazenamento

Os registros ficam em:

```bash
/var/lib/ipfix/
```

O collector separa os exportadores e organiza os flow-files por data.

Exemplo:

```text
/var/lib/ipfix/
└── 172-30-0-1/
    └── 2026/
        └── 09/
            └── 11/
                └── 13/
                    ├── nfcapd.202609111300
                    ├── nfcapd.202609111305
                    ├── nfcapd.202609111310
                    └── nfcapd.202609111315
```

A rotação padrão é:

```text
5 minutos
```

Assim, em condições normais, cada exportador poderá gerar aproximadamente:

```text
12 arquivos/hora
288 arquivos/dia
```

A quantidade real depende do comportamento da versão do collector e dos exportadores ativos.

---

# 11. Consultar os últimos flows

Foi criado o comando:

```bash
ipfix-last
```

Por padrão:

```bash
ipfix-last
```

exibe até 50 registros do flow-file fechado mais recente.

Para 100:

```bash
ipfix-last 100
```

---

# 12. Consulta manual com nfdump

Localize os arquivos:

```bash
find /var/lib/ipfix -type f -name 'nfcapd.*' ! -name 'nfcapd.current*' | tail
```

Leia um arquivo:

```bash
nfdump -r /caminho/do/nfcapd.202609111300
```

Formato estendido:

```bash
nfdump -r /caminho/do/arquivo -o extended
```

---

# 13. Filtrar por IP

Qualquer ocorrência do IP:

```bash
nfdump -r /caminho/do/arquivo 'host 100.64.10.20'
```

Somente origem:

```bash
nfdump -r /caminho/do/arquivo 'src ip 100.64.10.20'
```

Somente destino:

```bash
nfdump -r /caminho/do/arquivo 'dst ip 8.8.8.8'
```

---

# 14. Filtrar portas e protocolos

Porta 443:

```bash
nfdump -r /caminho/do/arquivo 'port 443'
```

TCP:

```bash
nfdump -r /caminho/do/arquivo 'proto tcp'
```

UDP:

```bash
nfdump -r /caminho/do/arquivo 'proto udp'
```

Exemplo combinado:

```bash
nfdump -r /caminho/do/arquivo 'src ip 100.64.10.20 and proto tcp'
```

---

# 15. Top Talkers

Por IP de origem:

```bash
nfdump -r /caminho/do/arquivo -s srcip/bytes -n 20
```

Por destino:

```bash
nfdump -r /caminho/do/arquivo -s dstip/bytes -n 20
```

Isso permite identificar rapidamente os maiores consumidores dentro do intervalo do flow-file consultado.

---

# 16. Proteção contra disco cheio

O servidor possui:

```bash
/usr/local/sbin/ipfix-cleanup
```

A rotina é executada automaticamente pelo cron.

Configuração padrão:

```text
< 80%
NORMAL

>= 80%
WARNING

>= 85%
Remove flow-files fechados mais antigos
até o filesystem chegar aproximadamente a 78%

>= 92%
CRITICAL
```

A retenção normal é:

```text
7 dias
```

Arquivos ativos:

```text
nfcapd.current*
```

são explicitamente excluídos da rotina de limpeza.

Execução manual:

```bash
/usr/local/sbin/ipfix-cleanup
```

Logs:

```bash
journalctl -t IPFIX-CLEANUP
```

---

# 17. Verificar armazenamento

Filesystem:

```bash
df -h /var/lib/ipfix
```

Total de flows:

```bash
du -sh /var/lib/ipfix
```

Por exportador:

```bash
du -sh /var/lib/ipfix/*
```

Maiores arquivos:

```bash
find /var/lib/ipfix -type f -printf '%s %p\n' | sort -nr | head -20
```

Quantidade de flow-files:

```bash
find /var/lib/ipfix -type f -name 'nfcapd.*' ! -name 'nfcapd.current*' | wc -l
```

---

# 18. Troubleshooting

## Serviço não inicia

```bash
systemctl status ipfix-collector
```

Depois:

```bash
journalctl -u ipfix-collector -n 100 --no-pager
```

## Porta UDP 2055 não aparece

```bash
ss -lunp | grep 2055
```

Reinicie:

```bash
systemctl restart ipfix-collector
```

## Não chegam pacotes

```bash
tcpdump -ni any udp port 2055
```

Se não aparecer tráfego, verifique:

```text
MikroTik
   |
   | rota / ACL / firewall
   |
   v
Debian:2055/UDP
```

Confira no MikroTik:

```routeros
/ip traffic-flow print
/ip traffic-flow target print detail
```

## Pacotes chegam, mas não existem flow-files

Acompanhe:

```bash
journalctl -u ipfix-collector -f
```

Depois:

```bash
find /var/lib/ipfix -type f | tail -20
```

Verifique o serviço:

```bash
systemctl cat ipfix-collector
```

## Reiniciar collector

```bash
systemctl restart ipfix-collector
```

---

# 19. Segurança

Não exponha indiscriminadamente:

```text
UDP/2055
```

à Internet.

O ideal é permitir somente os endereços dos exportadores autorizados:

```text
CCR-01 -----\
CCR-02 ------> UDP/2055 ---> IPFIX COLLECTOR
CCR-03 -----/
```

Prefira:

- rede de gerência;
- VLAN de infraestrutura;
- ACL;
- firewall;
- VPN quando o exportador estiver fora da rede administrativa.

---

# 20. Reinicialização do servidor

O collector é habilitado no boot.

Confira:

```bash
systemctl is-enabled ipfix-collector
```

Esperado:

```text
enabled
```

Depois de um reboot:

```bash
reboot
```

valide:

```bash
ipfix-status
```

---

# 21. Comandos rápidos

```bash
# Status
ipfix-status

# Últimos 50 flows
ipfix-last

# Últimos 100
ipfix-last 100

# Serviço
systemctl status ipfix-collector

# Reiniciar
systemctl restart ipfix-collector

# Porta
ss -lunp | grep 2055

# Pacotes IPFIX chegando
tcpdump -ni any udp port 2055

# Logs do collector
journalctl -u ipfix-collector -f

# Uso do filesystem
df -h /var/lib/ipfix

# Espaço dos flows
du -sh /var/lib/ipfix

# Espaço por exportador
du -sh /var/lib/ipfix/*

# Limpeza manual
/usr/local/sbin/ipfix-cleanup

# Log da proteção de disco
journalctl -t IPFIX-CLEANUP
```

---

# 22. Validação recomendada após implantação

Comece com **uma CCR**.

### Etapa 1 — Collector

```bash
ipfix-status
```

### Etapa 2 — Pacotes

```bash
tcpdump -ni any udp port 2055
```

### Etapa 3 — Arquivos

Depois de pelo menos 5 a 10 minutos:

```bash
find /var/lib/ipfix -type f | tail -20
```

### Etapa 4 — Conteúdo

```bash
ipfix-last 100
```

### Etapa 5 — Armazenamento

Após aproximadamente 24 horas:

```bash
du -sh /var/lib/ipfix
```

e:

```bash
du -sh /var/lib/ipfix/*
```

Com a medição de 24 horas é possível calcular de forma realista:

```text
GB/dia por CCR
GB/mês por CCR
flows por período
retenção possível
quantidade de CCRs suportadas
capacidade de disco necessária
necessidade de backup externo
```

---

# 23. Google Drive / backup externo

O backup para Google Drive **não está habilitado nestas versões dos instaladores**.

A recomendação é primeiro medir o volume real de IPFIX durante 24 horas. Depois disso, o projeto pode receber uma segunda etapa:

```text
nfcapd
   |
   v
flow-file fechado
   |
   +--> armazenamento local curto
   |
   +--> backup externo
             |
             v
        Google Drive
```

A política de exclusão local deve ser implementada somente depois da confirmação do upload remoto.

---

# 24. Observação importante sobre IPFIX e CGNAT

Traffic Flow/IPFIX é excelente para telemetria de fluxos:

- IP origem;
- IP destino;
- portas;
- protocolo;
- bytes;
- pacotes;
- duração;
- análise de tráfego;
- Top Talkers.

Entretanto, **não presuma que o Traffic Flow do equipamento exporta automaticamente todos os dados necessários para reconstruir uma tradução CGNAT**.

Antes de substituir qualquer mecanismo existente de registro de CGNAT, valide nos registros recebidos se estão presentes os campos necessários para a sua finalidade.

---

# 25. Arquivos do projeto

### Debian 11

```text
instalar_ipfix_amoreiratech.sh
```

### Debian 12

```text
instalar_ipfix_amoreiratech_debian12.sh
```

### README comum

```text
README_IPFIX_Amoreiratech_Debian11_Debian12.md
```

---

## Projeto

**Nome:** Amoreiratech IPFIX Flow Collector  
**Sistemas:** Debian 11 / Debian 12  
**Collector:** nfcapd  
**Consulta:** nfdump  
**Protocolo:** IPFIX  
**Porta padrão:** UDP/2055  
**Armazenamento:** `/var/lib/ipfix`  
**Rotação padrão:** 5 minutos  
**Retenção padrão:** 7 dias
