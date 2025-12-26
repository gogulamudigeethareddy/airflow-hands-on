# airflow-hands-on
airflow-hands-on

## Getting Started

### Start the Airflow Stack

1. Build and start all services in the background:
	```sh
	docker compose up --build -d
	```

2. (Optional) If you need to re-initialize the database and admin user:
	```sh
	docker compose down
	docker compose up airflow-init
	docker compose up --build -d
	```

### Access the Airflow Web UI

Open your browser and go to: [http://localhost:8080](http://localhost:8080)

#### Login Credentials

- **Username:** airflow
- **Password:** admin

If you need to reset the password, run:
```sh
docker compose run --rm airflow-cli airflow users reset-password --username airflow --password admin
```
