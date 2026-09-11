# LOG-Server

# Amoreiratech IPFIX Flow Collector

Servidor de coleta de **Traffic Flow via IPFIX** para equipamentos de rede, com foco inicial em MikroTik RouterOS v7.

## Arquitetura

```text
MikroTik / Equipamento de Rede
          |
          | Traffic Flow / IPFIX
          | UDP 2055
          v
+-----------------------------+
| Debian 11                   |
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

O servidor **não utiliza rsyslog para registrar tráfego**. O objetivo é receber registros estruturados de fluxo através de IPFIX.

---

## Configuração padrão

| Item | Configuração |
|---|---|
| Sistema operacional | Debian 11 |
| Collector | nfcapd |
| Ferramenta de consulta | nfdump |
| Protocolo | IPFIX |
| Transporte | UDP |
| Porta | 2055 |
| Diretório | `/var/lib/ipfix` |
| Rotação | 5 minutos |
| Compressão | LZO |
| Retenção local | 7 dias |
| Warning de disco | 80% |
| Limpeza automática | 85% |
| Meta após limpeza | 78% |
| Nível crítico | 92% |

Os valores podem ser alterados no início do instalador antes da execução.

---

# 1. Instalação

Copie o arquivo:

```bash
instalar_ipfix_amoreiratech.sh
```

para o Debian 11.

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

O instalador irá:

1. instalar `nfdump/nfcapd`;
2. criar o usuário de serviço `ipfix`;
3. criar `/var/lib/ipfix`;
4. configurar buffers UDP do Linux;
5. criar o serviço `ipfix-collector`;
6. habilitar inicialização automática;
7. configurar rotação dos arquivos;
8. configurar retenção;
9. instalar proteção contra disco cheio;
10. criar comandos de diagnóstico.

---

# 2. Verificar o collector

Execute:

```bash
ipfix-status
```

O serviço deve aparecer como:

```text
active (running)
```

Também deverá existir um socket UDP na porta:

```text
2055
```

Verificação manual:

```bash
systemctl status ipfix-collector
```

Porta:

```bash
ss -lunp | grep 2055
```

Logs do serviço:

```bash
journalctl -u ipfix-collector
```

Acompanhamento em tempo real:

```bash
journalctl -u ipfix-collector -f
```

---

# 3. Configuração MikroTik RouterOS v7

Substitua `IP_DO_SERVIDOR` pelo endereço IP do Debian.

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

E:

```routeros
/ip traffic-flow target print detail
```

> Antes de colocar em produção em uma CCR com muito tráfego, recomenda-se revisar interfaces monitoradas, timeouts e eventual sampling.

---

# 4. Confirmar chegada dos pacotes IPFIX

No Debian:

```bash
tcpdump -ni any udp port 2055
```

Se o MikroTik estiver enviando corretamente, deverão aparecer pacotes UDP chegando ao servidor.

Exemplo conceitual:

```text
IP 172.30.0.1.49123 > SERVIDOR.2055: UDP
```

Se não aparecer nada, verifique:

- IP configurado no MikroTik;
- rota entre MikroTik e collector;
- firewall;
- porta UDP 2055;
- configuração do Traffic Flow.

---

# 5. Estrutura dos arquivos

Os registros ficam em:

```bash
/var/lib/ipfix/
```

O collector separa os exportadores e organiza os arquivos por data.

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

Cada arquivo representa aproximadamente **5 minutos de coleta**.

---

# 6. Consultar os últimos flows

Foi criado:

```bash
ipfix-last
```

Por padrão:

```bash
ipfix-last
```

mostra até 50 registros do flow-file fechado mais recente.

Para solicitar 100:

```bash
ipfix-last 100
```

---

# 7. Consultar diretamente com nfdump

Localize um arquivo:

```bash
find /var/lib/ipfix -type f -name 'nfcapd.*' | tail
```

Depois:

```bash
nfdump -r /caminho/do/nfcapd.202609111300
```

Formato estendido:

```bash
nfdump -r /caminho/do/arquivo -o extended
```

---

# 8. Filtrar um IP

Exemplo:

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

# 9. Filtrar portas e protocolos

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

# 10. Top Talkers

Por IP de origem:

```bash
nfdump -r /caminho/do/arquivo -s srcip/bytes -n 20
```

Por IP de destino:

```bash
nfdump -r /caminho/do/arquivo -s dstip/bytes -n 20
```

Isso ajuda a identificar os IPs que mais geraram tráfego no período armazenado naquele arquivo.

---

# 11. Proteção contra disco cheio

O servidor executa:

```bash
/usr/local/sbin/ipfix-cleanup
```

automaticamente a cada 15 minutos.

Política padrão:

```text
< 80%     NORMAL

>= 80%    WARNING

>= 85%    remove os flow-files fechados
           mais antigos até o filesystem
           voltar aproximadamente para 78%

>= 92%    CRITICAL
```

Além disso, a retenção normal é:

```text
7 dias
```

Arquivos ativos `nfcapd.current*` são explicitamente protegidos contra a rotina de limpeza.

Para executar manualmente:

```bash
/usr/local/sbin/ipfix-cleanup
```

Logs da rotina:

```bash
journalctl -t IPFIX-CLEANUP
```

---

# 12. Espaço utilizado

Filesystem:

```bash
df -h /var/lib/ipfix
```

Tamanho total:

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

---

# 13. Troubleshooting

## Serviço não inicia

```bash
systemctl status ipfix-collector
```

Depois:

```bash
journalctl -u ipfix-collector -n 100 --no-pager
```

## Porta 2055 não aparece

```bash
ss -lunp | grep 2055
```

Reinicie:

```bash
systemctl restart ipfix-collector
```

## MikroTik envia, mas não aparecem flows

Primeiro confirme os pacotes:

```bash
tcpdump -ni any udp port 2055
```

Depois acompanhe:

```bash
journalctl -u ipfix-collector -f
```

Confira o diretório:

```bash
find /var/lib/ipfix -type f | tail -20
```

## Ver configuração do serviço

```bash
systemctl cat ipfix-collector
```

---

# 14. Segurança

A porta:

```text
UDP/2055
```

não deve ficar exposta indiscriminadamente à Internet.

O ideal é permitir somente os endereços dos roteadores/exportadores autorizados.

Exemplo conceitual:

```text
CCR/BNG -------- UDP/2055 --------> IPFIX Collector
```

Se os equipamentos estiverem em redes distintas, utilize rede de gerência, VPN ou ACL/firewall apropriado.

---

# 15. Reinicialização

O serviço está habilitado para iniciar automaticamente junto com o Debian.

Verifique:

```bash
systemctl is-enabled ipfix-collector
```

Resultado esperado:

```text
enabled
```

Teste:

```bash
reboot
```

Após retornar:

```bash
ipfix-status
```

---

# 16. Comandos rápidos

```bash
# Status geral
ipfix-status

# Últimos flows
ipfix-last

# Serviço
systemctl status ipfix-collector

# Reiniciar
systemctl restart ipfix-collector

# Porta
ss -lunp | grep 2055

# Pacotes chegando
tcpdump -ni any udp port 2055

# Logs
journalctl -u ipfix-collector -f

# Uso de disco
df -h /var/lib/ipfix
du -sh /var/lib/ipfix

# Limpeza manual
/usr/local/sbin/ipfix-cleanup
```

---

# Próxima etapa recomendada

Depois de colocar um MikroTik em produção, deixe o collector receber tráfego durante **24 horas**.

Depois execute:

```bash
du -sh /var/lib/ipfix
```

e:

```bash
du -sh /var/lib/ipfix/*
```

Com esses valores será possível calcular:

- GB/dia por CCR;
- GB/mês;
- retenção possível no disco atual;
- capacidade necessária para vários roteadores;
- política adequada de backup externo;
- capacidade necessária no Google Drive.

Somente depois dessa medição é recomendável definir a retenção definitiva e o processo de backup.

---

**Projeto:** Amoreiratech IPFIX Flow Collector  
**Plataforma:** Debian 11  
**Collector:** nfcapd / nfdump  
**Protocolo:** IPFIX  
**Porta padrão:** UDP/2055
