# How to Setup the Box Connector

## Box Setup
This has only been tested on the free version of Box.

1. Create a personal free account on box
2. Go to developer.box.com and create a new Custom App
3. Give it a name and purpose, and select "User Authentication (OAuth 2.0)" as the authentication method
4. Scroll down to the redirect URI section and the following uri: `http://localhost:8080`
5. Take note of the "Client ID" and "Client Secret" as you will need them in a later step
6. In a new tab go to ElasticSearch and prepare to add the new box connector.
7. Follow the displayed steps to get the connector running from source, you may need to do a find for `verify_certs` in the codebase to disable ssl verification. You can also get a specific version of the code from GitHub by looking at the versions [here](https://github.com/elastic/connectors/tags) and using `git clone -b v8.15.0.0 https://github.com/elastic/connectors` for example
8. With it connected, go to the configuration section, select the box free version, enter the client id and client secret
9. To get the refresh token:
   1. Go to the following URL, inserting your client ID: `https://account.box.com/api/oauth2/authorize?response_type=code&client_id=CLIENT_ID_GOES_HERE0&redirect_uri=http://localhost:8080`
   2. Authorize the application to access box
   3. When redirected to the localhost page which will fail, extract the code from the url and then run the following command
   4. ````sh
      curl -i -X POST "https://api.box.com/oauth2/token" \
        -H 'Content-Type: application/x-www-form-urlencoded' \
        -d "client_id=CLIENT_ID_GOES_HERE" \
        -d "client_secret=CLIENT_SECRET_GOES_HERE" \
        -d "code=CODE_GOES_HERE" \
        -d "grant_type=authorization_code"```
      ````
   5. Copy the refresh token from the response and paste it in the connector configuration. If the code is invalid, get another one as it's not valid for very long
10. Setup the pipeline as with the web crawler, with the below configuration.
11. Try running a sync and check it works. It can be slow to start but should start within ~5 minutes

## Pipeline Steps

### 1 - Script Processor

```java
if (ctx['body_content'] == null) {
    ctx['body_content'] = ctx['body'];
}
if(ctx['url'] == null ){
    ctx['url'] = "https://app.box.com/file/" + ctx["id"]
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

### 2 - Foreach Processor

Field: passages

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
