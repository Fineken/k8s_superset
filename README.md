# Apache Superset Helm Chart

Этот Helm чарт устанавливает [Apache Superset](https://superset.apache.org/) в Kubernetes кластер.

## Предварительные требования

- Kubernetes 1.19+
- Helm 3.0+
- PV provisioner support в кластере
- Ingress controller (опционально)
- Hashicorp Vault
- Jenkins

## Установка чарта

1. Добавьте репозиторий Helm:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

2. Установите чарт:
```bash
helm install my-superset ./superset
```

## Конфигурация

### Основные параметры

| Параметр | Описание | По умолчанию |
|----------|-----------|--------------|
| `image.repository` | Репозиторий образа Superset | `apache/superset` |
| `image.tag` | Тег образа Superset | `2.1.0` |
| `ingress.enabled` | Включить ingress | `true` |
| `ingress.hosts[0].host` | Хост для ingress | `superset.local` |

### Vault интеграция

Для интеграции с Vault необходимо настроить следующие переменные окружения в Jenkins pipeline:

```groovy
environment {
    VAULT_ADDR = "http://vault:8200"
    VAULT_SECRET_PATH = "secret/data/superset"
    VAULT_ROLE = "superset"
    VAULT_AUTH_PATH = "auth/kubernetes"
    K8S_NAMESPACE = "superset"
    SERVICE_ACCOUNT = "superset"
}
```

### Секреты в Vault

В Vault необходимо создать следующие секреты по пути `secret/data/superset`:

```json
{
    "admin_username": "admin",
    "admin_email": "admin@example.com",
    "admin_password": "secure_password",
    "secret_key": "your-secure-secret-key",
    "mapbox_api_key": "your-mapbox-api-key",
    "postgres_user": "superset",
    "postgres_password": "secure-postgres-password",
    "redis_password": "secure-redis-password"
}
```

### База данных

По умолчанию чарт устанавливает PostgreSQL. Параметры можно настроить в секции `postgresql` в values.yaml.

### Redis

Redis используется для кэширования и как брокер сообщений. Параметры можно настроить в секции `redis` в values.yaml.

## Доступ к Superset

После установки, Superset будет доступен по адресу, указанному в `ingress.hosts[0].host`. 

## Обновление

Для обновления установленного релиза:

```bash
helm upgrade my-superset ./superset
```

## Удаление

Для удаления установленного релиза:

```bash
helm delete my-superset
```

## Безопасность

- Все чувствительные данные хранятся в Vault
- Используется интеграция с Vault через Kubernetes service account
- При использовании в продакшене настройте SSL/TLS
- Убедитесь, что у сервисного аккаунта есть необходимые права для доступа к Vault