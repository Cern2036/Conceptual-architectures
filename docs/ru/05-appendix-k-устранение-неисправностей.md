---
title: 05 appendix k устранение неисправностей
lang: ru
---

Appendix K: Инструкции по устранению неисправностей (Troubleshooting Guide)
K.1. Общая методология диагностики
K.1.1. Диагностическое дерево решений
text

Проблема обнаружена → Проверить логи Kibana (http://localhost:5601) → 
→ Идентифицировать модуль-источник → 
→ Проверить метрики в Grafana → 
→ Свериться с таблицей распространенных проблем (K.2) → 
→ Применить специфичное решение

K.1.2. Ключевые команды диагностики
bash

# 1. Общий статус системы
./manage-system.sh status

# 2. Проверка логов последних ошибок
docker compose logs --tail=50 --timestamps | grep -E "(ERROR|FAILED|exception|timeout)"

# 3. Проверка здоровья всех сервисов
curl -s http://localhost:8000/api/v1/health | jq .
curl -s http://localhost:8080/health | jq .
curl -f http://localhost:9000/minio/health/live
curl -f http://localhost:9200/_cluster/health

# 4. Проверка дискового пространства
df -h /var/lib/docker/

# 5. Проверка использования GPU
nvidia-smi --query-gpu=utilization.gpu,memory.used --format=csv

K.2. Таблица распространенных проблем и решений
K.2.1. Проблемы развертывания лаборатории
Симптом	Возможная причина	Решение
Ошибка Lab initialization timeout в логах контроллера	EVE-NG API недоступен	1. Проверить статус EVE-NG: systemctl status eve-ng
2. Проверить сетевую связность: ping <eve-ng-ip>
3. Перезапустить EVE-NG: systemctl restart eve-ng
Terraform не может создать лабораторию	Недостаточно ресурсов на хосте EVE-NG	1. Проверить свободную память: free -h
2. Проверить доступные образы в EVE-NG
3. Увеличить лимиты памяти для виртуальных машин
Ansible не может подключиться к CSR1000v	Неправильные учетные данные или сетевые ACL	1. Проверить учетные данные в HashiCorp Vault
2. Проверить доступность CSR1000v по SSH с хоста управления
3. Проверить настройки firewall в EVE-NG
K.2.2. Проблемы модуля Attack Analyst
Симптом	Возможная причина	Решение
LLM возвращает пустой или некорректный JSON	Проблема с промпт-инжинирингом или моделью	1. Проверить логи ML-сервиса: docker compose logs ml-service
2. Проверить валидность входных данных
3. Перезапустить модель: curl -X POST http://ml-service:8080/models/reload
Netmiko таймаут при подключении к устройству	Проблемы с сетью или устройством	1. Проверить доступность устройства: nc -zv <device-ip> 22
2. Увеличить таймауты в конфигурации Netmiko
3. Проверить нагрузку на устройстве (CPU)
Парсер TextFSM не обрабатывает вывод CLI	Несоответствие шаблона	1. Проверить актуальность шаблонов NTC
2. Добавить raw вывод в лог для анализа
3. Использовать fallback-регулярные выражения
K.2.3. Проблемы модуля Attack Executor
Симптом	Возможная причина	Решение
Ping не проходит, хотя уязвимость существует	Проблемы с сетевым стеком в EVE-NG	1. Проверить ARP таблицу на CSR1000v
2. Проверить VLAN назначения на коммутаторе
3. Проверить firewall правила на Ubuntu VM
SSH подключение к атакующему хосту падает	Проблемы с ресурсами виртуальной машины	1. Увеличить память для VM в EVE-NG
2. Проверить нагрузку на хосте EVE-NG
3. Пересоздать VM из шаблона
K.2.4. Проблемы модуля Remediation
Симптом	Возможная причина	Решение
Сгенерированные команды вызывают ошибку на устройстве	Синтаксическая ошибка или неподдерживаемая команда	1. Проверить версию IOS и синтаксис команд
2. Включить dry-run режим для валидации
3. Использовать show parser dump для отладки
Применение патча вызывает потерю управления	Конфликтующая конфигурация	1. Автоматический откат должен сработать (проверить rollback_logs)
2. Проверить снапшот устройства в EVE-NG
3. Восстановить из бэкапа вручную
K.2.5. Проблемы производительности
Симптом	Возможная причина	Решение
Время цикла превышает 120 минут	Загрузка GPU или сетевые задержки	1. Проверить утилизацию GPU: nvidia-smi
2. Оптимизировать промпты для уменьшения времени инференса
3. Кэшировать результаты анализа для идентичных конфигураций
Высокая загрузка CPU на хосте	Утечка памяти или бесконечные циклы	1. Проверить контейнеры: docker stats
2. Перезапустить проблемные сервисы
3. Увеличить ресурсы хоста
K.3. Пошаговые процедуры для критических сбоев
K.3.1. Полная потеря связи с EVE-NG
bash

#!/bin/bash
# recovery-eve-ng.sh

echo "🚨 Восстановление связи с EVE-NG"

# 1. Проверка базовой связности
if ! ping -c 3 ${EVE_NG_IP}; then
    echo "❌ EVE-NG недоступен по IP"
    escalate_to_network_team
    exit 1
fi

# 2. Проверка сервисов EVE-NG
if ! curl -f http://${EVE_NG_IP}:8080/api/status; then
    echo "🔄 Перезапуск EVE-NG..."
    ssh root@${EVE_NG_IP} "systemctl restart eve-ng"
    sleep 60
fi

# 3. Проверка лаборатории
if ! curl -f http://${EVE_NG_IP}:8080/api/labs/zero1-mvp; then
    echo "🔄 Восстановление лаборатории из шаблона..."
    curl -X POST http://${EVE_NG_IP}:8080/api/labs \
        -d '{"name": "zero1-mvp", "template": "zero1-base"}'
fi

# 4. Валидация
if curl -f http://${EVE_NG_IP}:8080/api/labs/zero1-mvp/nodes; then
    echo "✅ EVE-NG восстановлен"
else
    echo "❌ Не удалось восстановить EVE-NG"
    escalate_to_level_3
fi

K.3.2. Коррупция базы данных
bash

#!/bin/bash
# recovery-database-corruption.sh

echo "🗄️  Восстановление после коррупции БД"

# 1. Остановка записей в БД
docker compose stop zero1-controller scheduler worker

# 2. Проверка целостности
if ! docker compose exec postgres pg_catalog.pg_checkdb(); then
    echo "⚠️  Обнаружена коррупция БД"
    
    # 3. Восстановление из последнего снапшота
    LAST_BACKUP=$(aws s3 ls s3://zero1-backups/postgres/ | tail -1 | awk '{print $4}')
    aws s3 cp s3://zero1-backups/postgres/${LAST_BACKUP} /tmp/recovery.sql.gz
    
    # 4. Восстановление
    gunzip -c /tmp/recovery.sql.gz | docker compose exec -T postgres psql -U zero1
    
    # 5. Проверка
    docker compose exec postgres psql -U zero1 -c "SELECT COUNT(*) FROM test_cycles;"
fi

# 6. Запуск сервисов
docker compose start zero1-controller scheduler worker

K.4. Диагностические утилиты
K.4.1. Скрипт диагностики системы
python

#!/usr/bin/env python3
"""
zero1-diagnostic.py - Комплексная диагностика системы
"""

import subprocess
import json
import requests
from datetime import datetime

def check_docker_services():
    """Проверка состояния Docker сервисов"""
    result = subprocess.run(
        ["docker", "compose", "ps", "--format", "json"],
        capture_output=True, text=True
    )
    
    services = json.loads(result.stdout)
    unhealthy = [s for s in services if "unhealthy" in s.get("Status", "")]
    
    return {
        "total": len(services),
        "unhealthy": len(unhealthy),
        "details": unhealthy
    }

def check_api_endpoints():
    """Проверка доступности API endpoints"""
    endpoints = [
        ("Контроллер", "http://localhost:8000/api/v1/health"),
        ("ML-сервис", "http://localhost:8080/health"),
        ("MinIO", "http://localhost:9000/minio/health/live"),
        ("Elasticsearch", "http://localhost:9200/_cluster/health"),
    ]
    
    results = []
    for name, url in endpoints:
        try:
            response = requests.get(url, timeout=5)
            results.append({
                "service": name,
                "status": "healthy" if response.status_code == 200 else "unhealthy",
                "response_time": response.elapsed.total_seconds()
            })
        except Exception as e:
            results.append({
                "service": name,
                "status": "unreachable",
                "error": str(e)
            })
    
    return results

def check_resource_usage():
    """Проверка использования ресурсов"""
    # Проверка диска
    disk = subprocess.run(
        ["df", "-h", "/var/lib/docker"],
        capture_output=True, text=True
    ).stdout
    
    # Проверка памяти
    memory = subprocess.run(
        ["free", "-h"],
        capture_output=True, text=True
    ).stdout
    
    return {"disk": disk, "memory": memory}

def generate_report():
    """Генерация диагностического отчета"""
    report = {
        "timestamp": datetime.now().isoformat(),
        "docker_services": check_docker_services(),
        "api_endpoints": check_api_endpoints(),
        "resources": check_resource_usage(),
        "recommendations": []
    }
    
    # Анализ и рекомендации
    if report["docker_services"]["unhealthy"] > 0:
        report["recommendations"].append(
            "Перезапустить неисправные сервисы: ./manage-system.sh restart"
        )
    
    unhealthy_apis = [e for e in report["api_endpoints"] if e["status"] != "healthy"]
    if unhealthy_apis:
        report["recommendations"].append(
            f"Восстановить API endpoints: {[e['service'] for e in unhealthy_apis]}"
        )
    
    return report

if __name__ == "__main__":
    report = generate_report()
    print(json.dumps(report, indent=2, ensure_ascii=False))

K.4.2. Мониторинг в реальном времени
bash

#!/bin/bash
# live-monitor.sh

watch -n 5 '
echo "=== Project ZERO-1 Live Monitor ==="
echo "Время: $(date)"
echo ""
echo "1. Состояние сервисов:"
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"
echo ""
echo "2. Использование ресурсов:"
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
echo ""
echo "3. Активные циклы:"
docker compose exec -T postgres psql -U zero1 -c "SELECT cycle_id, overall_status, start_time FROM test_cycles WHERE overall_status = '"'"'running'"'"';"
'

Appendix L: Политики безопасности и compliance (Security Policies)
L.1. Базовые принципы безопасности
L.1.1. Принцип наименьших привилегий (Least Privilege)
yaml

# zero1-rbac-policies.yml
role_definitions:
  zero1_operator:
    permissions:
      - start_cycle
      - view_reports
      - system_status
    denied:
      - modify_secrets
      - access_raw_data
      - bypass_approval
  
  zero1_auditor:
    permissions:
      - view_all_reports
      - verify_signatures
      - export_evidence
    denied:
      - execute_cycles
      - modify_config
  
  zero1_admin:
    permissions:
      - full_system_access
    constraints:
      - mfa_required: true
      - ip_whitelist: ["10.0.0.0/8"]
      - time_restriction: "08:00-18:00"

L.1.2. Defense in Depth
text

Уровень 1: Сетевая сегментация (EVE-NG изолированная сеть)
Уровень 2: Аутентификация (JWT + MFA)
Уровень 3: Авторизация (RBAC)
Уровень 4: Шифрование (TLS 1.3, encrypted volumes)
Уровень 5: Аудит (полная traceability, immutable logs)
Уровень 6: Криптографическая верификация (Merkle trees, hardware signing)

L.2. Политики обработки данных
L.2.1. Классификация данных
yaml

data_classes:
  confidential:
    - network_device_configs
    - security_findings
    - credentials
    - digital_signatures
    retention: 7 лет
    encryption: обязательное (AES-256-GCM)
    access_logging: полное
    
  internal:
    - system_logs
    - performance_metrics
    - ml_model_weights
    retention: 1 год
    encryption: рекомендованное
    access_logging: агрегированное
    
  public:
    - anonymized_statistics
    - compliance_reports
    - system_status
    retention: 5 лет
    encryption: не требуется
    access_logging: минимальное

L.2.2. Политика хранения и удаления
bash

#!/bin/bash
# data-retention-policy.sh

# Автоматическое удаление старых данных
RETENTION_DAYS=365

echo "🧹 Применение политики хранения данных..."

# 1. Удаление старых циклов из БД
docker compose exec -T postgres psql -U zero1 -c "
    DELETE FROM test_cycles 
    WHERE created_at < NOW() - INTERVAL '${RETENTION_DAYS} days'
    AND overall_status != 'running';
"

# 2. Удаление соответствующих артефактов из MinIO
CYCLES_TO_DELETE=$(docker compose exec -T postgres psql -U zero1 -t -c "
    SELECT cycle_id FROM test_cycles 
    WHERE created_at < NOW() - INTERVAL '${RETENTION_DAYS} days';
")

for CYCLE in $CYCLES_TO_DELETE; do
    docker compose run --rm mc rm --recursive --force minio/zero1-artifacts/cycles/${CYCLE}
done

# 3. Rotate логов Elasticsearch
curl -X POST "http://localhost:9200/_ilm/policy/zero1_logs_policy/_execute"

L.3. Соответствие стандартам
L.3.1. DISA STIG Compliance
yaml

stig_compliance_framework:
  controls:
    - id: "V-220668"
      description: "Default VLAN not assigned to access ports"
      implementation:
        automated_detection: "Attack Analyst module"
        automated_remediation: "Remediation Synthesizer module"
        verification: "Verification module"
        evidence: "Digital signature in final report"
    
    - id: "V-220543"
      description: "Password encryption required"
      status: "planned_for_phase_1.5"
      
    - id: "V-220521"
      description: "SSH version 2 required"
      status: "planned_for_phase_1.5"

  validation_procedures:
    quarterly_audit:
      frequency: "каждые 3 месяца"
      scope: "все проверки STIG"
      method: "автоматический + ручная выборочная проверка"
      responsible: "Chief Security Officer"
    
    continuous_compliance:
      frequency: "непрерывно"
      scope: "активные проверки"
      method: "автоматический мониторинг"
      alerts: "Kibana + Slack notifications"

L.3.2. NIST Cybersecurity Framework
yaml

nist_csf_mapping:
  identify:
    - asset_management: "Реестр всех сетевых устройств"
    - risk_assessment: "Оценка рисков на основе STIG findings"
    
  protect:
    - access_control: "RBAC, MFA, network segmentation"
    - data_security: "Encryption at rest and in transit"
    - maintenance: "Automated patching through remediation"
    
  detect:
    - anomalies_and_events: "Elasticsearch monitoring and alerting"
    - security_continuous_monitoring: "Real-time verification cycles"
    
  respond:
    - response_planning: "Disaster recovery procedures (Appendix I)"
    - communications: "Alerting system (Slack, email)"
    
  recover:
    - recovery_planning: "Backup and restore procedures"
    - improvements: "Lessons learned from failed cycles"

L.4. Криптографические политики
L.4.1. Управление ключами
yaml

key_management_policy:
  hardware_security_modules:
    primary: "YubiKey PIV (FIPS 140-2 Level 2)"
    backup: "Thales HSM (off-site)"
    
  key_generation:
    algorithm: "RSA 4096 или ECC P-384"
    source: "HSM hardware random generator"
    ceremony: "два оператора + witness"
    
  key_storage:
    private_keys: "никогда не покидают HSM"
    public_keys: "хранятся в HashiCorp Vault с ACL"
    
  key_rotation:
    frequency: "ежегодно или при компрометации"
    procedure: "генерировать новую пару, переподписать архив"
    
  revocation:
    conditions:
      - suspected_compromise
      - employee_termination
      - key_expiration
    mechanism: "CRL + OCSP stapling"

L.4.2. Политика подписания
python

# signing-policy.py
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import padding
from datetime import datetime, timedelta

class SigningPolicy:
    """Политика криптографического подписания"""
    
    VALID_ALGORITHMS = ["RSASSA-PSS", "ECDSA-SHA384"]
    MIN_KEY_SIZE = {"RSA": 3072, "ECC": 256}
    MAX_SIGNATURE_AGE = timedelta(days=365)
    
    @classmethod
    def validate_signature_context(cls, artifact_type, signer_role):
        """Валидация контекста подписания"""
        requirements = {
            "final_report": {
                "allowed_signers": ["zero1_system", "auditor"],
                "required_timestamp": True,
                "requires_counter_signature": True
            },
            "remediation_patch": {
                "allowed_signers": ["zero1_system"],
                "required_timestamp": True,
                "requires_approval": True
            }
        }
        
        if artifact_type not in requirements:
            raise ValueError(f"Unsupported artifact type: {artifact_type}")
        
        req = requirements[artifact_type]
        if signer_role not in req["allowed_signers"]:
            raise ValueError(f"Signer {signer_role} not allowed for {artifact_type}")
        
        return req

L.5. Политика аудита и мониторинга
L.5.1. Неизменяемое логирование
bash

#!/bin/bash
# immutable-logs-setup.sh

# Настройка аудита Docker
cat > /etc/docker/daemon.json << EOF
{
  "log-driver": "local",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "compress": "true"
  },
  "live-restore": true
}
EOF

# Настройка аудита системы
echo "Setting up immutable audit logs..."
auditctl -w /var/log/zero1/ -p wa -k zero1_audit
auditctl -w /opt/zero1/config/ -p wa -k zero1_config

# Настройка журналирования в WORM storage
echo "Configuring WORM storage for audit logs..."
mkdir -p /mnt/worm-storage/zero1-audit
chattr +i /mnt/worm-storage/zero1-audit

L.5.2. Политика реагирования на инциденты
yaml

incident_response_policy:
  classification:
    severity_levels:
      critical:
        response_time: "15 минут"
        escalation: "CISO + Legal"
        examples:
          - "компрометация корневого ключа"
          - "несанкционированное изменение конфигурации"
      
      high:
        response_time: "1 час"
        escalation: "Security Team Lead"
        examples:
          - "обход аутентификации"
          - "утечка конфиденциальных данных"
  
  procedures:
    containment:
      - "изоляция затронутых систем"
      - "отключение компрометированных учетных записей"
      - "активация резервных систем"
    
    eradication:
      - "патчинг уязвимостей"
      - "удаление вредоносного кода"
      - "смена всех затронутых ключей"
    
    recovery:
      - "восстановление из доверенных бэкапов"
      - "валидация целостности системы"
      - "постепенное возвращение к нормальной работе"
    
    lessons_learned:
      frequency: "после каждого инцидента уровня critical или high"
      deliverables:
        - "отчет об инциденте"
        - "обновление процедур"
        - "обновление training материалов"

Appendix M: Интеграционные тесты и тестовые сценарии
M.1. Архитектура тестирования
M.1.1. Тестовая пирамида
text

        ↗ E2E тесты (10%) - Полные циклы
       ↗ Интеграционные тесты (20%) - Взаимодействие модулей
     ↗ Компонентные тесты (30%) - Отдельные модули
   ↗ Юнит тесты (40%) - Функции и классы

M.1.2. Тестовое окружение
yaml

# docker-compose.test.yml
services:
  test-postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: zero1_test
  
  test-minio:
    image: minio/minio
    command: server /data --console-address ":9001"
  
  test-elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
  
  mock-eve-ng:
    image: python:3.11-slim
    command: python -m http.server 8080
    volumes:
      - ./test_data/mock_responses:/mock_responses
  
  test-runner:
    build:
      context: .
      dockerfile: Dockerfile.test
    depends_on:
      - test-postgres
      - test-minio
      - test-elasticsearch
      - mock-eve-ng

M.2. Интеграционные тесты
M.2.1. Тест полного цикла (End-to-End)
python

# tests/test_full_cycle.py
import pytest
import json
from datetime import datetime
from zero1_controller import CycleController
from zero1_orchestrator import Orchestrator

class TestFullCycle:
    """Тестирование полного цикла от обнаружения до верификации"""
    
    @pytest.fixture
    def test_environment(self):
        """Подготовка тестового окружения"""
        return {
            "device_ip": "192.168.1.1",
            "device_type": "cisco_csr1000v",
            "stig_id": "V-220668",
            "expected_vulnerability": "interface Gi2 in VLAN 1"
        }
    
    def test_cycle_happy_path(self, test_environment):
        """Тест успешного выполнения полного цикла"""
        
        # 1. Инициализация контроллера
        controller = CycleController(test_mode=True)
        
        # 2. Запуск цикла
        cycle_id = controller.start_cycle(
            stig_id=test_environment["stig_id"],
            target_device=test_environment["device_ip"]
        )
        
        # 3. Мониторинг выполнения
        max_wait_time = 7200  # 120 минут
        start_time = datetime.now()
        
        while (datetime.now() - start_time).seconds < max_wait_time:
            status = controller.get_cycle_status(cycle_id)
            
            if status["overall_status"] == "completed":
                # 4. Проверка результатов
                report = controller.get_cycle_report(cycle_id)
                
                # Проверки
                assert report["verification_result"] == True
                assert report["digital_signature"] is not None
                assert len(report["artifacts"]) >= 9  # A1-A9
                assert report["merkle_root_verified"] == True
                
                return  # Успех
            
            elif status["overall_status"] == "failed":
                pytest.fail(f"Cycle failed: {status.get('error_message')}")
        
        pytest.fail("Cycle timeout")
    
    def test_rollback_scenario(self):
        """Тест отката при ошибке применения патча"""
        
        # 1. Внедрение сбоя
        sabotaged_device = MockDevice()
        sabotaged_device.set_failure_mode("config_apply_failure")
        
        # 2. Запуск цикла
        controller = CycleController()
        cycle_id = controller.start_cycle(
            stig_id="V-220668",
            target_device=sabotaged_device
        )
        
        # 3. Проверка отката
        status = controller.wait_for_completion(cycle_id, timeout=300)
        
        assert status["overall_status"] == "rollback"
        assert sabotaged_device.config_restored_to_backup() == True
        assert controller.rollback_logs_contain(cycle_id, "Remediation Applier") == True
    
    def test_llm_fallback_mechanism(self):
        """Тест работы fallback при сбое LLM"""
        
        # 1. Отключение ML-сервиса
        ml_service.stop()
        
        # 2. Запуск цикла
        controller = CycleController()
        cycle_id = controller.start_cycle(...)
        
        # 3. Проверка использования fallback-правил
        artifacts = controller.get_artifacts(cycle_id)
        analysis_artifact = artifacts.get_by_type("llm_analysis")
        
        # Должен использоваться rule-based анализатор
        assert analysis_artifact["generator"] == "rule_based_fallback"
        assert analysis_artifact["contains_finding"] == True
        
        # 4. Проверка завершения цикла
        status = controller.wait_for_completion(cycle_id)
        assert status["overall_status"] == "completed"

M.2.2. Тесты взаимодействия модулей
python

# tests/test_module_integration.py
class TestModuleIntegration:
    
    def test_attack_analyst_to_executor_flow(self):
        """Тест передачи плана атаки от Analyst к Executor"""
        
        # 1. Analyst генерирует план
        analyst = AttackAnalyst()
        config = get_test_device_config()
        analysis = analyst.analyze(config, stig_id="V-220668")
        
        # 2. Проверка структуры плана
        assert "attack_plan" in analysis
        assert "steps" in analysis["attack_plan"]
        assert len(analysis["attack_plan"]["steps"]) > 0
        
        # 3. Executor выполняет план
        executor = AttackExecutor()
        result = executor.execute(analysis["attack_plan"])
        
        # 4. Проверка результатов
        assert "attack_successful" in result
        assert "evidence" in result
        assert "timestamp" in result
        
        # 5. Проверка соответствия ожиданиям
        if analysis["findings"][0]["severity"] == "medium":
            assert result["attack_successful"] == True
    
    def test_remediation_synthesis_and_application(self):
        """Тест синтеза и применения исправления"""
        
        # 1. Подготовка данных
        vulnerability = {
            "stig_id": "V-220668",
            "description": "Interface Gi2 in default VLAN 1",
            "affected_component": "GigabitEthernet2"
        }
        
        attack_report = {
            "attack_successful": True,
            "evidence": "ping succeeded to 192.168.1.100"
        }
        
        # 2. Синтез исправления
        synthesizer = RemediationSynthesizer()
        patch = synthesizer.generate(
            vulnerability=vulnerability,
            context=attack_report
        )
        
        # 3. Валидация синтаксиса
        assert patch.validate_syntax() == True
        assert "interface GigabitEthernet2" in patch.commands
        assert "switchport access vlan 100" in patch.commands
        
        # 4. Применение (dry-run)
        applier = RemediationApplier(dry_run=True)
        apply_result = applier.apply(patch, target_device)
        
        # 5. Проверка
        assert apply_result["success"] == True
        assert apply_result["validation_passed"] == True
        assert apply_result["rollback_prepared"] == True
    
    def test_evidence_chain_integrity(self):
        """Тест целостности цепочки доказательств"""
        
        # 1. Сбор артефактов тестового цикла
        artifacts = [
            Artifact(type="config", data=initial_config, hash=sha256(initial_config)),
            Artifact(type="analysis", data=analysis_report, hash=sha256(analysis_report)),
            Artifact(type="attack", data=attack_result, hash=sha256(attack_result)),
            Artifact(type="remediation", data=patch, hash=sha256(patch)),
            Artifact(type="verification", data=verification, hash=sha256(verification))
        ]
        
        # 2. Построение Merkle Tree
        merkle_builder = MerkleTreeBuilder()
        tree = merkle_builder.build(artifacts)
        
        # 3. Проверка целостности
        for artifact in artifacts:
            proof = tree.get_proof(artifact.hash)
            assert merkle_builder.verify_proof(
                leaf_hash=artifact.hash,
                proof=proof,
                root_hash=tree.root
            ) == True
        
        # 4. Подписание и верификация
        signer = DigitalSigner()
        signature = signer.sign(tree.root)
        
        assert signer.verify(tree.root, signature) == True

M.3. Тестовые сценарии
M.3.1. Таблица тестовых сценариев
yaml

test_scenarios:
  - id: "SCENARIO-001"
    name: "Успешное обнаружение и исправление уязвимости"
    preconditions:
      - "Устройство имеет порт в VLAN 1"
      - "Целевой хост доступен в VLAN 1"
    steps:
      - "Запуск цикла с STIG V-220668"
      - "Ожидание завершения"
    expected_results:
      - "Цикл завершен со статусом completed"
      - "Отчет содержит цифровую подпись"
      - "Порт перемещен в VLAN 100"
      - "Повторная атака неуспешна"
    
  - id: "SCENARIO-002"
    name: "Обработка ложноположительного срабатывания"
    preconditions:
      - "Устройство НЕ имеет портов в VLAN 1"
      - "LLM настроен на гиперчувствительность"
    steps:
      - "Запуск цикла с STIG V-220668"
      - "Мониторинг логов"
    expected_results:
      - "Attack Analyst не обнаруживает уязвимость"
      - "Цикл завершается досрочно с appropriate статусом"
      - "В логах запись о проверке и отсутствии findings"
    
  - id: "SCENARIO-003"
    name: "Восстановление после сетевого сбоя"
    preconditions:
      - "Цикл запущен"
      - "Сетевое соединение с устройством нестабильно"
    steps:
      - "Имитация обрыва сети во время анализа"
      - "Восстановление сети через 60 секунд"
    expected_results:
      - "Система выполняет retry (до 3 попыток)"
      - "При неудаче - корректный откат"
      - "Логи содержат записи о таймаутах и retry"
    
  - id: "SCENARIO-004"
    name: "Обработка отказа ML-сервиса"
    preconditions:
      - "ML-сервис работает нестабильно"
      - "Fallback правила настроены"
    steps:
      - "Остановка ML-сервиса во время анализа"
      - "Мониторинг реакции системы"
    expected_results:
      - "Система переключается на rule-based анализатор"
      - "Цикл завершается успешно (возможно с degraded точностью)"
      - "Логи содержат warning о fallback активации"
    
  - id: "SCENARIO-005"
    name: "Проверка производительности под нагрузкой"
    preconditions:
      - "Система развернута в production-конфигурации"
      - "10 виртуальных устройств готовы к тестированию"
    steps:
      - "Параллельный запуск 10 циклов"
      - "Мониторинг метрик в течение 2 часов"
    expected_results:
      - "Все циклы завершаются в течение 120 минут"
      - "Использование GPU не превышает 80%"
      - "Не более 5% циклов требуют отката"
      - "Среднее время цикла < 90 минут"

M.3.2. Скрипт выполнения тестовых сценариев
python

#!/usr/bin/env python3
"""
test-scenario-runner.py - Выполнение тестовых сценариев
"""

import yaml
import asyncio
from dataclasses import dataclass
from typing import Dict, List
from enum import Enum

class TestStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    PASSED = "passed"
    FAILED = "failed"
    SKIPPED = "skipped"

@dataclass
class TestResult:
    scenario_id: str
    status: TestStatus
    duration: float
    logs: List[str]
    artifacts: Dict

class TestScenarioRunner:
    def __init__(self, scenarios_file: str):
        with open(scenarios_file, 'r') as f:
            self.scenarios = yaml.safe_load(f)['test_scenarios']
        
        self.results = []
    
    async def run_scenario(self, scenario: Dict) -> TestResult:
        """Выполнение одного тестового сценария"""
        result = TestResult(
            scenario_id=scenario['id'],
            status=TestStatus.RUNNING,
            duration=0.0,
            logs=[],
            artifacts={}
        )
        
        try:
            # Подготовка окружения
            await self.setup_environment(scenario['preconditions'])
            
            # Выполнение шагов
            start_time = asyncio.get_event_loop().time()
            
            for step in scenario['steps']:
                await self.execute_step(step, result)
            
            result.duration = asyncio.get_event_loop().time() - start_time
            
            # Проверка ожидаемых результатов
            await self.verify_results(scenario['expected_results'], result)
            
            result.status = TestStatus.PASSED
            
        except Exception as e:
            result.status = TestStatus.FAILED
            result.logs.append(f"ERROR: {str(e)}")
        
        finally:
            await self.cleanup()
        
        return result
    
    async def run_all_scenarios(self):
        """Выполнение всех сценариев"""
        print(f"🚀 Запуск {len(self.scenarios)} тестовых сценариев")
        
        for scenario in self.scenarios:
            print(f"\n📋 Выполнение сценария: {scenario['name']}")
            
            result = await self.run_scenario(scenario)
            self.results.append(result)
            
            print(f"   Статус: {result.status.value}")
            print(f"   Длительность: {result.duration:.2f} секунд")
        
        # Генерация отчета
        self.generate_report()
    
    def generate_report(self):
        """Генерация итогового отчета"""
        passed = sum(1 for r in self.results if r.status == TestStatus.PASSED)
        failed = sum(1 for r in self.results if r.status == TestStatus.FAILED)
        
        report = {
            "summary": {
                "total": len(self.results),
                "passed": passed,
                "failed": failed,
                "success_rate": (passed / len(self.results)) * 100
            },
            "detailed_results": [
                {
                    "scenario_id": r.scenario_id,
                    "status": r.status.value,
                    "duration": r.duration,
                    "log_count": len(r.logs)
                }
                for r in self.results
            ]
        }
        
        # Сохранение отчета
        with open('/tmp/test_execution_report.json', 'w') as f:
            json.dump(report, f, indent=2)
        
        print(f"\n📊 Итоговый отчет:")
        print(f"   Успешно: {passed}/{len(self.results)}")
        print(f"   Успешность: {report['summary']['success_rate']:.1f}%")
        print(f"   Отчет сохранен: /tmp/test_execution_report.json")

async def main():
    runner = TestScenarioRunner('test_scenarios.yml')
    await runner.run_all_scenarios()

if __name__ == "__main__":
    asyncio.run(main())

M.4. Непрерывная интеграция и тестирование
M.4.1. Конфигурация GitHub Actions
yaml

# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run unit tests
        run: |
          docker compose -f docker-compose.test.yml run --rm test-runner \
            pytest tests/unit/ -v --cov=zero1 --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
  
  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v3
      
      - name: Run integration tests
        run: |
          docker compose -f docker-compose.test.yml up -d
          docker compose -f docker-compose.test.yml run --rm test-runner \
            pytest tests/integration/ -v
      
      - name: Test report
        if: always()
        run: |
          cat /tmp/test_execution_report.json
  
  security-scan:
    runs-on: ubuntu-latest
    needs: integration-tests
    steps:
      - uses: actions/checkout@v3
      
      - name: Run security scan
        run: |
          docker scan zero1-controller:latest --severity high
          docker scan zero1-ml-service:latest --severity high
      
      - name: STIG compliance check
        run: |
          python scripts/compliance_check.py --profile stig

  performance-tests:
    runs-on: [self-hosted, gpu]
    needs: security-scan
    steps:
      - uses: actions/checkout@v3
      
      - name: Run performance tests
        run: |
          locust -f tests/performance/locustfile.py \
            --host=http://localhost:8000 \
            --users=10 \
            --spawn-rate=1 \
            --run-time=1h \
            --csv=performance_results
      
      - name: Upload performance report
        uses: actions/upload-artifact@v3
        with:
          name: performance-report
          path: performance_results*.csv

M.4.2. Качество кода и стандарты
yaml

# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-added-large-files
  
  - repo: https://github.com/psf/black
    rev: 23.3.0
    hooks:
      - id: black
        language_version: python3.11
  
  - repo: https://github.com/PyCQA/flake8
    rev: 6.0.0
    hooks:
      - id: flake8
        additional_dependencies: [flake8-docstrings]
  
  - repo: https://github.com/PyCQA/isort
    rev: 5.12.0
    hooks:
      - id: isort
  
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.3.0
    hooks:
      - id: mypy
        additional_dependencies: [types-requests, types-PyYAML]
  
  - repo: https://github.com/commitizen-tools/commitizen
    rev: v3.5.3
    hooks:
      - id: commitizen
        stages: [commit-msg]

Итоговый анализ созданных разделов:
1. Полнота покрытия

Созданные разделы дополняют существующую документацию, обеспечивая:

    Операционную готовность через troubleshooting guide

    Соответствие регуляторным требованиям через security policies

    Надежность и качество через интеграционные тесты

2. Согласованность с архитектурой

Все разделы:

    Ссылаются на существующие компоненты системы

    Используют определенные в ТЗ технологии и инструменты

    Соответствуют заявленным требованиям и ограничениям

3. Практическая применимость

    Troubleshooting guide содержит конкретные команды и процедуры

    Security policies реализуемы и проверяемы

    Тестовые сценарии покрывают все критические пути системы

4. Масштабируемость

    Диагностические процедуры можно расширять для новых модулей

    Политики безопасности адаптируются под новые compliance требования

    Тестовая инфраструктура поддерживает добавление новых сценариев


