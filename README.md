# Projeto MQTT Seguro (Broker + Sensor + Subscriber)

## Descrição
Este projeto implementa um broker MQTT seguro com autenticação, autorização e criptografia TLS/SSL. O sistema foi desenvolvido para demonstrar as principais vulnerabilidades de segurança em brokers MQTT e suas respectivas correções.

## Serviços
- **mqtt-broker** (Eclipse Mosquitto 2.0.20), com persistência, logs e segurança implementada
- **temperature-sensor-1** (Python 3.12 + Paho MQTT), publica temperatura a cada 5s com autenticação
- **mqtt-subscriber** (mosquitto_sub), assina `sensor/#` e mostra mensagens no log com autenticação

## Vulnerabilidades Identificadas e Corrigidas

### 1. Acesso Anônimo (VULNERABILIDADE CRÍTICA)
**Problema:** O broker permitia conexões sem autenticação (`allow_anonymous true`)
**Solução:** 
- Desabilitado acesso anônimo (`allow_anonymous false`)
- Implementado sistema de autenticação com username/password
- Criados três usuários com diferentes níveis de acesso

### 2. Falta de Autorização (VULNERABILIDADE CRÍTICA)
**Problema:** Qualquer usuário autenticado podia acessar qualquer tópico
**Solução:**
- Implementado Access Control List (ACL) em `mosquitto/config/auth/acl`
- Restrições por usuário:
  - `sensor_user`: Apenas publicar em `sensor/+`
  - `subscriber_user`: Apenas ler de `sensor/+`
  - `admin_user`: Acesso completo a todos os tópicos

### 3. Comunicação Não Criptografada (VULNERABILIDADE CRÍTICA)
**Problema:** Todo tráfego MQTT era enviado em texto plano
**Solução:**
- Implementado TLS/SSL na porta 8883
- Gerados certificados SSL auto-assinados
- Configurado broker para aceitar conexões criptografadas
- Mantida porta 1883 para compatibilidade e testes

### 4. Clientes Sem Autenticação
**Problema:** Sensor e subscriber conectavam sem credenciais
**Solução:**
- Atualizado código Python do sensor para usar autenticação
- Configurado subscriber para usar credenciais
- Adicionadas variáveis de ambiente para credenciais

## Implementação Técnica

### Passo 1: Criação de Arquivos de Autenticação
```bash
# Criado diretório de autenticação
mkdir -p mosquitto/config/auth

# Gerados hashes de senha usando mosquitto_passwd
docker run --rm eclipse-mosquitto:2.0.20 sh -c "mosquitto_passwd -c -b /tmp/passwd sensor_user sensor123 && mosquitto_passwd -b /tmp/passwd subscriber_user subscriber123 && mosquitto_passwd -b /tmp/passwd admin_user admin123 && cat /tmp/passwd"
```

### Passo 2: Configuração do Access Control List (ACL)
Arquivo `mosquitto/config/auth/acl`:
```
user sensor_user
topic write sensor/+

user subscriber_user
topic read sensor/+

user admin_user
topic readwrite #
```

### Passo 3: Geração de Certificados SSL
```bash
# Criado diretório de certificados
mkdir -p mosquitto/config/certs

# Gerados certificados SSL
openssl req -new -x509 -days 365 -nodes -out mosquitto/config/certs/ca.crt -keyout mosquitto/config/certs/ca.key -subj "/C=BR/ST=SP/L=SaoPaulo/O=MQTT-Security/OU=IT/CN=MQTT-CA"
openssl genrsa -out mosquitto/config/certs/server.key 2048
openssl req -new -out mosquitto/config/certs/server.csr -key mosquitto/config/certs/server.key -subj "/C=BR/ST=SP/L=SaoPaulo/O=MQTT-Security/OU=IT/CN=172.16.39.60"
openssl x509 -req -in mosquitto/config/certs/server.csr -CA mosquitto/config/certs/ca.crt -CAkey mosquitto/config/certs/ca.key -CAcreateserial -out mosquitto/config/certs/server.crt -days 365
```

### Passo 4: Configuração do Broker
Arquivo `mosquitto/config/mosquitto.conf`:
```
# Listener padrão (não criptografado - apenas para testes)
listener 1883 0.0.0.0

# Listener TLS/SSL (criptografado - produção)
listener 8883 0.0.0.0
cafile /mosquitto/config/certs/ca.crt
certfile /mosquitto/config/certs/server.crt
keyfile /mosquitto/config/certs/server.key

# Configurações de segurança
allow_anonymous false
password_file /mosquitto/config/auth/passwd
acl_file /mosquitto/config/auth/acl
```

### Passo 5: Atualização dos Clientes
- **Sensor Python:** Adicionada autenticação com `client.username_pw_set()`
- **Subscriber:** Configurado com parâmetros `-u` e `-P`
- **Docker Compose:** Adicionadas variáveis de ambiente para credenciais

## Uso

### Iniciar o Sistema
```bash
docker compose up -d --build
docker compose logs -f mqtt-subscriber
```

### Testar Conexões

#### Conexão Não Criptografada (Porta 1883)
```bash
docker run --rm eclipse-mosquitto:2.0.20 mosquitto_pub -h 172.16.39.60 -p 1883 -t sensor/test -m "teste não criptografado" -u admin_user -P admin123
```

#### Conexão Criptografada TLS (Porta 8883)
```bash
docker run --rm -v $(pwd)/mosquitto/config/certs:/tmp/certs eclipse-mosquitto:2.0.20 mosquitto_pub -h 172.16.39.60 -p 8883 -t sensor/test -m "teste criptografado" -u admin_user -P admin123 --cafile /tmp/certs/ca.crt
```

## Detalhes de Conexão

### Broker MQTT
- **Host:** 172.16.39.60 (ou localhost)
- **Porta Não Criptografada:** 1883
- **Porta Criptografada:** 8883
- **Autenticação:** Obrigatória

### Usuários Configurados
- **admin_user / admin123** - Acesso completo a todos os tópicos
- **sensor_user / sensor123** - Apenas publicar em tópicos `sensor/+`
- **subscriber_user / subscriber123** - Apenas ler de tópicos `sensor/+`

## Segurança Implementada

✅ **Autenticação:** Acesso anônimo desabilitado, username/password obrigatório
✅ **Autorização:** ACL implementado com restrições por usuário
✅ **Criptografia:** TLS/SSL na porta 8883
✅ **Auditoria:** Logs detalhados para monitoramento de segurança
✅ **Isolamento:** Usuários com permissões mínimas necessárias
