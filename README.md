# Unified Enterprise CI/CD Cluster (30+ Projects)

**CLASSIFICATION:** INTERNAL - PROPRIETARY  
**ARCHITECTURE:** Native GitHub Actions Runner Controller (ARC)  
**TARGET:** Cost-Efficiency at Scale (30+ Repositories)

---

## 1. Концепция проекта

**Это высокопроизводительная инфраструктурная прослойка, которая переносит выполнение всех CI/CD процессов из облака GitHub (Hosted Runners) в наш выделенный **Kubernetes-кластер**. 

Вместо оплаты за каждую минуту работы в GitHub, мы используем **единый пул вычислительных мощностей** для всех 30+ репозиториев. Это позволяет достичь максимальной плотности упаковки задач и снизить операционные расходы на 70-80%.

---

## 2. Архитектурная схема (Zero-Layer Approach)

Система исключает лишние интерфейсы (как GitLab) и работает напрямую с API GitHub:

1.  **Source:** 30+ независимых репозиториев на **GitHub**.
2.  **Controller (ARC):** Actions Runner Controller, развернутый в K8s, который выступает в роли «диспетчера».
3.  **Compute Plane:** Авто-масштабируемый кластер Kubernetes, создающий Pod-раннеры "по требованию".
4.  **Storage:** Локальный S3-совместимый кэш (MinIO) для мгновенного доступа к зависимостям.

---

## 3. Ключевые преимущества для 30+ проектов

### 3.1. Динамическое масштабирование (HRA)
Используется **Horizontal Runner Autoscaler**. Если одновременно запускаются билды в 10 проектах — кластер мгновенно расширяет количество воркеров. Когда активность падает — ресурсы высвобождаются, и клиент за них не платит.

### 3.2. Экономика Spot-инстансов
Вся нагрузка CI/CD (сборка TSX, тесты Python) переносится на **прерываемые (Spot) виртуальные машины**. Это дает:
*   Снижение стоимости часа работы CPU в 4-5 раз.
*   Отсутствие фиксированной платы за каждый отдельный репозиторий.

### 3.3. Общий "Warm Cache"
Для 30 проектов на TypeScript (`node_modules`) и Python (`pip cache`) создается единый сетевой кэш. Это сокращает время сборки каждого проекта в среднем на 40%, так как пакеты скачиваются не из интернета, а из локальной сети кластера.

---

## 4. Технологический стек

*   **Runtime:** Kubernetes (Managed или Self-hosted).
*   **Orchestration:** Actions Runner Controller (ARC).
*   **IaC:** Terraform для описания узлов кластера и прав доступа.
*   **Languages:** Оптимизированные Docker-образы для сборки TypeScript, TSX, Python.
*   **Security:** Изоляция раннеров на уровне Kubernetes Namespaces; секреты доставляются напрямую через GitHub Secrets API или Vault.

---

## 5. Сравнение затрат (на 30 проектов)

| Характеристика | GitHub Hosted Runners | DevFlow ARC (K8s) |
| :--- | :--- | :--- |
| **Биллинг** | Поминутно (Дорого) | За ресурсы кластера (Выгодно) |
| **Производительность** | Стандартные VM (2 vCPU) | Настраиваемые (до 32 vCPU на билд) |
| **Параллелизм** | Лимитирован тарифом | Не ограничен (зависит от кластера) |
| **Итоговая стоимость** | Высокая (растет с каждым проектом) | Низкая (оптимизирована под общую нагрузку) |

---

## 6. План развертывания

1.  **Provisioning:** Развертывание K8s кластера и установка ARC через Helm.
2.  **Authentication:** Настройка GitHub App для управления 30 репозиториями через единый токен доступа.
3.  **Optimization:** Конфигурация MinIO для кэширования артефактов.
4.  **Migration:** Перевод `.github/workflows` всех проектов на использование тега `runs-on: self-hosted`.

---
**STATUS:** Ready for Deployment  
**OWNER:** DevOps Infrastructure Team

# Roadmap: Реализация (GitHub + Kubernetes)

Этот документ описывает технические шаги по развертыванию единого CI/CD центра для 30+ репозиториев.

## Шаг 1: Подготовка инфраструктуры (Infrastructure as Code)

Для 30 проектов нам нужен отказоустойчивый кластер. Рекомендуется использовать **Managed Kubernetes (K8s)** для снижения затрат на администрирование.

1.  **Создание кластера:** Через Terraform разворачиваем K8s кластер.
2.  **Node Pools (Группы узлов):**
    *   **System Pool:** 2 небольшие стабильные ВМ (для контроллеров).
    *   **CI Pool:** Масштабируемая группа из **Spot-инстансов** (прерываемые ВМ). Настраиваем Auto-scaling от 0 до N узлов.
3.  **Storage:** Развертывание **MinIO** внутри кластера для хранения общего кэша зависимостей (S3-compatible).

## Шаг 2: Настройка безопасности (GitHub App)

Вместо использования личных токенов (PAT), которые небезопасны и имеют лимиты, мы создаем **GitHub App** на уровне организации.

1.  Создайте GitHub App в настройках организации.
2.  Дайте права (Permissions): `Contents: Read`, `Metadata: Read`, `Actions: Read/Write`, `Workflows: Read/Write`.
3.  Установите приложение (Install App) на все 30 репозиториев.
4.  Скачайте `Private Key` и сохраните `App ID` и `Installation ID` — они понадобятся для контроллера.

## Шаг 3: Установка Actions Runner Controller (ARC)

ARC — это «мозг» системы, который живет внутри K8s и управляет раннерами.

1.  **Установка Cert-Manager:** (Необходим для работы ARC)
    ```bash
    helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true
    ```
2.  **Установка ARC:**
    ```bash
    helm install arc actions-runner-controller/actions-runner-controller \
      --namespace actions-runner-system --create-namespace \
      --set githubWebhookServer.enabled=true
    ```
3.  **Аутентификация:** Создайте K8s секрет с данными вашего GitHub App.

## Шаг 4: Конфигурация Автомасштабирования (HRA)

Это ключевой шаг для экономии денег. Мы настраиваем **Horizontal Runner Autoscaler**.

1.  Определите `RunnerDeployment` — шаблон ваших воркеров (Docker-образ с Node.js, Python, Terraform).
2.  Настройте **Webhook Driven Autoscaling**:
    *   Вместо того чтобы постоянно опрашивать GitHub (Polling), контроллер будет мгновенно получать сигнал от GitHub Webhook о новой задаче.
    *   Это позволяет держать 0 раннеров в простое и поднимать их только под задачу.

## Шаг 5: Оптимизация сборки (Speed Up)

Чтобы 30 проектов собирались быстрее, чем на стандартных серверах GitHub:

1.  **Shared Cache:** Настройте в `.github/workflows` использование вашего внутреннего MinIO через `actions/cache`.
2.  **Docker Registry:** Поднимите локальный Registry (или используйте облачный), чтобы скачивание базовых образов (Node/Python) происходило внутри сети облака, а не через интернет.

## Шаг 6: Миграция 30+ проектов

Перевод проектов происходит максимально просто:

1.  В каждом репозитории в файлах `.github/workflows/*.yml` измените одну строку:
    *   Было: `runs-on: ubuntu-latest`
    *   Стало: `runs-on: arc-runner-set` (название вашего пула)
2.  Теперь GitHub будет знать, что задачу нужно отправить не в свое облако, а в ваш кластер.

## Шаг 7: Мониторинг и контроль затрат

1.  **Grafana + Prometheus:** Настройте дашборд для отслеживания:
    *   Сколько раннеров запущено.
    *   Сколько денег сэкономлено (Spot vs Standard).
    *   Длительность билдов по каждому из 30 проектов.

---

### Итог реализации
Через 7 шагов вы получаете **автономную фабрику CI/CD**, которая:
*   Сама нанимает «рабочих» (Pod-раннеры), когда есть работа.
*   Сама их «увольняет», когда работа закончена.
*   Использует самые дешевые ресурсы (Spot) и работает на сверхскоростях за счет локального кэша.

# [CONFIDENTIAL] Minimal Hardware Footprint: DevFlow ARC

**PROJECT:** (GitHub CI/CD Consolidation)  
**TARGET:** 30+ Active Repositories  
**OPTIMIZATION:** Lean Infrastructure / Spot-Instance Driven

---

## 1. Management Layer (Static Infrastructure)
Эти ресурсы работают 24/7 для обеспечения связи с GitHub API и управления очередями задач.

| Component | Target VM Config | Resources | Purpose |
| :--- | :--- | :--- | :--- |
| **K8s Master / ARC** | Standard Instance | 4 vCPU / 8 GB RAM | Orchestration, ARC Controller, Webhook Server |
| **System Disk** | SSD / NVMe | 50 GB | Logs, OS, K8s System components |
| **Network** | Static IP / LB | 100 Mbps | Incoming GitHub Webhooks & API calls |

---

## 2. Execution Layer (Elastic Infrastructure)
Пул для запуска задач (Jobs). Масштабируется от 0 в зависимости от нагрузки во всех 30 проектах.

| Component | Instance Policy | VM Config | Scaling Limits |
| :--- | :--- | :--- | :--- |
| **CI/CD Runners** | **Spot / Preemptible** | 4 vCPU / 16 GB RAM | **Min: 0** / **Max: 5** Nodes |

*   **Почему 4/16?** Это оптимальный профиль для параллельной сборки TypeScript/TSX и запуска Python-тестов.
*   **Spot Policy:** Экономия до 80%. При прерывании узла ARC автоматически переназначает задачу на новый узел.

---

## 3. Storage Layer (Shared Services)
Централизованное хранилище для ускорения всех 30 проектов.

| Component | Technology | Volume | Note |
| :--- | :--- | :--- | :--- |
| **Global Cache** | MinIO (S3-compat) | 100 GB NVMe | Хранение `node_modules`, `pip_cache` и Docker layers |
| **Local Registry** | Docker Registry | Included in Cache | Ускоряет Pull базовых образов внутри кластера |

---

## 4. Summary of Minimum Consumption (SLA 99.0)

Для обслуживания **30+ проектов** минимальный порог ресурсов составляет:

1.  **Постоянный минимум (Always-on):**
    *   1 VM (4 vCPU, 8 GB RAM)
    *   150 GB Total Storage (NVMe/SSD)
    *   1 Load Balancer
2.  **Переменный максимум (Peak Load):**
    *   До 20 vCPU и 80 GB RAM суммарно (распределено по Spot-узлам).
    *   *Автоматически отключается при отсутствии задач.*

## 5. Cost-Effectiveness Ratio (Efficiency)

*   **System Overhead:** < 10% (ресурсы, потребляемые самой системой управления).
*   **Performance:** Каждый билд получает гарантированные 2-4 vCPU (в 2 раза мощнее стандартных GitHub Runners).
*   **Consolidation Benefit:** Вместо 30 разрозненных платных подписок — один эластичный кластер, работающий по фактическому потреблению.

---

