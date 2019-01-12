# hasura-graphql-on-gcloud
Hasura GraphQL Engine on Google Cloud

## Pre-requisites

1. Google cloud account with billing enabled
2. `gcloud` CLI

## 1. Create a Google Cloud Project

## 2. Create a Google Cloud SQL Postgres instance

Create a PostgreSQL instance:

```bash
gcloud sql instances create hge-pg --database-version=POSTGRES_9_6 \
       --cpu=1 --memory=3840MiB --region=asia-south1
```

Set a password for `postgres` user:

```bash
gcloud sql users set-password postgres no-host --instance=hge-pg \
       --password=[PASSWORD]
```

Make a note of the `[PASSWORD]`.

## 3. Create a Kubernetes cluster

```bash
gcloud container clusters create hge-k8s \
      --zone asia-south1-a \
      --num-nodes 1
```

## 4. Configure and deploy GraphQL Engine

Create a service account and download the json file by following [this
guide](https://cloud.google.com/sql/docs/postgres/connect-kubernetes-engine#2_create_a_service_account).

Or if you didn't read that guide, at least ensure you:

1. Enable [Cloud SQL Admin API](https://console.developers.google.com/apis/api/sqladmin.googleapis.com/) for your project
2. Create a new [Services Account](https://console.cloud.google.com/iam-admin/serviceaccounts/)
3. Select `Cloud SQL Admin` as the role
4. Click `Create Key` to download the json file

Create a k8s secret with this service account:
```bash
kubectl create secret generic cloudsql-instance-credentials \
    --from-file=credentials.json=[PROXY_KEY_FILE_PATH]
```

Replace `[PROXY_KEY_FILE_PATH]` with the filename of the download json.

Create another secret with the database user and password
(Use the `[PASSWORD]` noted earlier):
```bash
kubectl create secret generic cloudsql-db-credentials \
    --from-literal=username=postgres --from-literal=password=[PASSWORD]
```

Get the `INSTANCE_CONNECTION_NAME`:
```bash
gcloud sql instances describe hge-pg
# it'll be something like this:
# connectionName: myproject1:us-central1:myinstance1
```

Edit `deployment.yaml` and replace `INSTANCE_CONNECTION_NAME` with the value.

Create the deployment:

```bash
kubectl create -f deployment.yaml
```

Ensure the pod is running:

```bash
kubectl get pods
```

Check logs if there are errors.

Expose GraphQL Engine on a Google LoadBalancer:

```bash
kubectl expose deploy/hasura-graphql-engine \
        --port 80 --target-port 8080 \
        --type LoadBalancer
```

Get the external IP:
```bash
kubectl get service
```

Open the console by navigating the the external IP in a browser.

Note down this IP as `HGE_IP`.

## 5. Create table

Create the table using the console:

```
Table name: profile

Columns:

id: Integer auto-increment
name: Text
address: Text
lat: Numeric, Nullable
lng: Numeric, Nullable
```

## 6. Create a Google Cloud Function

We are going to use Google Maps API in our cloud function.

Enable Google Maps API and get an API key (we'll call it `GMAPS_API_KEY`) following [this
guide](https://developers.google.com/maps/documentation/geocoding/start?hl=el#auth)

Check the `Places` box to get access to Geocoding API.

We'll follow [this guide](https://cloud.google.com/functions/docs/quickstart)
and create a Cloud Function with NodeJS 8. 

```bash
gcloud components update &&
gcloud components install beta
```

Goto the `cloudfunction` directory:

```bash
cd cloudfunction
```

Edit `.env.yaml` and add values for the following as shown:
```yaml
# .env.yaml
GMAPS_API_KEY: '[GMAPS_API_KEY]'
HASURA_GRAPHQL_ENGINE_URL: 'http://[HGE_IP]/v1alpha1/graphql'
```

```bash
gcloud beta functions deploy trigger \
       --runtime nodejs8 \
       --trigger-http \
       --region asia-south1 \
       --env-vars-file .env.yaml
```

Get the trigger URL:
```yaml
httpsTrigger:
  url: https://asia-south1-hasura-test.cloudfunctions.net/trigger
```

Goto `HGE_IP` on browser, `Events -> Add Trigger` and create a new trigger:
```
Trigger name: profile_change
Schema/Table: public/profile
Operations: Insert
Webhook URL: [Trigger URL]
```

Once the trigger is created, goto `Data -> profile -> Insert row` and add a new
profile with name and address, save. Goto `Browse rows` tabs to see lat and lng
updated, by the cloud function.
