# Graduation-project

Полный цикл DevOps для интернет-магазина на базе микросервисов GoogleCloudPlatform/microservices-demo. Проект охватывает контейнеризацию, CI/CD, облачную инфраструктуру (GCP), безопасность секретов, оркестрацию Kubernetes и мониторинг.



## Содержание
- [Цель](#-цель)
- [Технологический стек](#-технологический-стек)
- [Архитектурная схема](#-архитектурная-схема)
- [Реализация](#-реализация)
  - [Шаг 0. Локальная разработка](#шаг-0-локальная-разработка-и-docker-compose)
  - [Шаг 1. CI (сборка и сканирование)](#шаг-1-ci-сборка-и-сканирование)
  - [Шаг 2. Инфраструктура (Terraform)](#шаг-2-инфраструктура-terraform)
  - [Шаг 3. Безопасность секретов](#шаг-3-безопасность-секретов)
  - [Шаг 4. Helm-чарт и ручной деплой](#шаг-4-helm-чарт-и-ручной-деплой)
  - [Шаг 5. CD (автоматический деплой)](#шаг-5-cd-автоматический-деплой)
  - [Шаг 6. Мониторинг и логирование](#шаг-6-мониторинг-и-логирование)
- [Ограничения и решения](#-ограничения-и-решения)
- [Быстрый старт](#-быстрый-старт)
- [Скриншоты](#-скриншоты)
- [Автор](#-автор)

## Цель
Показать компетенции Junior DevOps-инженера в реальном проекте: Infrastructure as Code, настройка наблюдаемости, автоматическая сборка и деплой.

## Технологический стек
| Область            | Инструменты                                           |
|--------------------|-------------------------------------------------------|
| Облако             | GCP (GKE Autopilot, Cloud KMS, Secret Manager, GCS)   |
| IaC                | Terraform                                             |
| Контейнеры         | Docker, GitHub Container Registry (GHCR)              |
| Оркестрация        | Kubernetes, Helm                                      |
| CI/CD              | GitHub Actions (CI + CD)                              |
| Безопасность       | External Secrets Operator, Workload Identity, Trivy   |
| Мониторинг         | Grafana, Loki (Prometheus запланирован)               |
| Языки сервисов     | Go, C# (.NET), Node.js, Python                        |
| Хранилище          | Redis (StatefulSet с PersistentVolume)                |

## Архитектурная схема
![Architecture](docs/architecture.png)  
*Полная схема развёртывания: от пуша в Git до отображения магазина.*

## Реализация

### Шаг 0. Локальная разработка и Docker Compose
- Взяты микросервисы `frontend`, `productcatalogservice`, `cartservice`, `currencyservice`.
- Написан `docker-compose.yml`, поднимающий все сервисы + Redis локально.
- Проверена работа корзины и каталога.

### Шаг 1. CI (сборка и сканирование)
- Workflow GitHub Actions собирает Docker-образы и публикует в GHCR.
- Интегрирован **Trivy** – поиск уязвимостей (CRITICAL/HIGH) с выводом SARIF-отчётов.
- Используется стратегия matrix для параллельной сборки.

### Шаг 2. Инфраструктура (Terraform)
- Созданы VPC, приватная подсеть, GKE Autopilot с Workload Identity.
- Настроены Cloud KMS (кольцо + ключ) и Secret Manager.
- Удалённое хранение `.tfstate` в GCS-бакете.

### Шаг 3. Безопасность секретов
- Установлен **External Secrets Operator**.
- Настроена связка Workload Identity → GCP Service Account.
- Пароль Redis хранится в Secret Manager и синхронизируется в Kubernetes Secret.

### Шаг 4. Helm-чарт и ручной деплой
- Написан Helm-чарт для всех микросервисов (Deployment / Service / Ingress).
- Redis развёрнут как **StatefulSet** с PersistentVolumeClaim.
- Использован GKE Ingress (встроенный HTTP-балансировщик) из-за ограничений Autopilot.
- Применены минимальные ресурсы (requests: 10m CPU, 16Mi memory) для вписывания в триальные квоты.

### Шаг 5. CD (автоматический деплой)
- Workflow CD запускает `helm upgrade` при пуше в `main`.
- Добавлен Health Check (проверка HTTP 200 от Ingress).

### Шаг 6. Мониторинг и логирование
- Установлена **Grafana** с доступом через LoadBalancer.
- Настроен **Loki** как Data Source (Prometheus и Promtail не совместимы с Autopilot в trial).
- Логи контейнеров доступны через `kubectl logs` и Grafana Explore.

## Ограничения и решения
| Проблема                                   | Решение                                                                                     |
|--------------------------------------------|---------------------------------------------------------------------------------------------|
| Триальный лимит CPU (12 vCPU)              | Снижение requests/limits до 10m CPU и 16 Mi памяти, отключение второстепенных сервисов. Удалось добиться стабильной работы 2 из 3 необходимых сервисов, далее ограничение ресурсов.     |
| GKE Autopilot запрещает DaemonSet + hostPath| Использование `kubectl logs` и отказ от Promtail (логи доступны в Grafana через Loki API)   |
| Кнопка "Edit Quota" неактивна в trial версии      | Ограничение, которое никак не изменить; в платном варианте не актуально                         |
| Grafana и Loki       | Установлены, но поды находятся в ожидании ресурсов                        |



## Быстрый старт
1. Клонируйте оба репозитория:
   ```bash
   git clone https://github.com/Apowmet/Graduation-project
   git clone https://github.com/Apowmet/Graduation-infra
2. Настройте GCP и примените Terraform (см. Graduation-infra/README.md)
3. Запустите CI (пуш в main) – образы соберутся автоматически.
4. Разверните приложение:
  ```bash
  cd Graduation-infra/online-shop
  helm upgrade --install online-shop . -n app --create-namespace
  ```
5. Получите внешний IP Ingress и откройте магазин.

## Скриншоты
<details> <summary>CI/CD Pipeline</summary> <img width="1050" height="498" alt="{88E64EB8-06CC-4B61-8041-22508D754019}" src="https://github.com/user-attachments/assets/2157f4d6-f47d-43f7-a944-e87f95a0fa26" /> 
</details> <details> <summary>Grafana (ограничение trial) + в качестве примера локальный вариант</summary> <img width="691" height="111" alt="{FFBCB59E-1202-491E-8B88-FF7A2FE69B1F}" src="https://github.com/user-attachments/assets/bd645381-f16a-4c9f-9aed-e07f732a68d6" />
<img width="1027" height="446" alt="{4190E443-A5B0-479F-85EF-D7E5F9ECBFA5}" src="https://github.com/user-attachments/assets/d87e8c7a-9d46-4600-8009-8bb63dba957c" />
</details> <details> <summary>Логи приложения</summary>  <img width="893" height="254" alt="{75109FD2-497B-4398-B4DB-7D9153556240}" src="https://github.com/user-attachments/assets/c62152e0-ca7f-4086-9681-fb8fb19ca950" />
</details> <details> <summary>Магазин (работа без ограничений, локально)</summary> <img width="1429" height="1053" alt="{4E4F1A19-1859-45AA-93DD-F6E39EA163CC}" src="https://github.com/user-attachments/assets/5992b456-be6d-4937-991e-1f869dd16d79" /> </details>
</details> <details> <summary>Магазин (работа при ограничениях)</summary> <img width="974" height="392" alt="image" src="https://github.com/user-attachments/assets/1351c034-083c-418a-971c-f0be0784b3d9" /> </details>


## Автор
  Алексей Иванищев
  
  GitHub: @Apowmet
  
  Дипломный проект Junior DevOps, 2024 (обновлен в 2026)
