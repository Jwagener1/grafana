# grafana
Repository to hold Grafana dashboards.

Perfect — this schema is actually *very* LLM-friendly already. Below is a **clean, copy-pasteable README** you can drop straight into other LLMs so they instantly understand your data model, tenancy rules, and how to query it safely.

I’ve written it in **“LLM instruction style”**: clear, explicit, and slightly bossy (models respond well to authority 😄).

---

# 📊 Device Telemetry Database – Schema README

## Purpose

This database stores **time-series telemetry and status data** for physical devices deployed across **multiple customers**.

It is designed for:

* Public-facing dashboards (e.g. Grafana)
* Multi-tenant data isolation
* Device health, performance, and operational monitoring

⚠️ **Critical rule**:
**Customers must NEVER see data belonging to other customers.**

---

## 🔐 Multi-Tenant Model (VERY IMPORTANT)

* **Tenancy is enforced via the `[devices]` table**
* Every telemetry table references devices using `device_id`
* **Customer ownership is defined by `devices.customer`**
* **All queries MUST join through `[devices]` and filter by customer**

Example tenant filter:

```sql
WHERE d.customer = @Customer
```

---

## 🧱 Core Tables Overview

### 1️⃣ `[dbo].[devices]` — Device Registry (Authoritative)

This is the **root table** for all devices.

| Column          | Description                                     |
| --------------- | ----------------------------------------------- |
| `id`            | Primary key used by all telemetry tables        |
| `serial_number` | Unique device identifier                        |
| `customer`      | Customer / tenant name (**used for isolation**) |
| `location`      | Physical site or store                          |
| `machine_name`  | Friendly device name                            |
| `created_at`    | Device registration timestamp                   |
| `updated_at`    | Last metadata update                            |

🔑 **All telemetry joins start here**

---

## 📡 Telemetry & Status Tables

All tables below:

* Are **append-only**
* Store **time-series data**
* Use both `ts_epoch` (Unix) and `ts_datetime` (SQL datetime)
* Reference devices via `device_id`

---

### 2️⃣ `[dbo].[device_status]` — Online / Offline State

Tracks whether a device is online.

| Column          | Description                                        |
| --------------- | -------------------------------------------------- |
| `status`        | e.g. `ONLINE`, `OFFLINE`                           |
| `offline_since` | Timestamp when device went offline (if applicable) |
| `ts_datetime`   | Event time                                         |

🟢 Use **latest record per device** for current state
🔴 Use history for uptime / SLA reporting

---

### 3️⃣ `[dbo].[device_application_status]` — App Health

Tracks whether the device’s main application is running.

| Column                | Description                  |
| --------------------- | ---------------------------- |
| `application_running` | `1` = running, `0` = stopped |
| `stopped_since`       | When the app stopped         |
| `ts_datetime`         | Event time                   |

Useful for:

* Crash detection
* Alerting
* MTBF calculations

---

### 4️⃣ `[dbo].[device_os_status]` — OS Version Tracking

Tracks operating system version over time.

| Column        | Description       |
| ------------- | ----------------- |
| `os_version`  | OS version string |
| `ts_datetime` | Reported time     |

Useful for:

* Fleet version compliance
* Upgrade monitoring

---

### 5️⃣ `[dbo].[device_uptime_status]` — Device Uptime

Tracks how long the device has been running.

| Column           | Description                  |
| ---------------- | ---------------------------- |
| `uptime_seconds` | Continuous uptime in seconds |
| `ts_datetime`    | Sample time                  |

Useful for:

* Reboot detection
* Stability analysis

---

### 6️⃣ `[dbo].[device_storage_status]` — Disk Usage

Tracks storage metrics per drive.

| Column          | Description                  |
| --------------- | ---------------------------- |
| `drive`         | Drive identifier (e.g. `C:`) |
| `drive_type`    | SSD, HDD, etc                |
| `total_gb`      | Total capacity               |
| `free_gb`       | Free space                   |
| `used_gb`       | Used space                   |
| `usage_percent` | Percent used                 |
| `ts_datetime`   | Sample time                  |

📈 Often visualized as gauges or thresholds

---

### 7️⃣ `[dbo].[device_statistics]` — Operational Metrics

High-volume production statistics.

| Column                          | Description             |
| ------------------------------- | ----------------------- |
| `total_items`                   | Total processed         |
| `good_read`                     | Successful reads        |
| `no_read`                       | Failed reads            |
| `no_dimension`                  | Missing dimension       |
| `no_weight`                     | Missing weight          |
| `data_sent` / `not_sent`        | Data transmission stats |
| `image_sent` / `image_not_sent` | Image transmission      |
| `item_out_of_spec`              | Rejected items          |
| `more_than_1_item`              | Multi-item errors       |
| `ts_datetime`                   | Measurement time        |

📊 Used for:

* Throughput charts
* Error rate tracking
* Operational KPIs

---

## 🧠 Query Design Rules (FOR LLMs)

When generating SQL:

1. **Always join telemetry → devices**
2. **Always filter by `devices.customer`**
3. Prefer `ts_datetime` for time filtering
4. Use window functions for “latest per device”
5. Never assume one row per device

Example pattern:

```sql
FROM device_status s
JOIN devices d ON d.id = s.device_id
WHERE d.customer = @Customer
```

---

## 📈 Typical Dashboard Patterns

* **Fleet overview** → latest status per device
* **Device drill-down** → full history for one `device_id`
* **Customer dashboards** → aggregate across devices
* **Alerts** → status changes or threshold breaches

---

## 🚫 Hard Rules

❌ Never query telemetry tables without joining `devices`
❌ Never return data for multiple customers
❌ Never infer tenancy from device_id alone

---

## ✅ Summary

* `devices` = authority
* `device_id` = foreign key everywhere
* `customer` = tenant boundary
* All telemetry is time-series
* Designed for safe public dashboards

---

If you want, next we can:

* Create a **shorter “LLM prompt version”** (1-page, ultra-compact)
* Generate **Grafana-ready SQL templates**
* Design **row-level security patterns**
* Create **alerting rules per customer**

Your schema is solid — this is the grown-up version of IoT telemetry 👌

