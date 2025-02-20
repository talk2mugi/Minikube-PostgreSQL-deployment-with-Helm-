# Minikube PostgreSQL Deployment Report

## Overview
This repository contains all necessary scripts and documentation for deploying a PostgreSQL database cluster with a load balancer on a Minikube Kubernetes cluster in a Windows local environment.

## Prerequisites
Ensure you have the following installed:
- Minikube
- Docker Desktop
- kubectl
- Helm
- PostgreSQL client

## Step 1: Start Minikube
```sh
minikube start --driver=docker
```

## Step 2: Deploy PostgreSQL Database Cluster using Helm
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install pg-cluster bitnami/postgresql-ha --set postgresql.replicaCount=2
```

## Step 3: Verify the Deployment
```sh
kubectl get pods
kubectl get svc
```

## Step 4: Expose PostgreSQL using a Load Balancer
```sh
kubectl expose statefulset pg-cluster-postgresql --type=LoadBalancer --port=5432 --name=pg-loadbalancer
```

## Step 5: Deploy a Standalone PostgreSQL Database for Replication
```sh
helm install pg-standalone bitnami/postgresql --set postgresqlPassword=adminpassword
```

## Step 6: Setup Asynchronous Replication
1. Access the primary database:
```sh
kubectl exec -it pg-cluster-postgresql-0 -- bash
psql -U postgres -d mydb
```
2. Enable replication in `postgresql.conf`:
```sh
ALTER SYSTEM SET wal_level = replica;
ALTER SYSTEM SET max_wal_senders = 5;
ALTER SYSTEM SET wal_keep_size = '32MB';
SELECT pg_reload_conf();
```
3. Create a replication user:
```sh
CREATE ROLE replicator WITH REPLICATION PASSWORD 'replicatorpassword' LOGIN;
```
4. Configure the standby database to follow the primary:
```sh
echo "standby_mode = 'on'" > /var/lib/postgresql/data/recovery.conf
echo "primary_conninfo = 'host=<primary-ip> port=5432 user=replicator password=replicatorpassword'" >> /var/lib/postgresql/data/recovery.conf
```

## Step 7: Define Database Schema and Populate Data
### Create Tables
```sql
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100) UNIQUE NOT NULL
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INT REFERENCES customers(id)
);
```

### Insert Sample Data
Using Python:
```python
import pg8000
from faker import Faker

# Initialize Faker
fake = Faker()
MINIKUBE_IP = "192.168.49.2"  # Replace with actual Minikube IP
NODE_PORT = 30929  # the actual NodePort

conn = pg8000.connect(
    database="mydb",
    user="admin",
    password="adminpassword",
    host="127.0.0.1",  # Use localhost after port forwarding
    port=5432
)
cur = conn.cursor()

# Number of records to insert
num_records = 100
batch_size = 50  # Insert 50 rows at a time for efficiency

print("Inserting records...")

# Insert Customers
customers_data = []
for _ in range(num_records):
    customers_data.append(f"('{fake.name()}', '{fake.email()}')")

# Convert list to raw SQL
customers_sql = "INSERT INTO customers (name, email) VALUES " + ", ".join(customers_data)

# Execute batch insert
cur.execute(customers_sql)
conn.commit()
print("Inserted Customers!")

# Insert Orders
cur.execute("SELECT id FROM customers")
customer_ids = [row[0] for row in cur.fetchall()]  # Fetch customer IDs

orders_data = []
for customer_id in customer_ids:
    orders_data.append(f"({customer_id})")

# Convert list to raw SQL
orders_sql = "INSERT INTO orders (customer_id) VALUES " + ", ".join(orders_data)

# Execute batch insert
cur.execute(orders_sql)
conn.commit()
print("Inserted Orders!")
conn.commit()
# Close connection
cur.close()
conn.close()
print("✅ Data Insertion Complete!")
```

## Conclusion
This repository contains all necessary configurations and scripts to deploy a PostgreSQL database cluster with a load balancer, along with asynchronous replication to a standalone database, and schema setup with sample data population.

