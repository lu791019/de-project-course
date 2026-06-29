# hahow-crawler-de-course-materials

> 引用來源：本專案改編自 [DataEngCamp/de-project](https://github.com/DataEngCamp/de-project)，加入整合版 Docker Compose、bug 修復與教學文件。

本專案是一個完整的資料工程管道，使用 Python 爬蟲擷取 Hahow 線上課程平台資料，整合了多個現代化的資料處理工具：

- **資料擷取**: 使用 Python 爬蟲技術擷取 Hahow 線上課程平台資料
- **任務調度**: 透過 Celery + RabbitMQ 實現分散式任務處理
- **工作流程管理**: 使用 Apache Airflow 進行 ETL 流程編排
- **資料存儲**: MySQL 資料庫儲存結構化資料
- **資料視覺化**: Metabase 建立商業智慧儀表板
- **容器化部署**: Docker & Docker Compose 統一管理服務

### 資料流程

```
Hahow 網站 → Python 爬蟲 → RabbitMQ → Celery Workers → MySQL → Metabase
                ↑                                                    ↓
            Airflow DAG                                         商業智慧報表
```

## 資料夾結構

```
hahow-crawler/
├── Dockerfile                               # Docker 映像檔配置
├── pyproject.toml                           # Python 專案配置檔
├── uv.lock                                  # uv 套件管理鎖定檔
│
├── data_ingestion/                          # 核心資料擷取模組
│   ├── config.py                            # 配置檔（環境變數）
│   ├── worker.py                            # Celery Worker 設定
│   ├── tasks.py                             # Celery 任務定義（範例）
│   ├── producer.py                          # 基本 Producer（範例）
│   ├── mysql.py                             # MySQL 連線模組
│   │
│   ├── # Hahow 爬蟲
│   ├── hahow_crawler_common.py              # 共用函式
│   ├── hahow_crawler_course.py              # 課程爬蟲
│   ├── hahow_crawler_course_optimized.py    # 課程爬蟲優化版
│   ├── hahow_crawler_course_optimized_sales.py # 課程銷售爬蟲
│   ├── hahow_crawler_article.py             # 文章爬蟲
│   ├── hahow_crawler_article_optimized.py   # 文章爬蟲優化版
│   │
│   ├── # Producer
│   ├── producer_crawler_hahow_all.py        # 全部資料 Producer
│   ├── producer_crawler_hahow_by_queue.py   # 分隊列 Producer
│   ├── producer_crawler_hahow_course.py     # 課程 Producer
│   │
│   └── # Tasks
│       ├── tasks_crawler_hahow.py           # Hahow 爬蟲任務（含 MySQL 寫入）
│       ├── tasks_crawler_hahow_course.py    # 課程爬蟲任務（CSV 版）
│       └── tasks_crawler_hahow_article.py   # 文章爬蟲任務（CSV 版）
│
├── airflow/                                 # Apache Airflow 工作流程管理
│   ├── Dockerfile                           # Airflow Docker 映像檔
│   ├── airflow.cfg                          # Airflow 配置檔
│   ├── docker-compose-airflow.yml           # Airflow Docker Compose
│   └── dags/                                # DAG 工作流程定義
│
├── example/                                 # SQL 範例
│   ├── employees.sql
│   ├── students.sql
│   ├── ecommerce.sql
│   └── mock_course_sales_data.sql
│
├── metabase/                                # Metabase 視覺化
│   └── docker-compose-metabase.yml
│
└── output/                                  # 爬蟲輸出（CSV）
```

## 六個服務說明

| 服務 | Image | Port | 角色 |
|------|-------|------|------|
| rabbitmq | rabbitmq:3-management | 5672 / 15672 | 訊息佇列（派工）|
| flower | mher/flower:latest | 5555 | Celery 任務監控 |
| mysql | mysql:8.0 | 3306 | 資料庫（存爬蟲結果）|
| phpmyadmin | phpmyadmin:latest | 8000 | 資料庫管理介面 |
| worker | 本地 build | — | 執行爬蟲（Celery Worker）|
| producer | 本地 build | — | 發送爬蟲任務（一次性）|

---

## 指令

### 🔧 環境設定

```bash
# 建立虛擬環境並安裝依賴
uv sync
```

### 🐳 Docker Compose（整合版，推薦）

本專案使用 `docker-compose-local.yml` 一個檔案管理所有服務，從 Dockerfile 本地 build，不依賴 DockerHub image。

#### 場景一：基礎設施用 Docker，Worker / Producer 本機跑

```bash
# 啟動基礎設施（RabbitMQ + Flower + MySQL + phpMyAdmin）
docker compose -f docker-compose-local.yml up -d rabbitmq flower mysql phpmyadmin

# 確認服務正常（等 20-30 秒）
docker compose -f docker-compose-local.yml ps -a

# 本機啟動 Worker
uv run python -m celery -A data_ingestion.worker worker --loglevel=info

# 本機發送任務
uv run data_ingestion/producer.py
uv run data_ingestion/producer_crawler_hahow_all.py
uv run data_ingestion/producer_crawler_hahow_course.py
```

#### 場景二：全部用 Docker 跑

```bash
# 啟動 infra + worker（先不起 producer）
docker compose -f docker-compose-local.yml up -d --build rabbitmq flower mysql phpmyadmin worker

# 確認 worker ready
docker compose -f docker-compose-local.yml logs worker | grep ready

# 發送任務
docker compose -f docker-compose-local.yml up producer
```

#### 停止與清理

```bash
# 停止（保留資料）
docker compose -f docker-compose-local.yml down

# 停止（清除資料庫資料）
docker compose -f docker-compose-local.yml down -v
```

### 🔍 Web 介面

| 服務 | 網址 | 帳密 |
|------|------|------|
| RabbitMQ 管理 | http://localhost:15672 | worker / worker |
| Flower 監控 | http://localhost:5555 | （無）|
| phpMyAdmin | http://localhost:8000 | root / 1234 |

### 🕷️ 爬蟲與任務執行（本機）

```bash
# 啟動 Worker（預設 queue）
uv run python -m celery -A data_ingestion.worker worker --loglevel=info

# 啟動 Worker（指定 hostname）
uv run python -m celery -A data_ingestion.worker worker --loglevel=info --hostname=worker1@%h
uv run python -m celery -A data_ingestion.worker worker --loglevel=info --hostname=worker2@%h

# 啟動 Worker（指定 concurrency）
uv run python -m celery -A data_ingestion.worker worker --loglevel=info --hostname=worker1@%h --concurrency=1

# 啟動 Worker（指定 queue）
uv run python -m celery -A data_ingestion.worker worker --loglevel=info --hostname=worker1@%h -Q hahow_course
uv run python -m celery -A data_ingestion.worker worker --loglevel=info --hostname=worker2@%h -Q hahow_article
uv run python -m celery -A data_ingestion.worker worker --loglevel=info --hostname=worker3@%h -Q hahow_course,hahow_article

# Producer 發送任務
uv run data_ingestion/producer.py                       # 最簡單範例
uv run data_ingestion/producer_crawler_hahow_all.py     # 全部分類（course + article）
uv run data_ingestion/producer_crawler_hahow_course.py  # 只發課程
uv run data_ingestion/producer_crawler_hahow_by_queue.py # 分 queue 發送

# 連遠端（RabbitMQ/MySQL 在雲端）
uv run --env-file .env python -m celery -A data_ingestion.worker worker --loglevel=info
uv run --env-file .env data_ingestion/producer_crawler_hahow_all.py
```

### 🗄️ 驗證資料

```bash
# MySQL 查資料
docker exec mysql mysql -uroot -p1234 hahow -e \
  "SHOW TABLES; SELECT 'hahow_course' as tbl, COUNT(*) as cnt FROM hahow_course UNION ALL SELECT 'hahow_article', COUNT(*) FROM hahow_article UNION ALL SELECT 'hahow_course_sales', COUNT(*) FROM hahow_course_sales;"

# 查看 Worker log
docker compose -f docker-compose-local.yml logs worker | grep -E "UPSERT|INSERT|succeeded"
```

### 🌍 環境變數設定

本專案的 `config.py` 已設定預設值，本機開發（RabbitMQ/MySQL 在 localhost Docker）不需要額外設定。

連遠端或需要自訂環境變數時，有三種方式：

**方法一：使用 uv 內建功能**
```bash
uv run --env-file .env data_ingestion/producer.py
uv run --env-file .env python -m celery -A data_ingestion.worker worker --loglevel=info
```
- uv 原生支援
- 明確指定環境變數來源
- 不污染系統環境

**方法二：使用 source 載入**
```bash
source .env
uv run data_ingestion/producer.py
uv run python -m celery -A data_ingestion.worker worker --loglevel=info
```
- 最簡單的方式
- 會影響當前 shell 環境

**方法三：直接在終端設定**
```bash
source .env
python data_ingestion/hahow_crawler_article_optimized.py
```

範例 `.env` 見 `.env.example`：
```bash
cp .env.example .env
# 修改裡面的 RABBITMQ_HOST、MYSQL_HOST 等
```

### 📋 查看 Container 狀態

```bash
# 查看所有 container 狀況
docker ps -a

# 查看特定 container log
docker logs -f rabbitmq
docker logs -f flower
docker logs -f mysql
```

---

## 進階：分開版 Docker Compose

每個服務一個 compose file，適合教學逐步展示。需先建立共用 network。

```bash
# 建立共用 network（只要做一次）
docker network create my_network

# 啟動 RabbitMQ + Flower
docker compose -f docker-compose-broker.yml up -d

# 啟動 MySQL + phpMyAdmin
docker compose -f docker-compose-mysql.yml up -d

# 啟動 Worker
docker compose -f docker-compose-worker.yml up -d

# 發送任務
docker compose -f docker-compose-producer.yml up

# 停止所有
docker compose -f docker-compose-broker.yml down
docker compose -f docker-compose-mysql.yml down
docker compose -f docker-compose-worker.yml down
docker network rm my_network
```

### Docker Compose 檔案一覽

| 檔案 | 用途 |
|------|------|
| `docker-compose-local.yml` | **推薦**。整合版，本地 build，一鍵啟動 |
| `docker-compose.yml` | 整合版，使用 DockerHub image（amd64-only） |
| `docker-compose-broker.yml` | 分開版：RabbitMQ + Flower |
| `docker-compose-mysql.yml` | 分開版：MySQL + phpMyAdmin |
| `docker-compose-worker.yml` | 分開版：Worker |
| `docker-compose-producer.yml` | 分開版：Producer |
| `docker-compose-worker-vm.yml` | VM 部署用 |

---

## 進階：Apache Airflow

```bash
# 啟動 Airflow
docker compose -f airflow/docker-compose-airflow.yml up -d

# 查看 Airflow 服務狀態
docker compose -f airflow/docker-compose-airflow.yml ps

# 查看 Airflow 服務日誌
docker compose -f airflow/docker-compose-airflow.yml logs -f

# 存取 Airflow 網頁介面
# http://localhost:8080（airflow / airflow）

# 停止 Airflow
docker compose -f airflow/docker-compose-airflow.yml down
```

## 進階：Metabase 商業智慧儀表板

```bash
# 啟動 Metabase
docker compose -f metabase/docker-compose-metabase.yml up -d

# 查看 Metabase 服務狀態
docker compose -f metabase/docker-compose-metabase.yml ps

# 存取 Metabase 網頁介面
# http://localhost:3000

# 停止 Metabase
docker compose -f metabase/docker-compose-metabase.yml down
```
