# Setup Orchestrate, Elasticsearch on TechZone 

This is a documentation about how to set up and use the web crawler in Elasticsearch and connect it to watsonx Orchestrate for Conversational Search.

It was written using the information from: [The Assistant Toolkit]([https://www.elastic.co/guide/en/machine-learning/current/ml-nlp-elser.html#download-deploy-elser](https://github.com/watson-developer-cloud/assistant-toolkit/blob/master/integrations/extensions/docs/elasticsearch-install-and-setup/how_to_use_web_crawler_in_elasticsearch.md)

This guide was written also using the original demo Michael Blumenthal <michael.blumenthal@ibm.com> created and testing and updates by Umar Khan <umar@ibm.com>. 

## Table of contents:

- [Setup Orchestrate, Elasticsearch on TechZone](#setup-orchestrate-elasticsearch-on-techzone)
  - [Table of contents:](#table-of-contents)
  - [Step 1: Provision TechZone environments](#step-1-provision-techzone-environments)
    - [Confirming content extraction enabled](#confirming-content-extraction-enabled)
    - [Confirming elseir](#confirming-elseir)
  - [Step 2: Create and configure a web crawler in Elasticsearch](#step-2-create-and-configure-a-web-crawler-in-elasticsearch)
  - [Step 3: Build an ELSER ingest pipeline with a chunking processor](#step-3-build-an-elser-ingest-pipeline-with-a-chunking-processor)
    - [Update the index mappings of your web crawler](#update-the-index-mappings-of-your-web-crawler)
    - [Build a custom ingest pipeline with two processors](#build-a-custom-ingest-pipeline-with-two-processors)
  - [Step 4: Connect a web crawler index to watsonx Orchestrate for conversational search](#step-4-connect-a-web-crawler-index-to-watsonx-orchestrate-for-conversational-search)
    - [Using built-in Search integration](#using-built-in-search-integration)
    - [Model Configuration \& Model Prompt](#model-configuration--model-prompt)
  - [Alternative: Configure Elastic Search using Dev Tools](#alternative-configure-elastic-search-using-dev-tools)


## Step 1: Provision TechZone environments 

Provision the following TechZone environments:

* Watsonx Orchestrate: https://techzone.ibm.com/my/reservations/create/66b37c3a4425f0001ef667a4

* watsonx Discovery with Assistant, AI, and Speech (Elastic Search): https://techzone.ibm.com/my/reservations/create/6696a8ada7b42e001ef21b52

Details for accessing both environments will be the techzone reservation details. 

### Confirming content extraction enabled

Ensure Deployment wide content extraction which will allow extraction searchable content from binary files, like PDFs and Word documents.

After opening the page, you'll see a menu on the left side. Underneath `search`, you will be able to press `overview`. Select `indices`, then select the "Default" setting bottom on the top right on the page. 

  <img src="images/2-default-setting-enable-content.png" />  

### Confirming Deployment of ELSER

Machine Learning -> Model Management -> Trained Models you should see elser model deployed. 

<img src="images/3-trained-models-elser.png" />  

If it's not, see: [download-deploy-elser instructions](https://www.elastic.co/guide/en/machine-learning/current/ml-nlp-elser.html#download-deploy-elser)
 
 Depending on your Elasticsearch version, you can choose to deploy either ELSER v1 or v2 model. .elser_model_2_linux-x86_64 is an optimized version of the ELSER v2 model and is preferred to use if it is available. Otherwise, use .elser_model_2 for the regular ELSER v2 model or .elser_model_1 for ELSER v1.


## Step 2: Create and configure a web crawler in Elasticsearch 

* In Kibana, navigate to the `Search Overview` page by clicking on `Search` from the home page, and you will see `Web Crawler` 
as an option to ingest content. Choose `Web Crawler` option, click on `Start`, and follow the steps to create a Web Crawler index. 


* On the `Manage Domains` tab, add a domain, for example, `https://www.nationalparks.org`.  
  NOTE: If the domain you are crawling has pages that require authentication, you can manage the authentication settings 
  in the Kibana UI. The web crawler supports two authentication methods:
  1. Basic authentication (username and password)
  2. Authentication header (e.g. bearer tokens)  
  
  Please refer to [the Elastic documentation](https://www.elastic.co/guide/en/enterprise-search/current/crawler-managing.html#crawler-managing-authentication) 
  for more details about Authentication. 


* Add an entry point under the domain `Entry points` tab, for example, `https://www.nationalparks.org/explore/parks`


* From the domain `Crawl Rules` tab, add two additional rules like below:  
  <img src="images/add_crawl_rules_for_web_crawler.png" width="802" height="309" />  
  NOTE: Open a few URLs which you wish to be crawled. If you see any commonalities within their URL, add them to the allow rule. In this case, the two rules we created tell the web crawler to only crawl URLs that begin with https://www.nationalparks.org/explore/parks. 
  Learn more about crawl rules from [here](https://www.elastic.co/guide/en/app-search/current/web-crawler-reference.html#web-crawler-reference-crawl-rule)


* Don't start the web crawler (by clicking on `Crawl` in the upper right corner) yet, because that would cause it to 
  start ingesting with the default index mappings and pipeline. Instead, continue on to the next section to build a 
  custom ingest pipeline before starting the crawl. 


## Step 3: Build an ELSER ingest pipeline with a chunking processor

### Update the index mappings of your web crawler

Define your web crawler index name as an environment variable:  
```shell
ES_INDEX_NAME=<your-web-crawler-index-name>
ES_PORT=<found in TechZone reservation information>
ES_URL=<found in TechZone reservation information>
ES_USER=<found in TechZone reservation information>
ES_PASSWORD=<found in TechZone reservation information>
```

Update the index mappings:  

```shell
curl -X PUT "${ES_URL}:${ES_PORT}/${ES_INDEX_NAME}/_mapping?pretty" -k \
-u "${ES_USER}:${ES_PASSWORD}" \
-H 'Content-Type: application/json' \
-d'
{
  "properties": {
    "passages": {
      "type": "nested",
      "properties": {
        "sparse.tokens": {
          "type": "sparse_vector"
        }
      }
    }
  }
}'
```

The above command will update the index mappings to specify `passages` to be `nested` type and `passages.sparse.tokens` to be `sparse_vector` type, because the default dynamic index mappings of the web crawler cannot recognize these types correctly during document ingestion.

NOTE: `sparse_vector` type is for the ELSER v2 model. For ELSER v1, please use `rank_features` type. ELSER v2 has become available since Elasticsearch 8.11. It is preferred to use ELSER v2 if it is avaiable. Learn more about ELSER v2 from [here](https://www.elastic.co/guide/en/machine-learning/current/ml-nlp-elser.html)

### Build a custom ingest pipeline with two processors
Now you can build a custom ingest pipeline for your web crawler index on Kibana, following these steps:

* Open http://localhost:5601 or https://localhost:5601 (for Kibana port-forwarded from CloudPak) and log into Kibana with your Elasticsearch credentials. Navigate to the indices page
  from the left-side menu via `Content` under `Search`, find your web crawler index, and click on it to go to the index page.


* Under `Pipelines` tab, click on `Copy and customize` to create a custom ingest pipeline, and you will see a new ingest pipeline named `<your-web-crawler-index-name>@custom`.  
  For example,  
  <img src="images/web_crawler_ingest_pipeline_custom.png" width="955" height="340" />


* Click on `edit pipeline` and then `Manage` -> `Edit`, it will take you to the ingest pipeline `Edit` page 
  where you can add processors to the pipeline.


* Add a `Script` processor for chunking  
  In the ingest pipeline page, click on `Add a processor`, choose `Script` processor, and then add [a painless script](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-scripting-painless.html) to the `Source` field.  
  For example,  
  <img src="images/web_crawler_script_processor.png" width="577" height="718" />

    ```Java
    if (ctx['body_content'] == null) {
        ctx['body_content'] = ctx['body'];
    }
    if (ctx['title'] == null) {
        String url = ctx['url'];
        int lastSlashIndex = url.lastIndexOf("/");
        String fileName = url.substring(lastSlashIndex + 1);
        fileName = fileName.replace("%20", "_");
        ctx['title'] = fileName;
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

    This script splits the `body_content` into sentences using regex and combines them into `passages`. The maximum number of characters in each paggase is controlled by the `model_limit` parameter. There is a overlapping between two adjacent passages, and it is controled by the `overlap_percentage` parameter. So, `model_limit` and `overlap_percentage` need to be configured in the `Parameters` field, for example, 
    
    ```json
    {
      "model_limit": 2048,
      "overlap_percentage": 0.25
    }
    ```
    NOTE: Chunking with 512 tokens and 25% overlapping was shown to perform better than other strategies in the experiment results published in [this blog](https://techcommunity.microsoft.com/t5/ai-azure-ai-services-blog/azure-ai-search-outperforming-vector-search-with-hybrid/ba-p/3929167), so the above `Parameters` configuration is recommeded for your chunking processor. 



* Add a `Foreach` processor to process chunked texts using the ELSER model  
  In the ingest pipeline `Edit` page, click on `Add a processor`, choose `Foreach` processor, specify `passages` as the `Field`, and then add a JSON processor config to the `Processors` field.  
  For example,  
  <img src="images/web_crawler_foreach_processor.png" width="575" height="716" />
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
                "pipeline": "search-crawler-with-chunking@custom",
                "timestamp": "{{{ _ingest.timestamp }}}"
              }
            ]
          }
        }
      ]
    }
  }
  ```
  NOTES:
  * `.elser_model_2_linux-x86_64` is an optimized version of the ELSER v2 model and is preferred to use if it is available. Otherwise, use `.elser_model_2` for the regular ELSER v2 model or `.elser_model_1` for ELSER v1.
  * `inference_config.text_expansion` is required in the config to tell the Foreach processor to use `text_expansion` 
    and store the results in `tokens` field for each chunked text.
  * `_ingest._value.sparse` expects a `sparse` field for each chunk object as the target field.
  * `_ingest._value.text` expects a `text` field for each chunk object as the input field.
  * `"_ingest._value.text": "text_field"` means ELSER uses `text_field` as the input field. You may need to update it if your ELSER input field is different.
  * `search-crawler-with-chunking@custom` is the name of the ingest pipeline. You need to update it with your ingest pipeline name.  
  

  > ⛔️
  > **Caution**  
  > Don't forget to click on `Save pipeline` to save your changes!


You can achieve the same with the following curl: 

```shell
curl -X PUT "${ES_URL}:{ES_PORT}/_ingest/pipeline/${ES_INDEX_NAME}@custom/_mapping?pretty" -k \
-u "${ES_USER}:${ES_PASSWORD}" \
-H "kbn-xsrf: reporting \ 
-H 'Content-Type: application/json' \
-d'
{
  "description": "Customizable ingest pipeline for the '\''${ES_INDEX_NAME}'\'' index",
  "version": 1,
  "processors": [
    {
      "script": {
        "source": "if (ctx['\''body_content'\''] == null) {\n   ctx['\''body_content'\''] = ctx['\''body'\''];\n}\nif (ctx['\''title'\''] == null) {\n   String url = ctx['\''url'\''];\n   int lastSlashIndex = url.lastIndexOf(\"/\");\n   String fileName = url.substring(lastSlashIndex + 1);\n   fileName = fileName.replace(\"%20\", \"_\");\n   ctx['\''title'\''] = fileName;\n}\n\nString[] envSplit = /((?<!M(r|s|rs)\\.)(?<=\\.) |(?<=\\!) |(?<=\\?) )/.split(ctx['\''body_content'\'']);\n\nctx['\''passages'\''] = [];\n\nStringBuilder overlappingText = new StringBuilder();\n\nint i = 0;\nwhile (i < envSplit.length) {\n    StringBuilder passageText = new StringBuilder(envSplit[i]);\n    int accumLength = envSplit[i].length();\n    ArrayList accumLengths = [];\n    accumLengths.add(accumLength);\n\n    int j = i + 1;\n    while (j < envSplit.length && passageText.length() + envSplit[j].length() < params.model_limit) {\n        passageText.append('\'' '\'').append(envSplit[j]);\n        accumLength += envSplit[j].length();\n        accumLengths.add(accumLength);\n        j++;\n    }\n\n    ctx['\''passages'\''].add(['\''text'\'': overlappingText.toString() + passageText.toString(), '\''title'\'': ctx['\''title'\''], '\''url'\'': ctx['\''url'\'']]);\n    def startLength = passageText.length() * (1 - params.overlap_percentage) + 1;\n\n    int k = Collections.binarySearch(accumLengths, (int)startLength);\n    if (k < 0) {\n        k = -k - 1;\n    }\n    overlappingText = new StringBuilder();\n    for (int l = i + k; l < j; l++) {\n        overlappingText.append(envSplit[l]).append('\'' '\'');\n    }\n\n    i = j;\n}\n\n",
        "params": {
          "model_limit": 2048,
          "overlap_percentage": 0.25
        }
      }
    },
    {
      "foreach": {
        "field": "passages",
        "processor": {
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
                      "message": "Processor '\''inference'\'' in pipeline '\''{{ _ingest.on_failure_pipeline }}'\'' failed with message '\''{{ _ingest.on_failure_message }}'\''",
                      "pipeline": "search-search-building-regulation-documents-3@custom",
                      "timestamp": "{{{ _ingest.timestamp }}}"
                    }
                  ]
                }
              }
            ]
          }
        }
      }
    }
  ]
}'
```



* Start your web crawler and monitor its progress  
  Once you have added the processors to your ingest pipeline, you can kick off your web crawler to crawl the website URLs 
  you have configured at earlier steps, following these steps:  
  * Go to your web crawler index page, and click on `Crawl` in the upper right corner to start it.  

  * You will see new crawl requests on the overview page, and you can click on the request ids to see more details and to monitor the progress of your crawl requests.
    <img src="images/web_crawler_overview_with_crawl_requests.png" width="966" height="563" />  
  
  * If you see your crawler is running and the number of documents is increasing, you can now inspect the documents to 
    see if they have the expected fields with content. For example, you should see chunked `passages`, each passage with
    `sparse.tokens` and a chunked `text`,  
    <img src="images/web_crawl_with_chunking_document_example.png" width="920" height="503">  
    Learn more about the web crawler from [Elastic documentation](https://www.elastic.co/guide/en/enterprise-search/current/crawler.html) to improve or customize your web crawler.

  * If you don't see expected documents with chunked passages, or you want to update any processors in the ingest 
    pipeline to customize your web crawler, you may need to delete the existing documents first before starting the crawl again. 
    You can delete all the documents in an index using the following cURL command:
    ```shell
    curl -k -X POST "${ES_URL}/${ES_INDEX_NAME}/_delete_by_query" \
    -u "${ES_USER}:${ES_PASSWORD}" \
    -H 'Content-Type: application/json' -d'
    {
       "query":{
         "match_all":{}
       }
    }'
    ```


* Run a nested `text_expansion` query using cURL
  ```shell
  curl -k -X GET "${ES_URL}:${ES_PORT}/${ES_INDEX_NAME}/_search?pretty" \
  -u "${ES_USER}:${ES_PASSWORD}" \
  -H 'Content-Type: application/json' -d'
  {
    "query": {
      "nested": {
        "path": "passages",
        "query": {
          "text_expansion": {
            "passages.sparse.tokens": {
              "model_id": ".elser_model_2_linux-x86_64",
              "model_text": "Tell me about Acadia"
            }
          }
        },
        "inner_hits": {"_source": {"excludes": ["passages.sparse"]}}
      }
    }
  }'
  ```
  The above command sends a nested query to the Elasticsearch index. `text_expansion` is used in this query on the ELSER tokens 
  generated for each chunked text by the ingest pipeline. So, the search happens on the chunked texts. Learn more about nested 
  query from the [Elastic documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-nested-query.html).

  If you see successful results from the above query, you have successfully created a web crawler index with an ELSER ingest pipeline with a chunking processor.
  

  NOTES:
  * If you run this query while the crawling is taking place, you might get a timeout error, because the ELSER model is busy indexing and thus might not respond quickly enough to your query. If that happens, you should just wait until the crawl finishes.
  * `.elser_model_2_linux-x86_64` is an optimized version of the ELSER v2 model and is preferred to use if it is available. Otherwise, use `.elser_model_2` for the regular ELSER v2 model or `.elser_model_1` for ELSER v1.

## Step 4: Connect a web crawler index to watsonx Orchestrate for conversational search 

<!-- TODO: update this section with some more information -->

### Using built-in Search integration

To configure your web crawler index in the built-in search integration, you need to follow the [product documentation](https://cloud.ibm.com/docs/watson-assistant?topic=watson-assistant-search-elasticsearch-add) to set up the search integration first.  

Importantly, you need to use the right fields to configure your result content (In this guide, use `title` for Title and `text` for Body). You also need to use the right query body to make the search integration work with your web crawler index. Here is an screenshot of the configuration:  

<img src="images/use_nested_query_in_search_integration_settings.png" width="512" height="641">

Here is the query body you need in the `Advanced Elasticsearch Settings` to search over the chunked passages:
```json
{
  "query": {
    "nested": {
      "path": "passages",
      "query": {
        "text_expansion": {
          "passages.sparse.tokens": {
            "model_id": ".elser_model_2_linux-x86_64",
            "model_text": "$QUERY"
          }
        }
      },
      "inner_hits": {"_source": {"excludes": ["passages.sparse"]}}
    }
  },
  "_source": false
}
```
Notes:
* `passages` is the nested field that stores nested documents. You may need to update it if you use a different nested field in your index.
* `passages.sparse.tokens` refers to the field that stores the ELSER tokens for the nested documents.
* `"inner_hits": {"_source": {"excludes": ["passages.sparse"]}}` is to exclude the ELSER tokens from the nested documents in the search results.
* `"_source": false` is to exclude the unnecessary top-level fields in the search results.
* Learn more about nested queries and fields from [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-nested-query.html)



### Model Configuration & Model Prompt

<!-- TODO add information to this -->

> When you enable this feature, search results are provided to an IBM watsonx generative AI model that produces a conversational reply to a user’s question.
> Important: The watsonx generative AI model is hosted only in the Dallas and Frankfurt regions.

https://cloud.ibm.com/docs/watson-assistant?topic=watson-assistant-conversational-search#conversational-search-setup

I can see in the ochestrate / watson assistant settings you can configure the following

    * llama-3-2-90b-vision-instruct
    * granite-3-8b-instruct (recommended)
    * granite-3-2b-instruct
    * llama-3-3-70b-instruct
    * llama-3-2-90b-vision-instruct
    * llama-3-2-11b-vision-instruct
    * llama-3-1-70b-instruct
    * llama-3-1-8b-instruct
    * llama-3-405b-instruct
    * mistral-large
    * mixtral-8x7b-instruct-v01




## Alternative: Configure Elastic Search using Dev Tools

Here are some useful commands you can enter in dev tools to help manage the index. 

Content -> Indices -> And Select `Console` at the bottom of the page. 

```shell

# Create web index crawler index can only be done via the UI

# Update the index. 
POST /${ES_INDEX_NAME}/_mapping 
{
  "properties": {
    "passages": {
      "type": "nested",
      "properties": {
        "sparse.tokens": {
          "type": "sparse_vector"
        }
      }
    }
  }
}

# Create custom pipeline in the UI then run this to update it.
PUT _ingest/pipeline/${ES_INDEX_NAME}@custom
{
  "description": "Customizable ingest pipeline for the '${ES_INDEX_NAME}' index",
  "version": 1,
  "processors": [
    {
      "script": {
        "source": "if (ctx['body_content'] == null) {\n   ctx['body_content'] = ctx['body'];\n}\nif (ctx['title'] == null) {\n   String url = ctx['url'];\n   int lastSlashIndex = url.lastIndexOf(\"/\");\n   String fileName = url.substring(lastSlashIndex + 1);\n   fileName = fileName.replace(\"%20\", \"_\");\n   ctx['title'] = fileName;\n}\n\nString[] envSplit = /((?<!M(r|s|rs)\\.)(?<=\\.) |(?<=\\!) |(?<=\\?) )/.split(ctx['body_content']);\n\nctx['passages'] = [];\n\nStringBuilder overlappingText = new StringBuilder();\n\nint i = 0;\nwhile (i < envSplit.length) {\n    StringBuilder passageText = new StringBuilder(envSplit[i]);\n    int accumLength = envSplit[i].length();\n    ArrayList accumLengths = [];\n    accumLengths.add(accumLength);\n\n    int j = i + 1;\n    while (j < envSplit.length && passageText.length() + envSplit[j].length() < params.model_limit) {\n        passageText.append(' ').append(envSplit[j]);\n        accumLength += envSplit[j].length();\n        accumLengths.add(accumLength);\n        j++;\n    }\n\n    ctx['passages'].add(['text': overlappingText.toString() + passageText.toString(), 'title': ctx['title'], 'url': ctx['url']]);\n    def startLength = passageText.length() * (1 - params.overlap_percentage) + 1;\n\n    int k = Collections.binarySearch(accumLengths, (int)startLength);\n    if (k < 0) {\n        k = -k - 1;\n    }\n    overlappingText = new StringBuilder();\n    for (int l = i + k; l < j; l++) {\n        overlappingText.append(envSplit[l]).append(' ');\n    }\n\n    i = j;\n}\n\n",
        "params": {
          "model_limit": 2048,
          "overlap_percentage": 0.25
        }
      }
    },
    {
      "foreach": {
        "field": "passages",
        "processor": {
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
                      "pipeline": "search-search-building-regulation-documents-3@custom",
                      "timestamp": "{{{ _ingest.timestamp }}}"
                    }
                  ]
                }
              }
            ]
          }
        }
      }
    }
  ]
}


# Add domains manaully and run the web crawl. 

# Query to test index is working: 
GET /${ES_INDEX_NAME}/_search?pretty
{
 "query": {
   "nested": {
     "path": "passages",
     "query": {
       "text_expansion": {
         "passages.sparse.tokens": {
           "model_id": ".elser_model_2_linux-x86_64",
           "model_text": "How are Ala Kahakai trail segments managed?"
         }
       }
     },
     "inner_hits": {"_source": {"excludes": ["passages.sparse"]}}
     }
   }
} 



# Query to delete documents if needing to start again. 

# POST /${ES_INDEX_NAME}/_delete_by_query
#{
#   "query":{
#     "match_all":{}
#   }
#}'

```
