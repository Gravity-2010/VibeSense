# 📊 VibeSense

### Event-Driven Social Media Sentiment Monitoring

> **Collaborative Team Project**
>
> VibeSense is a distributed social-media monitoring prototype that periodically collects comments from a YouTube video, analyzes sentiment using a pre-trained NLP model, aggregates sentiment trends over time, and delivers email updates to users.

The project explores how an AI-enabled monitoring workflow can be decomposed into independent services connected through asynchronous messaging and persistent scheduling.

---

## 🚀 What VibeSense Does

A user provides:

* Their name
* Email address
* YouTube video URL
* Monitoring duration

VibeSense then:

1. Validates the YouTube video.
2. Creates a monitoring job.
3. Schedules recurring comment collection.
4. Retrieves new comments from YouTube.
5. Cleans and preprocesses the text.
6. Sends comments through an NLP sentiment model.
7. Aggregates interval and overall sentiment.
8. Stores monitoring results.
9. Sends formatted email updates.

---

# 🏗️ Architecture

```mermaid
flowchart LR
    U[Streamlit UI]
    Y[YouTube Data API]

    DB[(PostgreSQL)]
    R[(Redis)]
    Q[(RabbitMQ)]

    C[Celery + RedBeat]
    I[Ingestion Service]
    AI[AI Service]
    A[Aggregation Service]
    N[Notification Service]

    U --> Y
    U --> DB

    DB --> C
    C --> I

    I --> Y
    I --> Q

    Q --> AI
    AI --> Q

    Q --> A
    A --> DB
    A --> Q

    Q --> N
    N --> DB
    N --> E[Email / SMTP]

    C --> R
```

---

# 🔄 Processing Pipeline

```text
Monitoring Request
       ↓
PostgreSQL Job
       ↓
Celery / RedBeat Schedule
       ↓
YouTube Comment Retrieval
       ↓
Text Preprocessing
       ↓
RabbitMQ
       ↓
Sentiment Analysis
       ↓
RabbitMQ
       ↓
Sentiment Aggregation
       ↓
PostgreSQL
       ↓
RabbitMQ
       ↓
Email Notification
```

---

# 🖥️ Streamlit UI

The Streamlit interface accepts:

* Full name
* Email address
* YouTube URL
* Monitoring duration

Before creating a monitoring job, VibeSense:

* Extracts the YouTube video ID
* Validates the URL
* Queries the YouTube API
* Confirms the video title
* Parses the monitoring duration
* Stores the monitoring job in PostgreSQL

The job can then be picked up by the dynamic scheduling workflow.

---

# ⏱️ Dynamic Scheduling

VibeSense uses:

`Celery` · `Redis` · `RedBeat`

Monitoring jobs contain:

```text
Polling interval
Total monitoring duration
Creation timestamp
Last-fetch timestamp
Scheduling status
```

A recurring scheduler periodically looks for unscheduled jobs and creates persistent RedBeat entries for them.

Each monitoring job runs at its configured interval until its total monitoring duration expires.

Expired schedules are removed automatically.

---

# 📥 Comment Ingestion

The ingestion service retrieves YouTube comments using the **YouTube Data API**.

The system supports pagination and stores information such as:

```text
comment ID
comment text
publication timestamp
like count
```

For repeated monitoring intervals, comments that were already processed are filtered using the job's last-fetch timestamp.

Only newly observed comments continue through the NLP pipeline.

---

# 🧹 Text Preprocessing

Comments are normalized before sentiment analysis.

The current preprocessing pipeline uses spaCy to remove:

* Stop words
* URLs
* Punctuation

and normalize text to lowercase.

---

# 🤖 Sentiment Analysis

The AI service uses a pre-trained Hugging Face sentiment-analysis pipeline.

Current model:

```text
tabularisai/multilingual-sentiment-analysis
```

Comments are processed in batches and converted into:

```text
Sentiment Label
Confidence Score
```

The AI service can process direct API requests for testing and also operate as a RabbitMQ consumer for the asynchronous pipeline.

---

# 📊 Sentiment Aggregation

Individual sentiment outputs are consumed by the aggregation service.

Sentiment labels are converted into numerical categories and combined using confidence-weighted averaging.

A simplified representation is:

```text
Very Negative / Negative → 0
Neutral                  → 1
Positive / Very Positive → 2
```

For each monitoring interval, the service calculates:

* Interval sentiment
* Interval confidence

It also combines stored intervals to calculate:

* Overall sentiment
* Overall confidence

Results are persisted in PostgreSQL and forwarded to the notification queue.

---

# 📧 Email Notifications

The notification service consumes aggregate results from RabbitMQ.

It retrieves the corresponding monitoring job from PostgreSQL and generates a formatted HTML email containing:

* Video title
* Monitoring interval
* Latest sentiment category
* Latest confidence category
* Overall sentiment trend
* Overall confidence

The email is rendered using **Jinja2** and delivered through an SMTP server.

---

# 📨 Asynchronous Messaging

RabbitMQ decouples the main processing stages.

```text
Ingestion
    ↓
analysis_queue
    ↓
AI Service
    ↓
aggregation_queue
    ↓
Aggregation Service
    ↓
notification_queue
    ↓
Notification Service
```

This allows individual stages to run independently rather than forcing the entire monitoring workflow through one synchronous request.

---

# 🗄️ Data Model

## Monitoring Jobs

Stored monitoring-job information includes:

```text
job ID
YouTube video ID
video title
user name
email
monitoring interval
total duration
schedule status
last fetched timestamp
creation timestamp
```

## Interval Results

Interval records store:

```text
job ID
timestamp
average sentiment
average confidence
```

These records allow VibeSense to calculate trends across multiple monitoring intervals.

---

# 🧰 Local Infrastructure

Docker Compose provisions the supporting development infrastructure:

```text
PostgreSQL
RabbitMQ
Redis
```

Application services themselves are currently run separately during development.

---

# 🛠️ Tech Stack

| Area                  | Technology                |
| --------------------- | ------------------------- |
| Language              | Python                    |
| UI                    | Streamlit                 |
| APIs                  | FastAPI                   |
| Task Processing       | Celery                    |
| Persistent Scheduling | RedBeat                   |
| Messaging             | RabbitMQ                  |
| Task Backend          | Redis                     |
| Database              | PostgreSQL                |
| ORM                   | SQLAlchemy                |
| Migrations            | Alembic                   |
| NLP                   | Hugging Face Transformers |
| Text Processing       | spaCy                     |
| Aggregation           | Pandas                    |
| Email Templates       | Jinja2                    |
| Notifications         | SMTP                      |
| External API          | YouTube Data API          |
| Infrastructure        | Docker Compose            |

---

# 📁 Repository Structure

```text
VibeSense/
│
├── alembic/
│
├── src/
│   ├── ui_service/
│   │   └── app.py
│   │
│   ├── ingestion_service/
│   │   ├── app.py
│   │   ├── preprocessor.py
│   │   ├── scheduler.py
│   │   ├── tasks.py
│   │   └── youtube_fetcher.py
│   │
│   ├── ai_service/
│   │   └── app.py
│   │
│   ├── aggregation_service/
│   │   └── app.py
│   │
│   ├── notification_service/
│   │   └── app.py
│   │
│   ├── models.py
│   └── utils.py
│
├── tests/
├── alembic.ini
├── docker-compose.yml
├── requirements.txt
├── .gitignore
├── LICENSE
└── README.md
```

---

# 🚀 Local Setup

## 1. Clone

```bash
git clone https://github.com/Gravity-2010/VibeSense.git
cd VibeSense
```

## 2. Create a virtual environment

```bash
python -m venv .venv
```

Linux/macOS:

```bash
source .venv/bin/activate
```

Windows:

```bash
.venv\Scripts\activate
```

## 3. Install dependencies

```bash
pip install -r requirements.txt
```

---

# 🔐 Environment Configuration

Create:

```text
.env
```

Example:

```env
YOUTUBE_API_KEY=your_youtube_api_key

DB_URL=postgresql://user:pass@localhost:5432/vibesense

RABBITMQ_URL=amqp://guest:guest@localhost:5672/

REDIS_URL=redis://localhost:6379/0

SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@example.com
SMTP_PASS=your_app_password
```

Never commit real credentials.

---

# 🐳 Start Supporting Infrastructure

```bash
docker compose up -d
```

This starts:

```text
PostgreSQL
RabbitMQ
Redis
```

---

# 🗃️ Run Database Migrations

```bash
alembic upgrade head
```

---

# ▶️ Run the Services

Services are started independently during local development.

### Streamlit UI

```bash
streamlit run src/ui_service/app.py
```

### Celery Worker

```bash
celery -A src.ingestion_service.app.celery_app worker --loglevel=INFO
```

### Celery Beat / RedBeat Scheduler

```bash
celery -A src.ingestion_service.app.celery_app beat --loglevel=INFO
```

### AI Consumer

```bash
python -m src.ai_service.app
```

### Aggregation Consumer

```bash
python -m src.aggregation_service.app
```

### Notification Consumer

```bash
python -m src.notification_service.app
```

---

# 🧪 Testing

Run:

```bash
python -m pytest
```

The repository currently contains an **early test suite** covering utilities and preprocessing behavior.

Additional integration and service-level tests remain an area for improvement.

---

# ⚠️ Limitations

VibeSense is a collaborative systems prototype rather than a production monitoring platform.

Current limitations include:

* YouTube-only ingestion
* Dependence on external YouTube API quotas
* Dependence on a pre-trained general sentiment model
* Limited automated test coverage
* No complete containerization of application services
* No production authentication/authorization layer
* No centralized observability stack
* SMTP configuration required for notifications
* No production deployment configuration

---

# 🔮 Future Improvements

Potential extensions include:

* End-to-end integration tests
* Containerizing each service
* CI/CD
* Dead-letter queues
* Stronger retry policies
* Structured monitoring and tracing
* User authentication
* Secure secrets management
* Multi-platform social-media ingestion
* Historical trend dashboards
* Domain-specific sentiment models
* Production deployment configuration

---

# 👥 Project Context

**VibeSense was developed collaboratively as a team project.**

This repository is a fork of the shared project repository and represents collaborative system-design and implementation work.

Individual contributions should be documented separately where appropriate rather than attributing the entire codebase to one contributor.

---

## 📌 Repository Status

**Active collaborative prototype**

The project demonstrates event-driven architecture, asynchronous processing, NLP integration, persistence, dynamic scheduling, and notification workflows.
