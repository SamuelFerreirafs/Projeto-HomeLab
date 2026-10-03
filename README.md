# Homelab: servidor caseiro com Linux, Docker, monitoramento e laboratório de AWS

Projeto pessoal de infraestrutura. Transformei um notebook antigo em um servidor Ubuntu para praticar administração de Linux, containers, monitoramento, backup e conceitos de nuvem AWS LocalStack, tudo em ambiente controlado e sem custo.

> **Projeto de estudo.** Registro aqui o que fiz, o que testei e também o que ainda não fiz ou não consegui provar.

## Objetivos

- Praticar Linux Server, redes e Docker na prática
- Rodar um serviço real (servidor de Minecraft) para gerar carga e testar o hardware
- Monitorar o servidor com Prometheus e Grafana
- Automatizar um backup e testar a restauração
- Estudar conceitos de AWS (S3 e IAM) usando o emulador LocalStack

## Ambiente

| Item | Detalhe |
|---|---|
| Máquina | Notebook antigo, i7 de 3ª geração (4 threads), 8 GB de RAM |
| Sistema | Ubuntu Server 26.04 LTS |
| Rede | Wi-Fi com IP fixo (netplan) |
| Containers | Docker Engine 29 + Docker Compose v5 |

## Arquitetura

```
 PC (SSH, navegador, jogo)
        │  rede local / Tailscale (VPN)
        ▼
┌──────────────── Notebook (Ubuntu Server) ────────────────┐
│  Docker                                                  │
│   ├─ minecraft (Paper)            porta 25565            │
│   ├─ node-exporter ─► prometheus ─► grafana  porta 3000  │
│   └─ localstack (emulador AWS)    porta 4566 (localhost) │
│                                                          │
│  cron (04:00) ─► backup.sh ─► ~/backups-minecraft/*.tar.gz│
│  Samba: pasta compartilhada com o Windows                │
└──────────────────────────────────────────────────────────┘
```

## O que foi feito

### 1. Base: acesso e rede

- **SSH** para administrar o servidor remotamente
- **Tailscale** (VPN) para acessar o servidor e conectar via SSH pelo celular utilizando o Termius
- **Samba** compartilhando uma pasta com o Windows, mapeada como unidade de rede.
- **Firewall (ufw):** criei regras para a porta do Minecraft, mas o ufw ficou inativo. Descobri que o Docker publica portas direto no iptables e passa por cima do ufw, então a proteção real aqui é não expor nada na internet.

### 2. Docker

Cada serviço roda em um container, descrito em um `docker-compose.yml`. Os dados importantes ficam em volumes fora do container, para sobreviverem a atualizações e recriações.

### 3. Servidor de Minecraft

Imagem `itzg/minecraft-server` com Paper (o Java já vem dentro da imagem).

```yaml
services:
  mc:
    image: itzg/minecraft-server
    container_name: minecraft
    ports:
      - "25565:25565"
    environment:
      EULA: "TRUE"
      TYPE: PAPER
      VERSION: "LATEST"      # ajustar para a versão usada no cliente
      MEMORY: "4G"
      ONLINE_MODE: "FALSE"
      VIEW_DISTANCE: "8"
    volumes:
      - ./data:/data
    restart: unless-stopped
    stdin_open: true
    tty: true
```

**Decisão de segurança:** o servidor roda em modo offline (`ONLINE_MODE: "FALSE"`), porque os clientes que uso não autenticam com conta Microsoft. Nesse modo qualquer pessoa pode entrar com o nome de outro jogador, inclusive de um administrador. Por isso o servidor só é acessível pela rede local e pela VPN, e a porta nunca foi aberta no roteador.

### 4. Backup automático

Script em bash agendado no `cron` (todo dia às 04:00).

```bash
#!/bin/bash
set -euo pipefail

ORIGEM="$HOME/minecraft/data"
DESTINO="$HOME/backups-minecraft"
MANTER=7
DATA=$(date +%Y-%m-%d_%H%M)

mkdir -p "$DESTINO"

# Pausa a gravação do mundo e garante que ela volte, mesmo se algo falhar
docker exec minecraft rcon-cli save-off
trap 'docker exec minecraft rcon-cli save-on' EXIT
docker exec minecraft rcon-cli save-all flush

cd "$ORIGEM"
tar -czf "$DESTINO/mundo_$DATA.tar.gz" world* server.properties

# Mantém só os 7 backups mais recentes
ls -1t "$DESTINO"/mundo_*.tar.gz | tail -n +$((MANTER+1)) | xargs -r rm --

echo "$(date '+%F %T') backup ok: mundo_$DATA.tar.gz"
```

Agendamento (`crontab -e`):

```
0 4 * * * /home/<usuario>/minecraft/backup.sh >> /home/<usuario>/backups-minecraft/backup.log 2>&1
```

Pontos de design:
- **`save-off` / `save-all flush` / `save-on`:** evitam copiar o mundo no meio de uma gravação, o que poderia gerar um backup corrompido.
- **`trap ... EXIT`:** garante que o salvamento volte a ser ligado mesmo se o script falhar.
- **`world*`:** nessa versão do Paper as dimensões ficam dentro de `world/dimensions/`, e o padrão pega tudo.
- **Rotação:** mantém 7 backups (cerca de 24 MB cada).

**Teste de restauração:** extraí o backup em uma pasta temporária e conferi o conteúdo.

### 5. Monitoramento (Prometheus + Grafana)

- **node-exporter** lê as métricas do servidor (CPU, memória, disco, rede, temperatura).
- **Prometheus** coleta a cada 15 s e guarda 15 dias de histórico.
- **Grafana** exibe os gráficos, usando o painel pronto *Node Exporter Full* (ID 1860).

`prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: servidor
    static_configs:
      - targets: ['host.docker.internal:9100']
```

`docker-compose.yml`:

```yaml
services:
  node-exporter:
    image: prom/node-exporter
    container_name: node-exporter
    network_mode: host
    pid: host
    command:
      - --path.rootfs=/host
    volumes:
      - /:/host:ro,rslave
    restart: unless-stopped

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    ports:
      - "127.0.0.1:9090:9090"
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=15d
    extra_hosts:
      - "host.docker.internal:host-gateway"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    restart: unless-stopped

  grafana:
    image: grafana/grafana-oss
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    restart: unless-stopped

volumes:
  prometheus-data:
  grafana-data:
```

O Prometheus fica acessível só de dentro do servidor (`127.0.0.1`). O Grafana fica na rede local, com a senha padrão trocada no primeiro acesso.

### 6. Laboratório de AWS com LocalStack

O [LocalStack](https://www.localstack.cloud/) emula serviços da AWS em um container. **É um emulador, não a AWS real.** Usei o plano gratuito Hobby (uso não comercial), que exige um token de autenticação.

```yaml
services:
  localstack:
    image: localstack/localstack
    container_name: localstack
    ports:
      - "127.0.0.1:4566:4566"
    environment:
      - LOCALSTACK_AUTH_TOKEN=${LOCALSTACK_AUTH_TOKEN}
    volumes:
      - ./volume:/var/lib/localstack
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
```

O token fica em um arquivo `.env` (fora do Git, veja a seção de segredos).

**AWS CLI:** instalei a versão 2 e criei um perfil `local` com credenciais fictícias (`test`/`test`) e a região `us-east-1`. Todo comando aponta para o emulador:

```bash
export AWS_PROFILE=local
export AWS_ENDPOINT_URL=http://localhost:4566

aws s3 mb s3://backup-minecraft
aws s3 cp mundo_AAAA-MM-DD_HHMM.tar.gz s3://backup-minecraft/
aws s3 ls s3://backup-minecraft/ --human-readable

# baixa de volta e confere a integridade
aws s3 cp s3://backup-minecraft/mundo_AAAA-MM-DD_HHMM.tar.gz /tmp/verifica.tar.gz
sha256sum mundo_AAAA-MM-DD_HHMM.tar.gz /tmp/verifica.tar.gz
```

Os dois hashes foram idênticos, então o arquivo voltou do S3 sem alterações.

**IAM:** criei um usuário `backup-minecraft` com uma política de menor privilégio (listar o bucket e enviar/ler objetos, sem apagar), gerei e troquei as chaves de acesso e configurei um perfil separado para ele.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::meu-primeiro-bucket"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::meu-primeiro-bucket/*"
    }
  ]
}
```

**O que não consegui provar:** o teste de "acesso negado" falhou no LocalStack, porque o plano Hobby não aplica as políticas IAM (esse recurso aparece nos planos pagos). O usuário conseguiu apagar objetos e listar outros buckets, mesmo sem ter permissão na política. A política está escrita, mas a verificação precisa ser feita na AWS real.

## Resultados observados

| Teste | Resultado |
|---|---|
| TPS do Minecraft (com 1 jogador) | 20.0 / 19.9 / 20.0 (1 min / 5 min / 15 min) |
| Uso de RAM com todos os containers | cerca de 52%, swap em 0% |
| Uso de disco | cerca de 19% |
| Temperatura da CPU em repouso | 53 a 61 °C (limite crítico informado pelo hardware: 105 °C) |
| Teste de carga (`stress-ng`, 4 threads, cerca de 1 min) | CPU próxima de 100%, visível no Grafana em tempo real |
| Backup do mundo | cerca de 24 MB, restauração testada em pasta temporária |
| Envio ao S3 (LocalStack) | arquivo de 23 MiB com hash idêntico após o download |

## Problemas e aprendizados

- **Docker e firewall:** portas publicadas pelo Docker ignoram o ufw. Aprendi a pensar na exposição pela rede e pelo roteador, e não só pelas regras do ufw.
- **Restauração por engano:** rodei um bloco de restauração de backup com um nome de arquivo de exemplo. O servidor subiu com um mundo vazio, mas o mundo original estava só movido de pasta e foi recuperado. Lição: preferir mover a apagar e não rodar comandos sem ler.
- **Região do AWS CLI:** a região do perfil ficou como `test` por um erro de digitação, e a criação do bucket falhou. Passei a conferir com `aws configure list`.
- **Variáveis de ambiente perdidas:** a sessão SSH caiu e, sem a variável `AWS_ENDPOINT_URL`, os comandos passaram a apontar para a AWS real. Lição: conferir o endpoint ao abrir uma sessão nova.
- **Limites do plano gratuito do LocalStack:** assumi que a aplicação de políticas IAM estava disponível e só descobri que não ao testar. Aprendi a verificar a documentação do plano antes de depender de um recurso.
- **Docker socket:** montar o `docker.sock` no LocalStack dá a ele poder equivalente a root sobre o Docker. Aceitável em laboratório local, e por isso a porta fica só em `127.0.0.1`.

## Limitações conhecidas

- O backup fica **no mesmo disco** do servidor: protege contra erro humano e corrupção, mas não contra falha do disco.
- O desempenho do Minecraft foi medido **com um único jogador**; ainda falta testar com vários.
- O teste de menor privilégio no IAM **não foi comprovado** (veja acima).
- O LocalStack Hobby **não guarda dados** depois de reiniciar o container.
- O servidor usa Wi-Fi, o que pode afetar a latência.

## Próximos passos

- [ ] Criar uma conta AWS real (MFA na conta raiz, alerta de orçamento, usuário administrador) e repetir o teste de permissões do IAM
- [ ] Enviar o backup para um bucket S3 real, com o usuário de menor privilégio
- [ ] Copiar os backups para outro dispositivo (segunda cópia fora do servidor)
- [ ] Configurar alertas no Grafana (disco, temperatura, queda de energia)
- [ ] Adicionar o cAdvisor para monitorar o uso de recursos por container
- [ ] Testar o servidor com vários jogadores conectados
- [ ] Subir um servidor de mídia (Jellyfin)

## Segredos e este repositório

- O token do LocalStack fica em um arquivo `.env`, **que não deve ser versionado**. Use um `.gitignore` com:

  ```
  .env
  *.tar.gz
  ```

- Mantenha um `.env.example` apenas com o nome da variável:

  ```
  LOCALSTACK_AUTH_TOKEN=
  ```

- Endereços IP e nomes de usuário foram substituídos por placeholders (`<usuario>`).
