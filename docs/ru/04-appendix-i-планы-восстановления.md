---
title: 04 appendix i планы восстановления
lang: ru
---

Appendix I: Планы аварийного восстановления (Disaster Recovery)
I.1. Классификация инцидентов
Уровень	Описание	Время восстановления (RTO)	Точка восстановления (RPO)
Уровень 1	Частичная деградация сервиса (1-2 контейнера)	≤ 15 минут	0 минут (горячее резервирование)
Уровень 2	Потеря одного сервиса (БД, MinIO, Elasticsearch)	≤ 1 часа	5 минут (репликация)
Уровень 3	Потеря всей системы (сбой хоста, ЦОД)	≤ 4 часа	1 час (ежечасные снапшоты)
Уровень 4	Катастрофический сбой (потеря данных, компрометация)	≤ 24 часа	24 часа (ежедневные бэкапы)
I.2. Процедуры восстановления по уровням

I.2.1. Восстановление уровня 1 (частичная деградация)
bash

#!/bin/bash
# recovery-level1.sh

echo "🔧 Восстановление уровня 1: Частичная деградация"

# 1. Идентификация проблемного сервиса
FAILED_SERVICES=$(docker ps --filter "health=unhealthy" --format "{{.Names}}")

for SERVICE in $FAILED_SERVICES; do
    echo "🔄 Перезапуск $SERVICE..."
    
    # 2. Сохранение логов перед перезапуском
    docker logs $SERVICE > /tmp/${SERVICE}_crash.log 2>&1
    
    # 3. Перезапуск с увеличением ресурсов
    docker compose restart $SERVICE
    
    # 4. Мониторинг восстановления
    timeout 120 bash -c "
        while ! docker ps --filter 'name=$SERVICE' --filter 'health=healthy' | grep -q $SERVICE; do
            sleep 5
            echo -n '.'
        done
    "
    
    if [ $? -eq 0 ]; then
        echo "✅ $SERVICE восстановлен"
    else
        echo "❌ $SERVICE не восстановлен, эскалация до уровня 2"
        escalate_to_level2 $SERVICE
    fi
done

# 5. Проверка интеграции
echo "🔍 Проверка интеграции сервисов..."
curl -f http://localhost:8000/api/v1/health || escalate_to_level2 "controller"
curl -f http://localhost:8080/health || escalate_to_level2 "ml-service"

echo "✅ Восстановление уровня 1 завершено"

I.2.2. Восстановление уровня 2 (потеря критического сервиса)
bash

#!/bin/bash
# recovery-level2.sh

SERVICE=$1
BACKUP_TIMESTAMP=$(date +%Y%m%d_%H%M%S)

echo "🚨 Восстановление уровня 2: Потеря $SERVICE"

case $SERVICE in
    postgres)
        echo "🗄️  Восстановление PostgreSQL..."
        
        # 1. Остановка сервиса
        docker compose stop postgres
        
        # 2. Восстановление из последнего снапшота
        SNAPSHOT=$(ls -t /backups/postgres/*.snapshot | head -1)
        
        # 3. Восстановление данных
        docker run --rm \
            -v ${PROJECT_NAME}_postgres_data:/restore \
            -v $(dirname $SNAPSHOT):/backup \
            alpine sh -c "rm -rf /restore/* && tar xzf /backup/$(basename $SNAPSHOT) -C /restore"
        
        # 4. Запуск с проверкой целостности
        docker compose up -d postgres
        
        # 5. Проверка восстановления
        timeout 300 bash -c '
            while ! docker compose exec -T postgres pg_isready -U zero1; do
                sleep 10
            done
        '
        
        # 6. Проверка данных
        docker compose exec -T postgres psql -U zero1 -d zero1 -c "SELECT COUNT(*) FROM test_cycles;" || {
            echo "❌ Ошибка восстановления данных"
            escalate_to_level3 "postgres_data_corrupt"
        }
        ;;
        
    minio)
        echo "🪣 Восстановление MinIO..."
        
        # Минио поддерживает репликацию автоматически
        # Переключение на резервный экземпляр
        docker compose stop minio
        docker compose up -d minio-standby
        
        # Обновление endpoint во всех сервисах
        update_service_endpoint "MINIO_ENDPOINT" "minio-standby:9000"
        ;;
        
    elasticsearch)
        echo "🔍 Восстановление Elasticsearch..."
        
        # 1. Остановка кластера
        docker compose stop elasticsearch kibana
        
        # 2. Восстановление из снапшота репозитория
        docker compose run --rm elasticsearch \
            curl -X POST "localhost:9200/_snapshot/backup_repo/snapshot_$(date +%Y%m%d)/_restore?wait_for_completion=true"
        
        # 3. Перезапуск
        docker compose up -d elasticsearch kibana
        
        # 4. Проверка
        curl -f http://localhost:9200/_cluster/health?wait_for_status=yellow&timeout=5m
        ;;
        
    *)
        echo "⚠️  Неизвестный сервис: $SERVICE"
        escalate_to_level3 "unknown_service_failure"
        ;;
esac

echo "✅ Восстановление $SERVICE завершено"

I.2.3. Восстановление уровня 3 (полная потеря системы)
bash

#!/bin/bash
# recovery-level3.sh

DISASTER_TYPE=$1
RECOVERY_POINT=$2

echo "💥 Восстановление уровня 3: $DISASTER_TYPE"

# 1. Активация резервного ЦОД
activate_dr_site() {
    echo "🌐 Активация резервного ЦОД..."
    
    # DNS переключение
    aws route53 change-resource-record-sets \
        --hosted-zone-id Z123456789 \
        --change-batch '{
            "Changes": [{
                "Action": "UPSERT",
                "ResourceRecordSet": {
                    "Name": "zero1-api.example.com",
                    "Type": "A",
                    "TTL": 60,
                    "ResourceRecords": [{"Value": "192.168.100.100"}]
                }
            }]
        }'
    
    # Запуск инфраструктуры в DR-сайте
    ssh dr-site-manager "cd /opt/zero1 && ./deploy.sh production dr"
}

# 2. Восстановление данных
restore_from_backup() {
    local backup_file=$1
    
    echo "💾 Восстановление из бэкапа: $backup_file"
    
    # Скачивание бэкапа
    aws s3 cp s3://zero1-dr-backups/$backup_file /tmp/recovery.tar.gz
    
    # Развертывание
    tar xzf /tmp/recovery.tar.gz -C /opt/zero1-recovery
    
    # Восстановление конфигурации
    cp -r /opt/zero1-recovery/config/* /opt/zero1/config/
    cp /opt/zero1-recovery/.env /opt/zero1/
    
    # Восстановление volumes
    restore_volumes /opt/zero1-recovery/volumes/
}

# 3. Процедуры для разных типов аварий
case $DISASTER_TYPE in
    "host_failure")
        echo "🖥️  Восстановление после сбоя хоста..."
        
        # Запуск на резервном железе
        migrate_to_backup_host
        
        # Восстановление из последнего снапшота
        restore_from_backup "hourly_$(date +%Y%m%d_%H).tar.gz"
        ;;
        
    "datacenter_outage")
        echo "🏢 Восстановление после аварии ЦОД..."
        
        activate_dr_site
        
        # Восстановление из последнего доступного бэкапа
        LAST_BACKUP=$(aws s3 ls s3://zero1-dr-backups/ | grep daily | tail -1 | awk '{print $4}')
        restore_from_backup $LAST_BACKUP
        ;;
        
    "data_corruption")
        echo "📛 Восстановление после коррупции данных..."
        
        # Остановка всех сервисов
        docker compose down
        
        # Поиск последнего чистого бэкапа
        find_clean_backup $RECOVERY_POINT
        
        # Полное восстановление
        restore_from_backup $CLEAN_BACKUP
        
        # Валидация данных
        validate_recovered_data
        ;;
        
    "security_breach")
        echo "🔒 Восстановление после компрометации..."
        
        # 1. Изоляция системы
        iptables -A INPUT -j DROP
        
        # 2. Создание чистого окружения
        setup_clean_environment
        
        # 3. Восстановление из доверенного бэкапа
        restore_from_backup "verified_$(date -d '7 days ago' +%Y%m%d).tar.gz"
        
        # 4. Смена всех ключей и сертификатов
        rotate_all_secrets
        
        # 5. Аудит восстановленной системы
        security_audit_recovered
        ;;
esac

# 4. Запуск системы
echo "🚀 Запуск восстановленной системы..."
docker compose up -d

# 5. Валидация
echo "🔍 Валидация восстановления..."
run_recovery_validation

echo "✅ Восстановление уровня 3 завершено"

I.3. Регулярные процедуры резервного копирования

I.3.1. Ежечасные снапшоты
bash

#!/bin/bash
# backup-hourly.sh

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/hourly/$TIMESTAMP"

echo "💾 Создание ежечасного снапшота..."

# 1. Снапшот PostgreSQL
docker compose exec -T postgres pg_dumpall -U zero1 | gzip > $BACKUP_DIR/postgres_full.sql.gz

# 2. Снапшот MinIO данных
docker compose run --rm mc mirror minio/zero1-artifacts $BACKUP_DIR/minio/

# 3. Снапшот конфигураций
cp -r /opt/zero1/config $BACKUP_DIR/
cp /opt/zero1/.env $BACKUP_DIR/

# 4. Создание контрольной суммы
find $BACKUP_DIR -type f -exec sha256sum {} \; > $BACKUP_DIR/checksums.sha256

# 5. Отправка в удаленное хранилище
aws s3 sync $BACKUP_DIR s3://zero1-backups/hourly/$TIMESTAMP/

# 6. Очистка старых бэкапов (храним 24 часа)
find /backups/hourly/* -mmin +1440 -exec rm -rf {} \;

echo "✅ Ежечасный снапшот создан: $BACKUP_DIR"

I.3.2. Ежедневные полные бэкапы
bash

#!/bin/bash
# backup-daily.sh

TIMESTAMP=$(date +%Y%m%d)
BACKUP_DIR="/backups/daily/$TIMESTAMP"

echo "💿 Создание ежедневного полного бэкапа..."

# 1. Остановка приложений (graceful)
docker compose stop zero1-controller verifier scheduler worker

# 2. Полный снапшот Docker volumes
docker run --rm \
    -v ${PROJECT_NAME}_postgres_data:/source_db:ro \
    -v ${PROJECT_NAME}_minio_data:/source_minio:ro \
    -v ${PROJECT_NAME}_elasticsearch_data:/source_es:ro \
    -v $BACKUP_DIR:/backup \
    alpine tar czf /backup/full_system_$TIMESTAMP.tar.gz -C / .

# 3. Снапшот Elasticsearch indices
curl -X PUT "localhost:9200/_snapshot/backup_repo/snapshot_$TIMESTAMP?wait_for_completion=true"

# 4. Запись состояния системы
docker info > $BACKUP_DIR/system_state.txt
docker compose config > $BACKUP_DIR/compose_state.yml

# 5. Запуск приложений
docker compose start zero1-controller verifier scheduler worker

# 6. Шифрование и отправка
gpg --encrypt --recipient backup@zero1.io $BACKUP_DIR/full_system_$TIMESTAMP.tar.gz
aws s3 cp $BACKUP_DIR/full_system_$TIMESTAMP.tar.gz.gpg s3://zero1-dr-backups/daily/

# 7. Ротация (храним 30 дней)
find /backups/daily/* -mtime +30 -exec rm -rf {} \;

echo "✅ Ежедневный бэкап создан: $BACKUP_DIR"

I.4. Тестирование восстановления

I.4.1. Ежеквартальные учения DR
yaml

# dr-test-scenario.yml
scenarios:
  - name: "Полная потеря ЦОД"
    steps:
      - step: "Инициировать отказ"
        action: "simulate_datacenter_outage.sh"
        
      - step: "Активировать DR-сайт"
        action: "activate_dr_site.sh"
        timeout: "1h"
        
      - step: "Восстановить из бэкапа"
        action: "restore_from_backup.sh --point 24h"
        timeout: "2h"
        
      - step: "Проверить функциональность"
        action: "run_dr_validation.sh"
        metrics:
          - "api_response_time < 5s"
          - "data_integrity = 100%"
          - "no_data_loss"
          
  - name: "Коррупция базы данных"
    steps:
      - step: "Имитировать коррупцию"
        action: "corrupt_postgres_tables.sh"
        
      - step: "Обнаружить проблему"
        action: "detect_corruption.sh"
        timeout: "15m"
        
      - step: "Восстановить транзакции"
        action: "recover_from_wal.sh"
        timeout: "30m"
        
      - step: "Валидация"
        action: "validate_recovery.sh"

I.4.2. Метрики успешности восстановления
bash

#!/bin/bash
# dr-metrics.sh

echo "📊 Метрики восстановления:"

# 1. Время восстановления (RTO)
START_TIME=$(cat /tmp/dr_start_time)
END_TIME=$(date +%s)
RTO=$((END_TIME - START_TIME))

echo "🕒 RTO: $RTO секунд"
echo "   Целевое: < 4 часа (14400 секунд)"

if [ $RTO -lt 14400 ]; then
    echo "   ✅ В пределах нормы"
else
    echo "   ❌ Превышено"
fi

# 2. Потеря данных (RPO)
LAST_BACKUP_TIME=$(aws s3 ls s3://zero1-backups/daily/ | tail -1 | awk '{print $1" "$2}')
LAST_BACKUP_TS=$(date -d "$LAST_BACKUP_TIME" +%s)
FAILURE_TIME=$(cat /tmp/failure_time)
RPO=$((FAILURE_TIME - LAST_BACKUP_TS))

echo "💾 RPO: $RPO секунд"
echo "   Целевое: < 1 час (3600 секунд)"

if [ $RPO -lt 3600 ]; then
    echo "   ✅ В пределах нормы"
else
    echo "   ❌ Превышено"
fi

# 3. Целостность данных
DATA_INTEGRITY=$(check_data_integrity | grep "Integrity check passed" | wc -l)

echo "🔍 Целостность данных:"
if [ $DATA_INTEGRITY -eq 1 ]; then
    echo "   ✅ 100% целостность"
else
    echo "   ❌ Нарушена целостность"
fi

# 4. Доступность сервисов
SERVICES_TOTAL=$(docker compose ps --services | wc -l)
SERVICES_HEALTHY=$(docker compose ps --services --filter "status=running" | wc -l)
AVAILABILITY=$((SERVICES_HEALTHY * 100 / SERVICES_TOTAL))

echo "🌐 Доступность сервисов: $AVAILABILITY%"
echo "   Целевое: > 99.9%"

# 5. Генерация отчета
generate_dr_report $RTO $RPO $DATA_INTEGRITY $AVAILABILITY

Appendix J: Полная документация API
J.1. OpenAPI спецификация для основного контроллера
yaml

openapi: 3.0.3
info:
  title: Project ZERO-1 API
  version: 1.0.0
  description: API для управления автономными циклами верификации безопасности

servers:
  - url: http://localhost:8000/api/v1
    description: Локальный сервер разработки

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      
  schemas:
    Cycle:
      type: object
      required:
        - cycle_id
        - stig_id
        - target_device
      properties:
        cycle_id:
          type: string
          format: uuid
          example: "123e4567-e89b-12d3-a456-426614174000"
        stig_id:
          type: string
          example: "V-220668"
        target_device:
          $ref: '#/components/schemas/Device'
        status:
          type: string
          enum: [pending, running, completed, failed, rollback]
        created_at:
          type: string
          format: date-time
        artifacts:
          type: array
          items:
            $ref: '#/components/schemas/Artifact'
            
    Device:
      type: object
      properties:
        ip_address:
          type: string
          format: ipv4
        model:
          type: string
        credentials_ref:
          type: string
          description: Ссылка на секрет в Vault
          
    Artifact:
      type: object
      properties:
        id:
          type: string
          format: uuid
        type:
          type: string
          enum: [config, log, report, proof]
        storage_path:
          type: string
        sha256:
          type: string
        created_at:
          type: string
          format: date-time

paths:
  /cycles:
    post:
      summary: Запуск нового цикла верификации
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - stig_id
                - target_device
              properties:
                stig_id:
                  type: string
                target_device:
                  $ref: '#/components/schemas/Device'
                priority:
                  type: string
                  enum: [low, normal, high]
                  default: normal
      responses:
        '202':
          description: Цикл принят в обработку
          content:
            application/json:
              schema:
                type: object
                properties:
                  cycle_id:
                    type: string
                    format: uuid
                  status_url:
                    type: string
                    example: "/api/v1/cycles/{cycle_id}/status"
                    
  /cycles/{cycle_id}:
    get:
      summary: Получение информации о цикле
      parameters:
        - name: cycle_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Информация о цикле
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Cycle'
                
    delete:
      summary: Отмена цикла
      parameters:
        - name: cycle_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '202':
          description: Цикл отменен
          
  /cycles/{cycle_id}/status:
    get:
      summary: Получение статуса цикла
      parameters:
        - name: cycle_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Статус цикла
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                  current_stage:
                    type: string
                  progress:
                    type: number
                    minimum: 0
                    maximum: 100
                  estimated_completion:
                    type: string
                    format: date-time
                    
  /cycles/{cycle_id}/artifacts:
    get:
      summary: Получение артефактов цикла
      parameters:
        - name: cycle_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
        - name: artifact_type
          in: query
          schema:
            type: string
      responses:
        '200':
          description: Список артефактов
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Artifact'
                  
  /cycles/{cycle_id}/report:
    get:
      summary: Получение финального отчета
      parameters:
        - name: cycle_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Финальный отчет
          content:
            application/json:
              schema:
                type: object
                properties:
                  cycle_id:
                    type: string
                  verification_result:
                    type: boolean
                  merkle_root:
                    type: string
                  digital_signature:
                    type: string
                  artifacts_summary:
                    type: object
                    
  /health:
    get:
      summary: Проверка здоровья системы
      responses:
        '200':
          description: Состояние системы
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    enum: [healthy, degraded, unhealthy]
                  components:
                    type: object
                  uptime:
                    type: number
                  timestamp:
                    type: string
                    format: date-time
                    
  /metrics:
    get:
      summary: Метрики системы
      responses:
        '200':
          description: Метрики в формате Prometheus
          
  /admin/backup:
    post:
      summary: Инициировать ручной бэкап
      security:
        - bearerAuth: []
      responses:
        '202':
          description: Бэкап начат
          
  /admin/restore:
    post:
      summary: Восстановление из бэкапа
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - backup_id
              properties:
                backup_id:
                  type: string
                validate_only:
                  type: boolean
                  default: false
      responses:
        '202':
          description: Восстановление начато

J.2. API для ML-сервиса
yaml

openapi: 3.0.3
info:
  title: ZERO-1 ML Service API
  version: 1.0.0

paths:
  /analyze:
    post:
      summary: Анализ конфигурации устройства
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - device_config
                - stig_reference
              properties:
                device_config:
                  type: object
                  description: Структурированная конфигурация устройства
                stig_reference:
                  type: string
                  description: ID проверки STIG
                context:
                  type: object
                  description: Дополнительный контекст
      responses:
        '200':
          description: Результат анализа
          content:
            application/json:
              schema:
                type: object
                properties:
                  findings:
                    type: array
                    items:
                      $ref: '#/components/schemas/Finding'
                  attack_plan:
                    $ref: '#/components/schemas/AttackPlan'
                  confidence:
                    type: number
                    minimum: 0
                    maximum: 1
                    
  /remediate:
    post:
      summary: Генерация исправления
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - finding
                - device_config
              properties:
                finding:
                  $ref: '#/components/schemas/Finding'
                device_config:
                  type: object
                constraints:
                  type: object
                  properties:
                    allowed_commands:
                      type: array
                      items:
                        type: string
                    maintenance_window:
                      type: object
      responses:
        '200':
          description: Сгенерированное исправление
          content:
            application/json:
              schema:
                type: object
                properties:
                  remediation:
                    type: array
                    items:
                      type: string
                  validation_script:
                    type: string
                  rollback_commands:
                    type: array
                    items:
                      type: string
                  estimated_risk:
                    type: string
                    enum: [low, medium, high]

  /models:
    get:
      summary: Список доступных моделей
      responses:
        '200':
          description: Список моделей
          content:
            application/json:
              schema:
                type: object
                properties:
                  models:
                    type: array
                    items:
                      $ref: '#/components/schemas/ModelInfo'
                  
  /models/{model_id}/fine-tune:
    post:
      summary: Дообучение модели
      parameters:
        - name: model_id
          in: path
          required: true
          schema:
            type: string
      requestBody:
        required: true
        content:
          multipart/form-data:
            schema:
              type: object
              properties:
                dataset:
                  type: string
                  format: binary
                parameters:
                  type: object
      responses:
        '202':
          description: Дообучение начато
          
components:
  schemas:
    Finding:
      type: object
      properties:
        stig_id:
          type: string
        description:
          type: string
        severity:
          type: string
          enum: [low, medium, high, critical]
        affected_components:
          type: array
          items:
            type: string
            
    AttackPlan:
      type: object
      properties:
        steps:
          type: array
          items:
            $ref: '#/components/schemas/AttackStep'
        estimated_success_rate:
          type: number
          
    AttackStep:
      type: object
      properties:
        technique_id:
          type: string
        command:
          type: string
        prerequisites:
          type: array
          items:
            type: string
            
    ModelInfo:
      type: object
      properties:
        id:
          type: string
        name:
          type: string
        type:
          type: string
          enum: [analyzer, remediator]
        version:
          type: string
        accuracy:
          type: number

J.3. WebSocket API для реального времени
javascript

// WebSocket соединение для отслеживания прогресса
const ws = new WebSocket('ws://localhost:8000/ws/cycles');

ws.onopen = () => {
    // Подписка на события цикла
    ws.send(JSON.stringify({
        action: 'subscribe',
        cycle_id: '123e4567-e89b-12d3-a456-426614174000'
    }));
};

ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    
    switch(data.type) {
        case 'stage_update':
            console.log(`Стадия: ${data.stage}, Прогресс: ${data.progress}%`);
            break;
            
        case 'artifact_created':
            console.log(`Создан артефакт: ${data.artifact_type}`);
            break;
            
        case 'verification_result':
            console.log(`Результат верификации: ${data.success ? 'Успех' : 'Неудача'}`);
            break;
            
        case 'error':
            console.error(`Ошибка: ${data.message}`);
            break;
    }
};

// Доступные события
const WS_EVENTS = {
    CYCLE_STARTED: 'cycle_started',
    STAGE_STARTED: 'stage_started',
    STAGE_COMPLETED: 'stage_completed',
    ARTIFACT_CREATED: 'artifact_created',
    VERIFICATION_RESULT: 'verification_result',
    CYCLE_COMPLETED: 'cycle_completed',
    ERROR_OCCURRED: 'error'
};
