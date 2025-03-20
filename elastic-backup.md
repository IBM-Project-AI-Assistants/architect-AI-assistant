# Backing Up and Restoring Elastic

In order to back up your Elastic cluster, you will need to:

- Setup a bucket in COS
- Setup the repository in Elastic using the REST API as Kibana does not support configuring the endpoint
- Create a policy to schedule backups

## Registering the repository in Elastic
In this step I assume a bucket has been created in COS, along with an HMAC access key and secret. The following curl command will register the repository, once you substitute the details of your elastic. Change any COS parameters as needed:

```bash
curl -X PUT "ELASTIC_HOST:ELASTIC_PORT/_snapshot/REPOSITORY_NAME?pretty" -u ELASTIC_USERNAME:ELASTIC_PASSWORD -H 'Content-Type: application/json' -d'
{
  "type": "s3",
  "settings": {
    "bucket": "my-bucket",
    "endpoint": "https://s3.us-south.cloud-object-storage.appdomain.cloud",
    "access_key": "<ACCESS KEY>",
    "secret_key": "<SECRET KEY>",
    "base_path": "backup/elasticsearch",
    "region": "us-south"
  }
}
'
```

## Setting up a policy

From the Kibana UI navigate to policies and select create a policy. Enter a name, snapshot name which can use expressions, for example I named mine `<daily-snapshot-{now/d}>`. Select your repository and then add a schedule. You can choose how often you want to back up by selecting from the drop down menu. Once you have selected the frequency of the backup click on next.

On this page we then choose what we want to backup, for ease select all data streams and indicies, include the global state and the feature state and press next.

Select how long you want the backups to exist in COS, for example 7 days, with a minimum or maximum number of total snapshots and press next.

Review the settings and press create policy.

You can now run your policy from the policy list to check it works as intended.

## Restoring from a snapshot
To restore from a snapshot, add the repository as above, but set the property `settings.readonly` to `true`. This means that you cannot write data into this repository.

With the repository added, you should now see the previous snapshots in the list of snapshots, and you can click the restore button to start the process. This should only be done on a blank cluster to prevent data loss.
