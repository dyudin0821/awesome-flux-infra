# Миграция кластера talos-dev-hel1-1 на Envoy Gateway

## Текущее состояние

Кластер мигрирован с Istio Service Mesh на Envoy Gateway с использованием Kubernetes Gateway API.

## Структура

### Envoy Gateway конфигурация
- **GatewayClass**: `talos-dev-eg` - класс шлюза с поддержкой merged gateways
- **Gateway**: `wildcard-gateway` - основной шлюз на портах 80/443
- **Wildcard Certificate**: `*.talos-dev-hel1-1.portogate.app` 
- **MetalLB**: Использует пул `public-pool` для LoadBalancer

### HTTPRoutes

Созданы HTTPRoute ресурсы для всех приложений:

#### Victoria Metrics (namespace: monitoring)
- `grafana-route` → grafana.${cluster_subdomain}
- `vmagent-route` → vmagent.${cluster_subdomain}
- `vmalert-route` → vmalert.${cluster_subdomain}
- `vmalertmanager-route` → vmalertmanager.${cluster_subdomain}
- `vmsingle-route` → vmsingle.${cluster_subdomain}

#### Другие приложения
- `n8n-route` (namespace: n8n) → n8n.${cluster_subdomain}
- `kubelinks-route` (namespace: kubelinks) → links.${cluster_subdomain}

#### Автоматический редирект
- `tls-redirect` - автоматическое перенаправление HTTP → HTTPS

## Применение миграции

### Поэтапное развертывание

**Шаг 1: Проверка конфигурации**
```bash
# Убедиться что конфигурация собирается без ошибок
kubectl kustomize clusters/talos-dev-hel1-1 --enable-helm | grep -E "^kind:" | sort | uniq -c

# Должно быть:
# - 1 Gateway
# - 1 GatewayClass  
# - 8 HTTPRoute
# - 1 EnvoyProxy
```

**Шаг 2: Применить Flux Kustomization**
```bash
# Применить изменения через Flux
flux reconcile kustomization talos-dev-hel1-1 --with-source

# Или применить напрямую (для тестирования)
kubectl kustomize clusters/talos-dev-hel1-1 --enable-helm | kubectl apply -f -
```

**Шаг 3: Проверить развертывание Envoy Gateway**
```bash
# Проверить pod'ы Envoy Gateway
kubectl get pods -n envoy-gateway-system

# Проверить Gateway
kubectl get gateway -n envoy-gateway-system wildcard-gateway

# Проверить сертификат
kubectl get certificate -n envoy-gateway-system wildcard-gateway-tls

# Получить IP LoadBalancer
kubectl get svc -n envoy-gateway-system
```

**Шаг 4: Проверить HTTPRoutes**
```bash
# Все HTTPRoutes
kubectl get httproute -A

# Статус конкретного route
kubectl describe httproute -n monitoring grafana-route
```

**Шаг 5: Обновить DNS**
```bash
# Получить новый IP от MetalLB
NEW_IP=$(kubectl get svc -n envoy-gateway-system -o jsonpath='{.items[?(@.spec.type=="LoadBalancer")].status.loadBalancer.ingress[0].ip}')
echo "Новый IP: $NEW_IP"

# External-DNS должен автоматически обновить записи
# Или обновить вручную в DNS провайдере
```

**Шаг 6: Проверить доступность приложений**
```bash
# Проверить HTTPS редирект
curl -I http://grafana.talos-dev-hel1-1.portogate.app

# Проверить приложения
curl -k https://grafana.talos-dev-hel1-1.portogate.app
curl -k https://n8n.talos-dev-hel1-1.portogate.app
curl -k https://links.talos-dev-hel1-1.portogate.app
```

**Шаг 7: Удалить Istio (после проверки)**
```bash
# Раскомментировать удаление istio в kustomization
# Применить изменения
flux reconcile kustomization talos-dev-hel1-1
```

## Откат

Если что-то пошло не так:

```bash
# 1. Восстановить istio в apps/bundles/talos-flex/kustomization.yaml
# Раскомментировать:
#   - ../../base/istio

# 2. Восстановить istio патчи в apps/bundles/talos-flex/hetzner-talos.yaml

# 3. Восстановить istio в clusters/talos-dev-hel1-1/kustomization.yaml
# Заменить:
#   - envoy-gateway
# На:
#   - istio

# 4. Применить
flux reconcile kustomization talos-dev-hel1-1

# 5. Вернуть DNS на старый IP
```

## Преимущества

1. ✅ Соответствие Kubernetes Gateway API стандарту
2. ✅ Упрощенная архитектура (нет полного service mesh)
3. ✅ Меньше overhead (только прокси, без control plane как istiod)
4. ✅ Легче переключаться между реализациями Gateway API
5. ✅ Автоматический HTTP → HTTPS редирект
6. ✅ Поддержка merged gateways для упрощения конфигурации

## Мониторинг

После миграции проверить:
- Метрики Envoy Gateway в Grafana
- Логи envoy-gateway pods
- Статус HTTPRoutes
- Доступность всех приложений
