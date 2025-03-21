# Connecting Elastic to COS

To connect to COS you must make use of the AWS S3 connector which is available in Kibana / Elastic

Follow the steps as in the docs. The only change needed is when running the connector locally, you need to add the following environment variables: `AWS_ENDPOINT_URL` and `AWS_PORT` which needs to be the COS endpoint such as `https://s3.us-south.cloud-object-storage.appdomain.cloud` and port `443`. Set this with export in shell before running your command

```bash
export AWS_ENDPOINT_URL=https://s3.us-south.cloud-object-storage.appdomain.cloud
export AWS_PORT=443
make run
```

## Pipeline

```java
if (ctx['body_content'] == null) {
    ctx['body_content'] = ctx['body'];
}
if(ctx['url'] == null ){
    ctx['url'] = "https://s3.us-south.cloud-object-storage.appdomain.cloud/" + ctx["bucket"] + "/" + ctx["filename"]
}
if (ctx['title'] == null) {
    ctx['title'] = ctx['name'];
}

String[] envSplit = /((?<!M(r|s|rs)\.)(?<=\.) |(?<=\!) |(?<=\?) )/.split(ctx['body_content']);

ctx['passages'] = [];

StringBuilder overlappingText = new StringBuilder();

int i = 0;
while (i < envSplit.length) {
    StringBuilder passageText = new StringBuilder(envSplit[i]);
    int accumLength = envSplit[i].length();
    ArrayList accumLengths = [];
    accumLengths.add(accumLength);

    int j = i + 1;
    while (j < envSplit.length && passageText.length() + envSplit[j].length() < params.model_limit) {
        passageText.append(' ').append(envSplit[j]);
        accumLength += envSplit[j].length();
        accumLengths.add(accumLength);
        j++;
    }

    ctx['passages'].add(['text': overlappingText.toString() + passageText.toString(), 'title': ctx['title'], 'url': ctx['url']]);
    def startLength = passageText.length() * (1 - params.overlap_percentage) + 1;

    int k = Collections.binarySearch(accumLengths, (int)startLength);
    if (k < 0) {
        k = -k - 1;
    }
    overlappingText = new StringBuilder();
    for (int l = i + k; l < j; l++) {
        overlappingText.append(envSplit[l]).append(' ');
    }

    i = j;
}
```

Parameters:

```json
{
  "model_limit": 2048,
  "overlap_percentage": 0.25
}
```

ForEach Processor:

```json
{
  "inference": {
    "field_map": {
      "_ingest._value.text": "text_field"
    },
    "model_id": ".elser_model_2_linux-x86_64",
    "target_field": "_ingest._value.sparse",
    "inference_config": {
      "text_expansion": {
        "results_field": "tokens"
      }
    },
    "on_failure": [
      {
        "append": {
          "field": "_source._ingest.inference_errors",
          "value": [
            {
              "message": "Processor 'inference' in pipeline '{{ _ingest.on_failure_pipeline }}' failed with message '{{ _ingest.on_failure_message }}'",
              "pipeline": "box-architecture-docs@custom",
              "timestamp": "{{{ _ingest.timestamp }}}"
            }
          ]
        }
      }
    ]
  }
}
```
