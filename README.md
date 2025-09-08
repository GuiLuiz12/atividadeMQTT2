# Projeto MQTT (Broker + Sensor + Subscriber)

## Serviços
- **mqtt-broker** (Eclipse Mosquitto 2.0.20), com persistência e logs.
- **temperature-sensor-1** (Python 3.12 + Paho MQTT), publica temperatura a cada 5s.
- **mqtt-subscriber** (mosquitto_sub), assina `sensor/#` e mostra mensagens no log.

## Uso
```bash
docker compose up -d --build
docker compose logs -f mqtt-subscriber
```

## Notas
- Porta 1883 exposta no host. Se não precisar, remova `ports` do broker.
- `allow_anonymous true` apenas para laboratório. Em produção, configure `password_file`, `acl_file` e TLS.

MQTT Broker Connection Details:
- Host: [VM_IP_ADDRESS] (172.16.39.60) (localhost)
- Port: 1883
- Authentication: Required
- Users:
  * admin_user / admin123 (full access)
  * sensor_user / sensor123 (publish to sensor/+)
  * subscriber_user / subscriber123 (read from sensor/+)

  Arquivo de senhas criado e com hashes configuradas, acl configurado e funcionando, TLS também funcionando