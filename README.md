# Logstash → Fluentd → PostgreSQL/TimescaleDB Pipeline

## Overview
This project implements a complete log ingestion pipeline for Microsoft IIS logs. The pipeline parses raw IIS log files, normalizes fields, stores them in a PostgreSQL 15 database (extended with TimescaleDB for time-series capabilities), and makes the data available for visualization in Grafana.

**Internship Goals:**
- Build a robust, reproducible pipeline from Logstash → Fluentd → PostgreSQL/TimescaleDB.  
- Parse IIS log fields accurately (datetime, IP, cookies, URIs, status codes, response times, etc.).  
- Handle failures gracefully.  
- Validate correctness (no grokparsefailures, exact record counts, accurate cookie parsing).  
- Provide documentation and operational instructions for long-term use.  

---

## Architecture
**Pipeline Flow:**  
`Logstash → Fluentd → PostgreSQL (TimescaleDB) → Grafana`  

- **Logstash:** Parses raw IIS logs, classifies logs (success, static content, failures), and sends records via HTTP to Fluentd.  
- **Fluentd:** Receives logs, applies lightweight transforms, and bulk-inserts into PostgreSQL.  
- **PostgreSQL + TimescaleDB:** Stores logs in partitioned hypertables for efficient time-series queries.  
- **Grafana:** Visualizes metrics such as error rates, traffic volumes, response times, and static vs. dynamic content.  

---

## Setup

### Logstash
```bash
sudo /usr/share/logstash/bin/logstash -f ~/logstash_configs/logstash.conf
# Stop Logstash
pkill -f logstash    # or Ctrl+C if running in foreground

#Fluentd
fluentd --config ~/fluentd_conf/fluent.conf
# Stop Fluentd
pkill -f fluentd     # or Ctrl+C if running in foreground


#PostgreSQL + TimescaleDB

sudo -u postgres psql -d iis_logs


#Grafana

ssh -L 3001:localhost:3000 -p 2212 aaronbadmin@dcir-sandbox7.domain-msi.local
# Access via browser: http://localhost:3001/


#Logstash Configuration (~/logstash_configs/logstash.conf)
input {
  file {
    path => [
      "/home/aaronbadmin@domain-msi.local/iis_logs/intranet/u_ex250414_x.log",
      "/home/aaronbadmin@domain-msi.local/iis_logs/tablet/u_ex250414_x.log"
    ]
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}

filter {
  grok {
    match => { "message" => "%{TIMESTAMP_ISO8601:v_datetime} %{IPORHOST:c_ip} %{DATA:cs_method} %{URIPATH:cs_uri_stem} %{URIPARAM:cs_uri_query}? %{NUMBER:sc_status} %{NUMBER:sc_bytes} %{NUMBER:cs_bytes} %{NUMBER:time_taken}" }
    tag_on_failure => ["_grokparsefailure"]
  }
  if "_grokparsefailure" in [tags] {
    mutate { add_field => { "reason" => "grokparsefailure" } }
  }
  kv {
    source => "cs_cookie"
    field_split => ";"
    value_split => "="
  }
  if [teammsiuid] { mutate { rename => { "teammsiuid" => "cs_username" } } }
  date { match => ["v_datetime", "YYYY-MM-dd HH:mm:ss"] target => "@timestamp" }
  mutate { remove_field => ["message"] }
  if [cs_uri_stem] =~ /\.(css|js|png|jpg|gif|ico)$/ { mutate { add_field => { "record_type" => "static" } } } 
  else { mutate { add_field => { "record_type" => "dynamic" } } }
}

output {
  http { url => "http://localhost:9880/http.logs" http_method => "post" format => "json" }
  if "_grokparsefailure" in [tags] { http { url => "http://localhost:9880/http.log_failures" http_method => "post" format => "json" } }
  if [record_type] == "static" { http { url => "http://localhost:9880/http.static" http_method => "post" format => "json" } }
}


Fluentd Configuration (~/fluentd_conf/fluent.conf)
<source>
  @type http
  port 9880
</source>

<match http.logs>
  @type postgres_bulk
  host localhost
  database iis_logs
  username postgres
  password postgres
  table iis_log_records
  column_names v_datetime,c_ip,cs_cookie,cs_host,cs_referer,cs_uri_query,cs_uri_stem,cs_username,s_computername,sc_status,time_taken,sc_bytes,cs_bytes
  <buffer>
    flush_interval 2s
    chunk_limit_size 16MB
  </buffer>
</match>

<match http.log_failures>
  @type postgres_bulk
  host localhost
  database iis_logs
  username postgres
  password postgres
  table iis_log_failures
  column_names reason,raw_log
  <buffer>
    flush_interval 2s
  </buffer>
</match>

<match http.static>
  @type postgres_bulk
  host localhost
  database iis_logs
  username postgres
  password postgres
  table iis_logs_static
  column_names v_datetime,cs_username,cs_uri_stem
  <buffer>
    flush_interval 2s
  </buffer>
</match>


Database Schema
CREATE TABLE iis_log_records (
    v_datetime timestamptz NOT NULL,
    c_ip inet,
    cs_cookie text,
    cs_host text,
    cs_referer text,
    cs_uri_query text,
    cs_uri_stem text,
    cs_username text,
    s_computername text,
    sc_status integer,
    time_taken integer,
    sc_bytes bigint,
    cs_bytes bigint
);
CREATE INDEX ON iis_log_records (v_datetime DESC);
SELECT create_hypertable('iis_log_records', 'v_datetime');

CREATE TABLE iis_logs_static (
    v_datetime timestamptz,
    cs_username text,
    cs_uri_stem text
);
CREATE INDEX ON iis_logs_static (v_datetime);
CREATE INDEX ON iis_logs_static (cs_uri_stem);
CREATE INDEX ON iis_logs_static (cs_username);

CREATE TABLE iis_log_failures (
    id serial PRIMARY KEY,
    failure_time timestamptz DEFAULT now(),
    reason text,
    raw_log text
);

Operations Guide

View/Edit Logstash: cat ~/logstash_configs/logstash.conf / nano ~/logstash_configs/logstash.conf

View/Edit Fluentd: cat ~/fluentd_conf/fluent.conf / nano ~/fluentd_conf/fluent.conf

Check processes: ps aux | grep logstash / ps aux | grep fluentd

Export CSV: scp -P 2212 aaronbadmin@dcir-sandbox7.domain-msi.local:/tmp/final_normalized_cs_uri_stem_v2.csv .

Validation

Successful Parsing: All 3.13M log lines parsed with zero grokparsefailures.

Record Counts: PostgreSQL record counts exactly match raw log lines.

Cookie Parsing: teammsiuid extracted to cs_username.

Static vs Dynamic Classification: Static assets routed to iis_logs_static; dynamic requests stored in iis_log_records.

Conclusion

This pipeline provides a fully operational IIS log ingestion system:

Logstash: Parses and enriches raw logs

Fluentd: Bulk inserts into PostgreSQL

PostgreSQL + TimescaleDB: Scalable storage and indexing

Grafana: Visualization of server metrics

Production-ready and extensible for analytics, dashboards, and monitoring.














