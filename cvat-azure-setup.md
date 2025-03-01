# CVAT instance and setup instructions

## CVAT remote address

Azure VM and docker compose is used in setting up the cvat remotely.
http://52.149.15.226/

username: root
password: 123456

The audio playback is acting perfectly on my machine, but i have also noticed that machine with poor condition can act badly.

## CVAT Installation Guide for Azure VM

Similiar to running cvat locally, you can reference the official [instructions](https://docs.cvat.ai/docs/contributing/development-environment/) for detailed information.

### 1. Docker startup

```
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build \
  cvat_opa cvat_db cvat_redis_inmem cvat_redis_ondisk cvat_server
```

### 2. Check the nginx is running

Test the configuration and reload Nginx if need:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 3. Start the Backend Services

A startup script is used for the backend services:

```bash
vim /root/cvat/start_cvat.sh
```

which contains the following content:

```bash
#!/bin/bash

cd /root/cvat

source .env/bin/activate

export CVAT_SERVERLESS=1
export ALLOWED_HOSTS="*"
export DJANGO_LOG_SERVER_HOST="localhost"
export DJANGO_LOG_SERVER_PORT="8282"

nohup python manage.py runserver --noreload --insecure 0.0.0.0:7000 > /tmp/cvat_server.log 2>&1 &

nohup python manage.py rqworker import --worker-class cvat.rqworker.SimpleWorker > /tmp/cvat_import.log 2>&1 &
nohup python manage.py rqworker export --worker-class cvat.rqworker.SimpleWorker > /tmp/cvat_export.log 2>&1 &
nohup python manage.py rqworker annotation --worker-class cvat.rqworker.SimpleWorker > /tmp/cvat_annotation.log 2>&1 &
nohup python manage.py rqworker webhooks --worker-class cvat.rqworker.SimpleWorker > /tmp/cvat_webhooks.log 2>&1 &
nohup python manage.py rqworker quality_reports --worker-class cvat.rqworker.SimpleWorker > /tmp/cvat_quality.log 2>&1 &
nohup python manage.py rqworker analytics_reports --worker-class cvat.rqworker.SimpleWorker > /tmp/cvat_analytics.log 2>&1 &
nohup python manage.py rqworker cleaning --worker-class cvat.rqworker.SimpleWorker > /tmp/cvat_cleaning.log 2>&1 &
nohup python manage.py rqworker chunks --worker-class cvat.rqworker.SimpleWorker > /tmp/cvat_chunks.log 2>&1 &
nohup python manage.py rqworker consensus --worker-class cvat.rqworker.SimpleWorker > /tmp/cvat_consensus.log 2>&1 &

nohup python rqscheduler.py -i 1 > /tmp/cvat_scheduler.log 2>&1 &

# Save all process PIDs
echo "Main server PID: $!" >> /tmp/cvat_services.pid
ps aux | grep "python manage.py rqworker" | grep -v grep | awk '{print $2}' >> /tmp/cvat_services.pid
ps aux | grep "python rqscheduler.py" | grep -v grep | awk '{print $2}' >> /tmp/cvat_services.pid

echo "All CVAT services started. Logs are in /tmp/"
```

Make the script executable and run it:

```bash
chmod +x start_cvat.sh
./start_cvat.sh
```

After starting the server, check the health endpoint: http://YOUR_VM_IP_ADDRESS/api/server/health/

### 4. Start the Frontend

Start the frontend application in tmux background:

```bash
tmux
yarn run start:cvat-ui
```

CVAT installation should now be accessible at http://YOUR_VM_IP_ADDRESS

Now the frontend should be visitable.