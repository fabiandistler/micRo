# 🚀 Modern Microservice System Architecture in R

## **System Overview**

This architecture demonstrates an enterprise-grade microservice
ecosystem built entirely in R, optimized for data-intensive operations,
ML model serving, and statistical computing services.

------------------------------------------------------------------------

## **🏗️ Architecture Diagram**

    ┌─────────────────────────────────────────────────────────────────┐
    │                        API GATEWAY (Kong/Traefik)                │
    │                    Load Balancer & Rate Limiting                 │
    └────────────┬────────────────────────────────────────────────────┘
                 │
        ┌────────┴──────────┬──────────────┬──────────────┬───────────┐
        │                   │              │              │           │
    ┌───▼────┐      ┌──────▼─────┐  ┌─────▼─────┐  ┌────▼────┐  ┌──▼────┐
    │ User   │      │   ML       │  │  Data     │  │ Report  │  │ Real  │
    │ Auth   │      │   Model    │  │  Process  │  │ Gen     │  │ Time  │
    │ Service│      │   Service  │  │  Service  │  │ Service │  │ Stream│
    │(Plumber2)     │(RestRserve)│  │(Plumber2) │  │(Plumber2│  │(Rest  │
    │        │      │            │  │           │  │)        │  │Rserve)│
    └───┬────┘      └──────┬─────┘  └─────┬─────┘  └────┬────┘  └──┬────┘
        │                  │              │             │          │
        └──────────────────┴──────────────┴─────────────┴──────────┘
                                    │
                        ┌───────────┴────────────┐
                        │                        │
                ┌───────▼─────┐          ┌──────▼──────┐
                │  PostgreSQL │          │    Redis    │
                │   Database  │          │    Cache    │
                └─────────────┘          └─────────────┘
                        │                        │
                ┌───────┴────────┐      ┌────────┴────────┐
                │ Event Bus      │      │  Message Queue  │
                │ (RabbitMQ)     │      │  (RabbitMQ)     │
                └────────────────┘      └─────────────────┘
                        │
            ┌───────────┴────────────┬──────────────┐
            │                        │              │
    ┌───────▼──────┐      ┌─────────▼────┐   ┌─────▼──────┐
    │ Monitoring   │      │   Logging    │   │  Tracing   │
    │ (Prometheus) │      │ (ELK Stack)  │   │  (Jaeger)  │
    └──────────────┘      └──────────────┘   └────────────┘

------------------------------------------------------------------------

## **📦 Core Microservices**

### **1. User Authentication Service**

**Framework:** plumber2  
**Purpose:** JWT-based authentication, OAuth2, user management

``` r
# auth-service/plumber.R
library(plumber2)
library(jose)
library(DBI)
library(pool)

# Database connection pool
db_pool <- dbPool(
  RPostgres::Postgres(),
  host = Sys.getenv("DB_HOST"),
  dbname = "users",
  user = Sys.getenv("DB_USER"),
  password = Sys.getenv("DB_PASSWORD"),
  minSize = 5,
  maxSize = 20
)

#* @apiTitle User Authentication Service
#* @apiDescription Handles user authentication and authorization
#* @apiVersion 1.0.0

#* Login endpoint
#* @post /api/v1/auth/login
#* @param username:string User's email or username
#* @param password:string User's password
#* @serializer json
function(req, res, username, password) {
  tryCatch({
    # Validate credentials
    user <- dbGetQuery(
      db_pool,
      "SELECT id, username, password_hash, role FROM users WHERE username = $1",
      params = list(username)
    )
    
    if (nrow(user) == 0 || !bcrypt::checkpw(password, user$password_hash)) {
      res$status <- 401
      return(list(error = "Invalid credentials"))
    }
    
    # Generate JWT token
    claims <- jwt_claim(
      user_id = user$id,
      username = user$username,
      role = user$role,
      exp = Sys.time() + 3600 * 24  # 24 hours
    )
    
    token <- jwt_encode_hmac(claims, secret = Sys.getenv("JWT_SECRET"))
    
    # Log authentication event
    send_to_event_bus("auth.login", list(
      user_id = user$id,
      timestamp = Sys.time()
    ))
    
    list(
      token = token,
      user = list(id = user$id, username = user$username, role = user$role)
    )
  }, error = function(e) {
    res$status <- 500
    list(error = "Internal server error", message = e$message)
  })
}

#* Validate token endpoint
#* @post /api/v1/auth/validate
#* @param token:string JWT token
#* @serializer json
function(req, res, token) {
  tryCatch({
    claims <- jwt_decode_hmac(token, secret = Sys.getenv("JWT_SECRET"))
    
    if (claims$exp < Sys.time()) {
      res$status <- 401
      return(list(error = "Token expired"))
    }
    
    list(valid = TRUE, user_id = claims$user_id, role = claims$role)
  }, error = function(e) {
    res$status <- 401
    list(valid = FALSE, error = "Invalid token")
  })
}

#* Health check
#* @get /health
function() {
  list(status = "healthy", service = "auth", timestamp = Sys.time())
}
```

------------------------------------------------------------------------

### **2. ML Model Service (High Performance)**

**Framework:** RestRserve  
**Purpose:** Serve trained ML models with high throughput

``` r
# ml-service/app.R
library(RestRserve)
library(reticulate)
library(lightgbm)
library(jsonlite)
library(future)
library(promises)

# Enable async processing
plan(multisession, workers = 8)

# Load pre-trained models
models <- list(
  classification = readRDS("models/classification_model.rds"),
  regression = readRDS("models/regression_model.rds"),
  forecast = readRDS("models/forecast_model.rds")
)

# Model cache for hot models
model_cache <- new.env()

# Prediction endpoint
predict_handler <- function(request, response) {
  body <- request$body
  
  tryCatch({
    data <- fromJSON(body)
    model_type <- data$model_type
    features <- as.data.frame(data$features)
    
    # Check cache
    cache_key <- digest::digest(list(model_type, features))
    if (exists(cache_key, envir = model_cache)) {
      predictions <- get(cache_key, envir = model_cache)
    } else {
      # Make prediction
      model <- models[[model_type]]
      predictions <- predict(model, features)
      
      # Cache result (with TTL)
      assign(cache_key, predictions, envir = model_cache)
    }
    
    # Send metrics to monitoring
    send_metric("prediction.latency", Sys.time() - request$start_time)
    send_metric("prediction.count", 1)
    
    response$set_body(toJSON(list(
      predictions = predictions,
      model_version = model$version,
      timestamp = Sys.time()
    )))
    response$set_content_type("application/json")
    
  }, error = function(e) {
    response$set_status_code(500L)
    response$set_body(toJSON(list(error = e$message)))
  })
}

# Batch prediction endpoint (async)
batch_predict_handler <- function(request, response) {
  body <- request$body
  data <- fromJSON(body)
  
  # Process batches asynchronously
  future_promise({
    results <- lapply(data$batches, function(batch) {
      model <- models[[batch$model_type]]
      predict(model, as.data.frame(batch$features))
    })
    
    list(
      batch_id = data$batch_id,
      results = results,
      processed_at = Sys.time()
    )
  }) %...>% {
    response$set_body(toJSON(.))
    response$set_content_type("application/json")
  }
}

# Model metrics endpoint
metrics_handler <- function(request, response) {
  metrics <- list(
    models_loaded = length(models),
    cache_size = length(ls(model_cache)),
    uptime = Sys.time() - app_start_time,
    memory_usage = pryr::mem_used()
  )
  
  response$set_body(toJSON(metrics))
  response$set_content_type("application/json")
}

# Create application
app <- Application$new(content_type = "application/json")

# Add middleware
app$add_middleware(
  MiddlewareRequestId$new(),
  MiddlewareLogging$new(),
  MiddlewareCORS$new()
)

# Routes
app$add_post(path = "/api/v1/predict", FUN = predict_handler)
app$add_post(path = "/api/v1/predict/batch", FUN = batch_predict_handler)
app$add_get(path = "/api/v1/metrics", FUN = metrics_handler)
app$add_get(path = "/health", FUN = function(request, response) {
  response$set_body(toJSON(list(status = "healthy")))
})

# Start server
app_start_time <- Sys.time()
backend <- BackendRserve$new()
backend$start(app, http_port = 8080)
```

------------------------------------------------------------------------

### **3. Data Processing Service**

**Framework:** plumber2  
**Purpose:** ETL operations, data transformations, aggregations

``` r
# data-service/plumber.R
library(plumber2)
library(data.table)
library(arrow)
library(dplyr)
library(DBI)
library(future)
library(promises)

plan(multisession, workers = 4)

#* @apiTitle Data Processing Service
#* @apiDescription High-performance data processing and transformation

# Parquet file processing endpoint
#* @post /api/v1/process/parquet
#* @param file_path:string Path to parquet file
#* @param operations:array List of operations to perform
#* @serializer json
function(req, res, file_path, operations) {
  future_promise({
    # Read parquet file efficiently
    df <- arrow::read_parquet(file_path)
    
    # Apply operations pipeline
    for (op in operations) {
      df <- switch(op$type,
        "filter" = filter(df, !!parse_expr(op$condition)),
        "mutate" = mutate(df, !!parse_expr(op$expression)),
        "aggregate" = group_by(df, !!!syms(op$group_by)) %>%
                      summarize(!!!parse_exprs(op$metrics)),
        "join" = left_join(df, read_parquet(op$join_path), by = op$keys),
        df
      )
    }
    
    # Return result
    list(
      rows = nrow(df),
      columns = ncol(df),
      sample = head(df, 10),
      processing_time = Sys.time() - req$start_time
    )
  }) %...>% {
    res$body <- .
  } %...!% {
    res$status <- 500
    res$body <- list(error = as.character(.))
  }
}

# Real-time streaming aggregation
#* @post /api/v1/stream/aggregate
#* @serializer json
function(req, res) {
  stream_data <- jsonlite::fromJSON(req$body)
  
  # Process streaming data with data.table for speed
  dt <- as.data.table(stream_data$events)
  
  # Windowed aggregations
  results <- dt[, .(
    count = .N,
    mean_value = mean(value),
    max_value = max(value),
    min_value = min(value),
    stddev = sd(value)
  ), by = .(window = cut(timestamp, breaks = "1 min"))]
  
  # Publish to event bus
  publish_event("data.aggregated", results)
  
  list(
    windows = nrow(results),
    aggregations = results
  )
}

# Data quality check endpoint
#* @post /api/v1/quality/check
#* @serializer json
function(req, res) {
  data <- jsonlite::fromJSON(req$body)$data
  df <- as.data.frame(data)
  
  quality_report <- list(
    total_rows = nrow(df),
    total_columns = ncol(df),
    missing_values = colSums(is.na(df)),
    duplicates = sum(duplicated(df)),
    data_types = sapply(df, class),
    summary_stats = summary(df),
    outliers = lapply(df[sapply(df, is.numeric)], function(x) {
      q <- quantile(x, c(0.25, 0.75), na.rm = TRUE)
      iqr <- q[2] - q[1]
      sum(x < (q[1] - 1.5 * iqr) | x > (q[2] + 1.5 * iqr), na.rm = TRUE)
    })
  )
  
  quality_report
}
```

------------------------------------------------------------------------

### **4. Report Generation Service**

**Framework:** plumber2  
**Purpose:** Generate PDF reports, dashboards, visualizations

``` r
# report-service/plumber.R
library(plumber2)
library(rmarkdown)
library(ggplot2)
library(plotly)
library(DT)
library(htmlwidgets)

#* @apiTitle Report Generation Service

#* Generate PDF report
#* @post /api/v1/reports/pdf
#* @serializer contentType list(type="application/pdf")
function(req, res) {
  params <- jsonlite::fromJSON(req$body)
  
  # Create temporary R Markdown file
  rmd_content <- sprintf('
---
title: "%s"
author: "%s"
date: "%s"
output: pdf_document
params:
  data: !r NULL
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = FALSE, warning = FALSE, message = FALSE)
library(ggplot2)
library(knitr)
library(dplyr)
```

# Executive Summary

`{r} data <- params$data summary(data)`

# Visualizations

`{r} ggplot(data, aes(x = x, y = y)) + geom_line() + theme_minimal() + labs(title = "Trend Analysis")`

# Detailed Tables

`{r} kable(head(data, 20))` ’, params$title,params$author, Sys.Date())

\# Write to temp file tmp_rmd \<- tempfile(fileext = “.Rmd”) tmp_pdf \<-
tempfile(fileext = “.pdf”) writeLines(rmd_content, tmp_rmd)

\# Render report rmarkdown::render( tmp_rmd, output_file = tmp_pdf,
params = list(data = params\$data), quiet = TRUE )

\# Return PDF readBin(tmp_pdf, “raw”, n = file.info(tmp_pdf)\$size) }

\#\* Generate interactive dashboard \#\* @post /api/v1/reports/dashboard
\#\* @serializer html function(req, res) { params \<-
jsonlite::fromJSON(req$body)data < - as.data.frame(params$data)

\# Create interactive plots p1 \<- plot_ly(data, x = ~x, y = ~y, type =
“scatter”, mode = “lines”) p2 \<- plot_ly(data, x = ~category, y =
~value, type = “bar”)

\# Combine in dashboard dashboard \<- subplot(p1, p2, nrows = 2)

\# Return HTML htmlwidgets::saveWidget(dashboard, tmp_html \<-
tempfile(fileext = “.html”)) readLines(tmp_html) }

    ---

    ### **5. Real-Time Streaming Service**
    **Framework:** RestRserve
    **Purpose:** WebSocket connections, real-time data feeds

    ```r
    # stream-service/app.R
    library(RestRserve)
    library(websocket)
    library(jsonlite)
    library(later)

    # WebSocket connection pool
    ws_connections <- new.env()

    # Stream handler
    stream_handler <- function(request, response) {
      ws <- request$websocket

      if (!is.null(ws)) {
        # Generate connection ID
        conn_id <- uuid::UUIDgenerate()
        ws_connections[[conn_id]] <- ws

        # Handle messages
        ws$onMessage(function(binary, message) {
          data <- fromJSON(message)

          # Process streaming data
          result <- process_stream(data)

          # Broadcast to all clients
          broadcast_to_clients(toJSON(result))
        })

        # Handle disconnect
        ws$onClose(function() {
          rm(list = conn_id, envir = ws_connections)
        })
      }
    }

    # Broadcast function
    broadcast_to_clients <- function(message) {
      for (conn in ls(ws_connections)) {
        tryCatch({
          ws_connections[[conn]]$send(message)
        }, error = function(e) {
          # Remove dead connections
          rm(list = conn, envir = ws_connections)
        })
      }
    }

    # Periodic data push (simulating real-time updates)
    later::later(function() {
      # Fetch latest data
      latest_data <- fetch_latest_metrics()
      broadcast_to_clients(toJSON(latest_data))
    }, delay = 1, loop = TRUE)

    app <- Application$new()
    app$add_get(path = "/stream", FUN = stream_handler)

------------------------------------------------------------------------

## **🐳 Docker Configuration**

### **Individual Service Dockerfile**

``` dockerfile
# Dockerfile for R Microservice
FROM rocker/r-ver:4.3.2

# Install system dependencies
RUN apt-get update && apt-get install -y \
    libcurl4-openssl-dev \
    libssl-dev \
    libxml2-dev \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install R packages
RUN R -e "install.packages(c('plumber2', 'RestRserve', 'jsonlite', \
    'DBI', 'RPostgres', 'pool', 'jose', 'bcrypt', 'future', 'promises', \
    'data.table', 'arrow', 'dplyr', 'ggplot2', 'rmarkdown'), \
    repos='https://cloud.r-project.org/')"

# Create app directory
WORKDIR /app

# Copy application files
COPY . /app

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD Rscript -e "httr::GET('http://localhost:8080/health')" || exit 1

# Run the service
CMD ["Rscript", "run.R"]
```

### **docker-compose.yml**

``` yaml
version: '3.8'

services:
  # API Gateway
  kong:
    image: kong:3.4
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: postgres
      KONG_PG_DATABASE: kong
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: kong
    ports:
      - "8000:8000"
      - "8001:8001"
    depends_on:
      - postgres
    networks:
      - microservices

  # Auth Service
  auth-service:
    build: ./auth-service
    environment:
      DB_HOST: postgres
      DB_USER: postgres
      DB_PASSWORD: postgres
      JWT_SECRET: ${JWT_SECRET}
    ports:
      - "8081:8080"
    depends_on:
      - postgres
      - redis
    networks:
      - microservices
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1'
          memory: 1G

  # ML Model Service
  ml-service:
    build: ./ml-service
    environment:
      REDIS_HOST: redis
    ports:
      - "8082:8080"
    depends_on:
      - redis
    networks:
      - microservices
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: '4'
          memory: 8G

  # Data Processing Service
  data-service:
    build: ./data-service
    environment:
      DB_HOST: postgres
    ports:
      - "8083:8080"
    volumes:
      - data-volume:/data
    depends_on:
      - postgres
    networks:
      - microservices
    deploy:
      replicas: 3

  # Report Service
  report-service:
    build: ./report-service
    ports:
      - "8084:8080"
    networks:
      - microservices
    deploy:
      replicas: 2

  # Streaming Service
  stream-service:
    build: ./stream-service
    environment:
      RABBITMQ_HOST: rabbitmq
    ports:
      - "8085:8080"
    depends_on:
      - rabbitmq
    networks:
      - microservices

  # PostgreSQL Database
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: microservices
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - microservices

  # Redis Cache
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - microservices

  # RabbitMQ Message Broker
  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"
    networks:
      - microservices

  # Prometheus Monitoring
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - microservices

  # Grafana Dashboards
  grafana:
    image: grafana/grafana:latest
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - microservices

volumes:
  postgres-data:
  data-volume:
  prometheus-data:
  grafana-data:

networks:
  microservices:
    driver: bridge
```

------------------------------------------------------------------------

## **☸️ Kubernetes Deployment**

### **auth-service-deployment.yaml**

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: r-microservices
spec:
  replicas: 3
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
        version: v1
    spec:
      containers:
      - name: auth-service
        image: your-registry/auth-service:v1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: db.host
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: jwt-secret
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: auth-service
  namespace: r-microservices
spec:
  selector:
    app: auth-service
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: auth-service-hpa
  namespace: r-microservices
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: auth-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

------------------------------------------------------------------------

## **📊 Monitoring & Observability**

### **Prometheus Configuration**

``` yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'r-microservices'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - r-microservices
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: keep
        regex: .*-service
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod_name
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres:5432']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis:6379']
```

### **Custom Metrics in R**

``` r
# metrics.R - Prometheus metrics exporter
library(RestRserve)
library(prometheus)

# Create metrics
request_counter <- prometheus_counter(
  "http_requests_total",
  "Total HTTP requests",
  labels = c("method", "endpoint", "status")
)

request_duration <- prometheus_histogram(
  "http_request_duration_seconds",
  "HTTP request duration",
  labels = c("method", "endpoint"),
  buckets = c(0.001, 0.01, 0.1, 0.5, 1, 5)
)

# Middleware to capture metrics
metrics_middleware <- function(request, response) {
  start_time <- Sys.time()
  
  # Process request
  NextMiddleware$process_request(request, response)
  
  # Record metrics
  duration <- as.numeric(Sys.time() - start_time)
  request_counter$inc(
    method = request$method,
    endpoint = request$path,
    status = response$status_code
  )
  request_duration$observe(
    duration,
    method = request$method,
    endpoint = request$path
  )
}
```

------------------------------------------------------------------------

## **🔐 Security Best Practices**

### **1. API Gateway Security (Kong)**

``` yaml
# kong.yml
_format_version: "3.0"

services:
  - name: auth-service
    url: http://auth-service:80
    routes:
      - name: auth-route
        paths:
          - /api/v1/auth
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: local
      - name: cors
        config:
          origins:
            - "*"
          methods:
            - GET
            - POST
          headers:
            - Authorization
      - name: request-validator
        config:
          body_schema: |
            {
              "type": "object",
              "required": ["username", "password"]
            }
```

### **2. Service-to-Service Authentication**

``` r
# middleware/jwt_validator.R
jwt_auth_middleware <- function(request, response) {
  auth_header <- request$headers$Authorization
  
  if (is.null(auth_header) || !grepl("^Bearer ", auth_header)) {
    response$status_code <- 401
    response$body <- toJSON(list(error = "Missing or invalid authorization header"))
    return(FALSE)
  }
  
  token <- sub("^Bearer ", "", auth_header)
  
  tryCatch({
    claims <- jose::jwt_decode_hmac(token, secret = Sys.getenv("JWT_SECRET"))
    
    if (claims$exp < Sys.time()) {
      response$status_code <- 401
      response$body <- toJSON(list(error = "Token expired"))
      return(FALSE)
    }
    
    # Attach user info to request
    request$user <- list(
      id = claims$user_id,
      role = claims$role
    )
    
    TRUE
  }, error = function(e) {
    response$status_code <- 401
    response$body <- toJSON(list(error = "Invalid token"))
    FALSE
  })
}
```

------------------------------------------------------------------------

## **🚀 Deployment Commands**

### **Local Development**

``` bash
# Start all services with Docker Compose
docker-compose up -d

# View logs
docker-compose logs -f auth-service

# Scale specific service
docker-compose up -d --scale ml-service=5

# Stop all services
docker-compose down
```

### **Production Kubernetes**

``` bash
# Create namespace
kubectl create namespace r-microservices

# Apply configurations
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secrets.yaml

# Deploy services
kubectl apply -f k8s/deployments/

# Check status
kubectl get pods -n r-microservices
kubectl get svc -n r-microservices

# View logs
kubectl logs -f deployment/auth-service -n r-microservices

# Port forwarding for local testing
kubectl port-forward svc/auth-service 8081:80 -n r-microservices
```

------------------------------------------------------------------------

## **📈 Performance Optimization**

### **1. Connection Pooling**

``` r
# db_pool.R
library(pool)

create_db_pool <- function() {
  dbPool(
    RPostgres::Postgres(),
    host = Sys.getenv("DB_HOST"),
    dbname = Sys.getenv("DB_NAME"),
    user = Sys.getenv("DB_USER"),
    password = Sys.getenv("DB_PASSWORD"),
    minSize = 5,
    maxSize = 20,
    idleTimeout = 3600000  # 1 hour
  )
}
```

### **2. Redis Caching**

``` r
# cache.R
library(redux)

redis_conn <- redux::hiredis(
  host = Sys.getenv("REDIS_HOST"),
  port = 6379
)

cache_get <- function(key) {
  result <- redis_conn$GET(key)
  if (!is.null(result)) jsonlite::fromJSON(result)
}

cache_set <- function(key, value, ttl = 3600) {
  redis_conn$SETEX(key, ttl, jsonlite::toJSON(value))
}
```

### **3. Async Processing**

``` r
# async.R
library(future)
library(promises)

plan(multisession, workers = availableCores())

async_process <- function(data) {
  future_promise({
    # Heavy computation
    result <- complex_analysis(data)
    result
  }) %...>% {
    # Success handler
    .
  } %...!% {
    # Error handler
    list(error = as.character(.))
  }
}
```

------------------------------------------------------------------------

## **🧪 Testing Strategy**

### **Unit Tests**

``` r
# tests/test_auth_service.R
library(testthat)
library(httr)

test_that("Login endpoint returns valid JWT", {
  response <- POST(
    "http://localhost:8081/api/v1/auth/login",
    body = list(username = "test@example.com", password = "password123"),
    encode = "json"
  )
  
  expect_equal(status_code(response), 200)
  
  content <- content(response)
  expect_true("token" %in% names(content))
  expect_true(nchar(content$token) > 0)
})
```

### **Integration Tests**

``` r
# tests/integration/test_e2e_flow.R
test_that("End-to-end prediction flow works", {
  # 1. Authenticate
  auth_response <- POST(
    "http://localhost:8081/api/v1/auth/login",
    body = list(username = "test", password = "test"),
    encode = "json"
  )
  token <- content(auth_response)$token
  
  # 2. Make prediction
  pred_response <- POST(
    "http://localhost:8082/api/v1/predict",
    add_headers(Authorization = paste("Bearer", token)),
    body = list(
      model_type = "classification",
      features = data.frame(x1 = 1:10, x2 = rnorm(10))
    ),
    encode = "json"
  )
  
  expect_equal(status_code(pred_response), 200)
  expect_true("predictions" %in% names(content(pred_response)))
})
```

### **Load Testing with k6**

``` javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '1m', target: 100 },
    { duration: '3m', target: 100 },
    { duration: '1m', target: 200 },
    { duration: '3m', target: 200 },
    { duration: '1m', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const url = 'http://localhost:8082/api/v1/predict';
  const payload = JSON.stringify({
    model_type: 'classification',
    features: { x1: [1, 2, 3], x2: [4, 5, 6] }
  });
  
  const params = {
    headers: { 'Content-Type': 'application/json' },
  };
  
  let response = http.post(url, payload, params);
  
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  sleep(1);
}
```

------------------------------------------------------------------------

## **🎯 Key Features Summary**

✅ **Modern R Frameworks**: plumber2 & RestRserve (2025-ready)  
✅ **High Performance**: Async processing, connection pooling, caching  
✅ **Container Native**: Docker & Kubernetes ready  
✅ **Observability**: Prometheus metrics, distributed tracing, logging  
✅ **Security**: JWT auth, API gateway, service mesh  
✅ **Scalability**: Horizontal auto-scaling, load balancing  
✅ **Cloud Native**: 12-factor app principles, health checks  
✅ **DevOps Ready**: CI/CD pipelines, IaC, monitoring

------------------------------------------------------------------------

## **📚 Additional Resources**

- **plumber2 Documentation**: <https://www.rplumber.io/>
- **RestRserve**: <https://restrserve.org/>
- **Kubernetes R Tutorial**: <https://kubernetes.io/docs/tutorials/>
- **Docker R Images**: <https://hub.docker.com/r/rocker/>

------------------------------------------------------------------------
