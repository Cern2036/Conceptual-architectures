---
title: 03 appendix h скрипты развертывания
lang: ru
---

Appendix H: Детальные скрипты развертывания и управления
H.1. Основной скрипт развертывания (deploy.sh)
bash

#!/bin/bash
set -e

# Конфигурация
export PROJECT_NAME="zero1"
export ENVIRONMENT="${1:-staging}"
export COMPOSE_PROFILES="${2:-all}"

# Валидация окружения
validate_environment() {
    echo "🔍 Проверка окружения..."
    
    # Проверка необходимых переменных
    required_vars=("DB_PASSWORD" "MINIO_ROOT_PASSWORD" "GRAFANA_PASSWORD")
    for var in "${required_vars[@]}"; do
        if [ -z "${!var}" ]; then
            echo "❌ Переменная $var не установлена"
            exit 1
        fi
    done
    
    # Проверка Docker
    if ! command -v docker &> /dev/null; then
        echo "❌ Docker не установлен"
        exit 1
    fi
    
    # Проверка Docker Compose
    if ! docker compose version &> /dev/null; then
        echo "❌ Docker Compose не установлен"
        exit 1
    fi
    
    # Проверка NVIDIA Docker (если требуется)
    if [[ "$COMPOSE_PROFILES" == *"gpu"* ]]; then
        if ! docker run --rm --gpus all nvidia/cuda:12.1.0-base nvidia-smi &> /dev/null; then
            echo "❌ NVIDIA Docker не настроен"
            exit 1
        fi
    fi
    
    echo "✅ Окружение проверено"
}

# Инициализация инфраструктуры
init_infrastructure() {
    echo "🏗️  Инициализация инфраструктуры..."
    
    # Создание сетей Docker
    docker network create ${PROJECT_NAME}-backend 2>/dev/null || true
    docker network create ${PROJECT_NAME}-monitoring 2>/dev/null || true
    
    # Создание директорий для данных
    mkdir -p ./data/{postgres,minio,elasticsearch,prometheus,grafana}
    
    # Настройка прав доступа
    chown -R 1000:1000 ./data/postgres
    chown -R 1000:1000 ./data/minio
    chown -R 1000:1000 ./data/elasticsearch
    
    echo "✅ Инфраструктура готова"
}

# Развертывание базовых сервисов
deploy_base_services() {
    echo "🚀 Развертывание базовых сервисов..."
    
    # Запуск базового стека
    docker compose --profile base up -d \
        postgres \
        minio \
        elasticsearch \
        kibana
    
    # Ожидание готовности сервисов
    wait_for_service "PostgreSQL" "pg_isready -U zero1" 60
    wait_for_service "MinIO" "curl -f http://localhost:9000/minio/health/live" 30
    wait_for_service "Elasticsearch" "curl -f http://localhost:9200/_cluster/health" 45
    
    # Инициализация баз данных
    echo "📦 Инициализация баз данных..."
    docker compose exec -T postgres psql -U zero1 -d zero1 < ./sql/init_schema.sql
    docker compose exec -T postgres psql -U zero1 -d zero1 < ./sql/seed_data.sql
    
    # Настройка MinIO
    echo "🪣 Настройка MinIO..."
    docker compose run --rm mc mb minio/zero1-artifacts
    docker compose run --rm mc anonymous set download minio/zero1-artifacts
    
    echo "✅ Базовые сервисы запущены"
}

# Развертывание ML-сервисов
deploy_ml_services() {
    echo "🧠 Развертывание ML-сервисов..."
    
    # Загрузка моделей (если нет локальных копий)
    if [ ! -f "./models/llama-3.1-8b-ft.tar.gz" ]; then
        echo "⬇️  Загрузка моделей..."
        wget -O ./models/llama-3.1-8b-ft.tar.gz \
            https://storage.example.com/models/llama-3.1-8b-ft.tar.gz
        tar -xzf ./models/llama-3.1-8b-ft.tar.gz -C ./models/
    fi
    
    # Запуск ML-сервисов
    docker compose --profile ml up -d \
        ml-service \
        ml-cache
    
    # Ожидание готовности
    wait_for_service "ML Service" "curl -f http://localhost:8080/health" 120
    
    echo "✅ ML-сервисы запущены"
}

# Развертывание основных сервисов приложения
deploy_app_services() {
    echo "⚙️  Развертывание сервисов приложения..."
    
    # Запуск приложения
    docker compose --profile app up -d \
        zero1-controller \
        verifier \
        scheduler \
        worker
    
    # Ожидание готовности
    wait_for_service "Controller" "curl -f http://localhost:8000/api/v1/health" 30
    wait_for_service "Verifier" "curl -f http://localhost:8081/health" 30
    
    # Настройка планировщика задач
    echo "⏰ Настройка планировщика..."
    docker compose exec scheduler python init_scheduler.py
    
    echo "✅ Сервисы приложения запущены"
}

# Развертывание мониторинга
deploy_monitoring() {
    echo "📊 Развертывание мониторинга..."
    
    # Запуск мониторинга
    docker compose --profile monitoring up -d \
        prometheus \
        grafana \
        node-exporter \
        cadvisor
    
    # Ожидание готовности
    wait_for_service "Prometheus" "curl -f http://localhost:9090/-/healthy" 30
    wait_for_service "Grafana" "curl -f http://localhost:3000/api/health" 45
    
    # Настройка Grafana
    echo "⚙️  Настройка Grafana..."
    docker compose exec grafana grafana-cli admin reset-admin-password $GRAFANA_PASSWORD
    
    # Импорт дашбордов
    curl -X POST \
        -H "Content-Type: application/json" \
        -H "Accept: application/json" \
        -d @./monitoring/dashboards/zero1-overview.json \
        http://admin:$GRAFANA_PASSWORD@localhost:3000/api/dashboards/db
    
    echo "✅ Мониторинг запущен"
}

# Вспомогательная функция ожидания сервиса
wait_for_service() {
    local service_name=$1
    local check_command=$2
    local timeout=$3
    local start_time=$(date +%s)
    
    echo -n "⏳ Ожидание $service_name..."
    
    while true; do
        # Проверка таймаута
        local current_time=$(date +%s)
        local elapsed=$((current_time - start_time))
        
        if [ $elapsed -ge $timeout ]; then
            echo "❌"
            echo "Таймаут ожидания $service_name"
            exit 1
        fi
        
        # Попытка подключения
        if eval "$check_command" &> /dev/null; then
            echo "✅"
            return 0
        fi
        
        sleep 2
        echo -n "."
    done
}

# Основная логика
main() {
    echo "🚀 Запуск развертывания Project ZERO-1"
    echo "📌 Окружение: $ENVIRONMENT"
    echo "📌 Профили: $COMPOSE_PROFILES"
    
    # Проверка привилегий
    if [ "$EUID" -ne 0 ]; then
        echo "⚠️  Скрипт запущен без привилегий root"
        read -p "Продолжить? (y/N): " -n 1 -r
        echo
        if [[ ! $REPLY =~ ^[Yy]$ ]]; then
            exit 1
        fi
    fi
    
    # Выполнение этапов развертывания
    validate_environment
    init_infrastructure
    
    case "$COMPOSE_PROFILES" in
        "all")
            deploy_base_services
            deploy_ml_services
            deploy_app_services
            deploy_monitoring
            ;;
        "base")
            deploy_base_services
            ;;
        "ml")
            deploy_ml_services
            ;;
        "app")
            deploy_app_services
            ;;
        "monitoring")
            deploy_monitoring
            ;;
        *)
            echo "❌ Неизвестный профиль: $COMPOSE_PROFILES"
            exit 1
            ;;
    esac
    
    # Финальная проверка
    echo "🔍 Финальная проверка системы..."
    sleep 10
    
    # Проверка всех сервисов
    services=$(docker compose ps --services)
    for service in $services; do
        if [ "$(docker compose ps -q $service)" ]; then
            status=$(docker compose ps --format json | jq -r ".[] | select(.Service==\"$service\") | .Status")
            echo "  $service: $status"
        else
            echo "  ❌ $service: НЕ ЗАПУЩЕН"
        fi
    done
    
    echo ""
    echo "🎉 Развертывание завершено!"
    echo ""
    echo "🌐 Доступ к сервисам:"
    echo "  • Контроллер API: http://localhost:8000/docs"
    echo "  • MinIO Console: http://localhost:9001"
    echo "  • Kibana: http://localhost:5601"
    echo "  • Grafana: http://localhost:3000"
    echo "  • Prometheus: http://localhost:9090"
    echo ""
    echo "🔑 Учетные данные находятся в файле .env"
}

# Запуск основной функции
main "$@"

H.2. Скрипт управления системой (manage-system.sh)
bash

#!/bin/bash

COMMAND="${1:-help}"
SERVICE="${2:-all}"

case $COMMAND in
    start)
        echo "▶️  Запуск системы..."
        docker compose start $SERVICE
        ;;
        
    stop)
        echo "⏸️  Остановка системы..."
        docker compose stop $SERVICE
        ;;
        
    restart)
        echo "🔄 Перезапуск системы..."
        docker compose restart $SERVICE
        ;;
        
    status)
        echo "📊 Статус системы:"
        docker compose ps
        
        echo ""
        echo "📈 Метрики:"
        docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}" | head -10
        ;;
        
    logs)
        echo "📋 Логи системы:"
        docker compose logs --tail=100 -f $SERVICE
        ;;
        
    backup)
        echo "💾 Создание бэкапа..."
        TIMESTAMP=$(date +%Y%m%d_%H%M%S)
        BACKUP_DIR="./backups/backup_$TIMESTAMP"
        
        mkdir -p $BACKUP_DIR
        
        # Бэкап баз данных
        docker compose exec -T postgres pg_dump -U zero1 zero1 > $BACKUP_DIR/database.sql
        docker compose exec -T postgres pg_dump -U zero1 --schema-only zero1 > $BACKUP_DIR/schema.sql
        
        # Бэкап конфигураций
        cp -r ./config $BACKUP_DIR/
        cp .env $BACKUP_DIR/
        
        # Бэкап Docker volumes
        docker run --rm \
            -v ${PROJECT_NAME}_postgres_data:/source:ro \
            -v $(pwd)/$BACKUP_DIR:/backup \
            alpine tar czf /backup/postgres_data.tar.gz -C /source .
        
        # Создание архива
        tar czf ./backups/system_backup_$TIMESTAMP.tar.gz -C ./backups backup_$TIMESTAMP
        
        echo "✅ Бэкап создан: backups/system_backup_$TIMESTAMP.tar.gz"
        ;;
        
    update)
        echo "🔄 Обновление системы..."
        
        # Обновление образов
        docker compose pull
        
        # Остановка системы
        docker compose down
        
        # Запуск с новыми образами
        docker compose up -d
        
        # Применение миграций
        docker compose exec -T postgres psql -U zero1 -d zero1 < ./sql/migrations/latest.sql
        
        echo "✅ Система обновлена"
        ;;
        
    cleanup)
        echo "🧹 Очистка системы..."
        
        read -p "Остановить и удалить все контейнеры? (y/N): " -n 1 -r
        echo
        if [[ $REPLY =~ ^[Yy]$ ]]; then
            docker compose down -v
            docker system prune -af
            echo "✅ Система очищена"
        fi
        ;;
        
    test-cycle)
        echo "🧪 Запуск тестового цикла..."
        
        CYCLE_ID=$(uuidgen)
        
        # Запуск цикла через API
        curl -X POST \
            -H "Content-Type: application/json" \
            -d "{\"cycle_id\": \"$CYCLE_ID\", \"target\": \"csr1000v-test\", \"stig_id\": \"V-220668\"}" \
            http://localhost:8000/api/v1/cycle/start
            
        echo "✅ Цикл запущен: $CYCLE_ID"
        
        # Мониторинг прогресса
        while true; do
            STATUS=$(curl -s http://localhost:8000/api/v1/cycle/$CYCLE_ID/status | jq -r '.status')
            echo "Статус: $STATUS"
            
            if [[ "$STATUS" == "completed" ]] || [[ "$STATUS" == "failed" ]]; then
                break
            fi
            
            sleep 10
        done
        
        # Получение результата
        curl -s http://localhost:8000/api/v1/cycle/$CYCLE_ID/report | jq .
        ;;
        
    *)
        echo "Использование: $0 {start|stop|restart|status|logs|backup|update|cleanup|test-cycle} [service]"
        echo ""
        echo "Команды:"
        echo "  start       - Запуск системы или конкретного сервиса"
        echo "  stop        - Остановка системы или сервиса"
        echo "  restart     - Перезапуск системы или сервиса"
        echo "  status      - Показать статус системы и метрики"
        echo "  logs        - Показать логи системы или сервиса"
        echo "  backup      - Создать полный бэкап системы"
        echo "  update      - Обновить все образы и перезапустить"
        echo "  cleanup     - Очистить все контейнеры и образы"
        echo "  test-cycle  - Запустить тестовый цикл"
        ;;
esac

H.3. Скрипт мониторинга и алертинга (monitor.sh)
bash

#!/bin/bash

# Конфигурация
ALERT_THRESHOLDS=(
    "cpu_usage:80"
    "memory_usage:85"
    "disk_usage:90"
    "container_restarts:5"
    "api_latency_ms:5000"
    "error_rate:5"
)

# Проверка метрик
check_metrics() {
    echo "📊 Проверка метрик системы..."
    
    # CPU Usage
    CPU_USAGE=$(docker stats --no-stream --format "{{.CPUPerc}}" | sed 's/%//' | awk '{sum+=$1} END {print sum/NR}')
    if (( $(echo "$CPU_USAGE > 80" | bc -l) )); then
        send_alert "high_cpu" "Использование CPU: ${CPU_USAGE}%"
    fi
    
    # Memory Usage
    MEM_USAGE=$(docker stats --no-stream --format "{{.MemUsage}}" | awk '{print $1}' | sed 's/%//' | awk '{sum+=$1} END {print sum/NR}')
    if (( $(echo "$MEM_USAGE > 85" | bc -l) )); then
        send_alert "high_memory" "Использование памяти: ${MEM_USAGE}%"
    fi
    
    # Container Health
    UNHEALTHY=$(docker ps --filter "health=unhealthy" --format "{{.Names}}" | wc -l)
    if [ $UNHEALTHY -gt 0 ]; then
        send_alert "unhealthy_containers" "Неисправные контейнеры: $UNHEALTHY"
    fi
    
    # API Latency
    API_LATENCY=$(curl -o /dev/null -s -w '%{time_total}\n' http://localhost:8000/api/v1/health)
    if (( $(echo "$API_LATENCY > 5" | bc -l) )); then
        send_alert "high_latency" "Задержка API: ${API_LATENCY}с"
    fi
    
    # Database Connections
    DB_CONNS=$(docker compose exec -T postgres psql -U zero1 -t -c "SELECT count(*) FROM pg_stat_activity WHERE state = 'active';")
    if [ $DB_CONNS -gt 50 ]; then
        send_alert "high_db_connections" "Активных подключений к БД: $DB_CONNS"
    fi
}

# Проверка целостности данных
check_data_integrity() {
    echo "🔍 Проверка целостности данных..."
    
    # Проверка хэшей артефактов
    INTEGRITY_ERRORS=0
    
    # Проверка последних 10 циклов
    CYCLES=$(docker compose exec -T postgres psql -U zero1 -t -c "SELECT cycle_id FROM test_cycles ORDER BY created_at DESC LIMIT 10;")
    
    for CYCLE in $CYCLES; do
        CYCLE=$(echo $CYCLE | xargs)
        
        # Проверка Merkle Root
        EXPECTED_ROOT=$(docker compose exec -T postgres psql -U zero1 -t -c "SELECT merkle_root FROM test_cycles WHERE cycle_id = '$CYCLE';" | xargs)
        
        if [ -n "$EXPECTED_ROOT" ]; then
            # Проверка всех артефактов цикла
            ARTIFACTS=$(docker compose exec -T postgres psql -U zero1 -t -c "SELECT storage_path, sha256_hash FROM artifacts_registry WHERE cycle_id = '$CYCLE';")
            
            while IFS='|' read -r PATH HASH; do
                PATH=$(echo $PATH | xargs)
                HASH=$(echo $HASH | xargs)
                
                # Проверка хэша файла в MinIO
                ACTUAL_HASH=$(docker compose run --rm mc cat minio/zero1-artifacts/$PATH | sha256sum | awk '{print $1}')
                
                if [ "$ACTUAL_HASH" != "$HASH" ]; then
                    echo "❌ Нарушена целостность: $PATH"
                    ((INTEGRITY_ERRORS++))
                fi
            done <<< "$ARTIFACTS"
        fi
    done
    
    if [ $INTEGRITY_ERRORS -gt 0 ]; then
        send_alert "data_integrity" "Обнаружены ошибки целостности: $INTEGRITY_ERRORS"
    fi
}

# Проверка безопасности
check_security() {
    echo "🔒 Проверка безопасности..."
    
    # Проверка обновлений безопасности
    SECURITY_UPDATES=$(docker scout cves $(docker images -q) | grep -c "CRITICAL\|HIGH")
    
    if [ $SECURITY_UPDATES -gt 0 ]; then
        send_alert "security_updates" "Критические обновления безопасности: $SECURITY_UPDATES"
    fi
    
    # Проверка открытых портов
    OPEN_PORTS=$(ss -tuln | grep -E ':(8000|9000|9001|5601|3000|9090)' | wc -l)
    if [ $OPEN_PORTS -gt 6 ]; then
        send_alert "open_ports" "Обнаружены неожиданные открытые порты"
    fi
}

# Отправка алертов
send_alert() {
    local alert_type=$1
    local message=$2
    local timestamp=$(date +"%Y-%m-%d %H:%M:%S")
    
    echo "[$timestamp] 🚨 $message"
    
    # Запись в лог
    echo "[$timestamp] $alert_type: $message" >> /var/log/zero1/alerts.log
    
    # Отправка в Slack (если настроено)
    if [ -n "$SLACK_WEBHOOK" ]; then
        curl -X POST \
            -H 'Content-type: application/json' \
            --data "{\"text\":\"🚨 Project ZERO-1 Alert: $message\"}" \
            $SLACK_WEBHOOK > /dev/null 2>&1
    fi
    
    # Отправка email (если настроено)
    if [ -n "$ALERT_EMAIL" ]; then
        echo "Alert: $message" | mail -s "[ZERO1] $alert_type Alert" $ALERT_EMAIL
    fi
}

# Генерация отчетов
generate_report() {
    echo "📄 Генерация отчета..."
    
    REPORT_DATE=$(date +%Y-%m-%d)
    REPORT_FILE="/var/log/zero1/reports/daily_$REPORT_DATE.md"
    
    cat > $REPORT_FILE << EOF
# Ежедневный отчет Project ZERO-1
## Дата: $REPORT_DATE

### 📊 Статистика системы
- **Всего циклов:** $(docker compose exec -T postgres psql -U zero1 -t -c "SELECT count(*) FROM test_cycles;")
- **Успешных циклов:** $(docker compose exec -T postgres psql -U zero1 -t -c "SELECT count(*) FROM test_cycles WHERE overall_status = 'completed';")
- **Неудачных циклов:** $(docker compose exec -T postgres psql -U zero1 -t -c "SELECT count(*) FROM test_cycles WHERE overall_status = 'failed';")
- **Активных циклов:** $(docker compose exec -T postgres psql -U zero1 -t -c "SELECT count(*) FROM test_cycles WHERE overall_status = 'running';")

### ⚠️  Последние ошибки
$(docker compose exec -T postgres psql -U zero1 -t -c "SELECT error_message, created_at FROM cycle_stages WHERE status = 'failed' ORDER BY created_at DESC LIMIT 5;" | sed 's/|/ | /g')

### 💾 Использование ресурсов
- **CPU:** ${CPU_USAGE}%
- **Память:** ${MEM_USAGE}%
- **Диск:** $(df -h / | awk 'NR==2 {print $5}')
- **Сеть:** $(docker stats --no-stream --format "{{.NetIO}}" | head -1)

### 🔍 Рекомендации
EOF
    
    # Добавление рекомендаций на основе метрик
    if (( $(echo "$CPU_USAGE > 70" | bc -l) )); then
        echo "- ⚠️  Высокая нагрузка CPU. Рекомендуется масштабирование" >> $REPORT_FILE
    fi
    
    if [ $UNHEALTHY -gt 0 ]; then
        echo "- ⚠️  Обнаружены неисправные контейнеры. Требуется вмешательство" >> $REPORT_FILE
    fi
    
    echo "" >> $REPORT_FILE
    echo "*Отчет сгенерирован автоматически*" >> $REPORT_FILE
}

# Основная функция
main() {
    echo "🔍 Запуск мониторинга Project ZERO-1"
    echo "⏰ Время: $(date)"
    
    # Создание директорий для логов
    mkdir -p /var/log/zero1/{alerts,reports}
    
    # Выполнение проверок
    check_metrics
    check_data_integrity
    check_security
    generate_report
    
    echo "✅ Мониторинг завершен"
    echo "📁 Отчет: /var/log/zero1/reports/daily_$(date +%Y-%m-%d).md"
}

# Запуск с cron каждые 5 минут
if [ "$1" = "cron" ]; then
    main >> /var/log/zero1/monitoring.log 2>&1
else
    main
fi

H.4. Файл конфигурации Prometheus (prometheus.yml)
yaml

global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

rule_files:
  - "alerts/*.yml"

scrape_configs:
  # Node Exporter
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
    metrics_path: /metrics

  # Docker
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
    metrics_path: /metrics

  # PostgreSQL
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
    metrics_path: /metrics

  # Приложение
  - job_name: 'zero1-app'
    static_configs:
      - targets:
        - 'zero1-controller:8000'
        - 'ml-service:8080'
        - 'verifier:8081'
    metrics_path: /metrics
    scrape_interval: 30s

  # Blackbox exporter
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - http://zero1-controller:8000/health
        - http://ml-service:8080/health
        - http://minio:9000/minio/health/live
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

H.5. Правила алертинга Prometheus (alerts/zero1-alerts.yml)
yaml

groups:
  - name: zero1
    rules:
      # Алерты инфраструктуры
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Высокая загрузка CPU на {{ $labels.instance }}"
          description: "CPU usage is {{ $value }}%"

      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Высокая загрузка памяти на {{ $labels.instance }}"
          description: "Memory usage is {{ $value }}%"

      # Алерты приложения
      - alert: HighAPILatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{job="zero1-app"}[5m])) > 5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Высокая задержка API"
          description: "95-й перцентиль задержки {{ $value }}s"

      - alert: HighErrorRate
        expr: rate(http_requests_total{job="zero1-app", status=~"5.."}[5m]) / rate(http_requests_total{job="zero1-app"}[5m]) * 100 > 5
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Высокий уровень ошибок"
          description: "Error rate is {{ $value }}%"

      # Алерты базы данных
      - alert: DatabaseDown
        expr: up{job="postgres"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "База данных недоступна"
          description: "PostgreSQL instance is down"

      - alert: HighDatabaseConnections
        expr: pg_stat_activity_count > 50
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Высокое количество подключений к БД"
          description: "{{ $value }} активных подключений"

      # Алерты хранилища
      - alert: MinIODown
        expr: up{job="blackbox", instance="http://minio:9000/minio/health/live"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "MinIO недоступен"
          description: "Object storage is down"

      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Заканчивается место на диске"
          description: "Free space is {{ $value }}%"

      # Бизнес-метрики
      - alert: HighCycleFailureRate
        expr: rate(test_cycles_total{status="failed"}[1h]) / rate(test_cycles_total[1h]) * 100 > 10
        for: 30m
        labels:
          severity: critical
        annotations:
          summary: "Высокий процент неудачных циклов"
          description: "Failure rate is {{ $value }}%"

      - alert: NoCompletedCycles
        expr: increase(test_cycles_total{status="completed"}[1h]) == 0
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Нет завершенных циклов за час"
          description: "Система не обрабатывает задания"
