# nodejs-echo

An example function which echos Hasura GraphQL Engine Event payload.

Deploy the function:

```bash
gcloud beta functions deploy nodejs-echo \
       --runtime nodejs8 \
       --trigger-http
```

Get the trigger URL:
```yaml
httpsTrigger:
  url: https://us-central1-hasura-test.cloudfunctions.net/nodejs-echo
```

Open Hasura console, goto `Events -> Add Trigger` and create a new trigger:
```
Trigger name: profile_change
Schema/Table: public/profile
Operations: Insert, Update, Delete
Webhook URL: [Trigger URL]
```

Once the trigger is created, goto `Data -> profile -> Insert row` and add a row. 
