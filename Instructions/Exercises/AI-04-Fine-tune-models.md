---
lab:
  title: Fine-Tuning Large Language Models using Azure Databricks and Microsoft Foundry
  description: You'll gain hands-on experience fine-tuning a GPT-4.1 model with custom datasets using Azure Databricks and Microsoft Foundry, including validating token counts, submitting fine-tuning jobs, and monitoring their progress. You'll learn how to deploy your customized fine-tuned model and use it via the chat completion API for domain-specific tasks that require specialized knowledge beyond what the base model provides.
  duration: 60 minutes
  level: 400
  islab: true
  primarytopics:
    - Azure Databricks
    - Azure Portal
    - Microsoft Foundry
---

# Fine-Tuning Large Language Models using Azure Databricks and Microsoft Foundry

With Azure Databricks and Microsoft Foundry, you can fine-tune large language models on your own data to enhance domain-specific performance. In this lab, Azure Databricks serves as the development environment where you prepare data, call the Microsoft Foundry fine-tuning API, and test the resulting model. Fine-tuning lets you adapt a pre-trained base model to specialized tasks using relatively small curated datasets.

This lab will take approximately **60** minutes to complete.

> **Note**: The Azure Databricks user interface is subject to continual improvement. The user interface may have changed since the instructions in this exercise were written.

## Before you start

You'll need an [Azure subscription](https://azure.microsoft.com/free) in which you have administrative-level access.

## Create a Microsoft Foundry resource and project

If you don't already have one, create a Microsoft Foundry resource and project in your Azure subscription.

> **Note**: Creating a Foundry resource only requires a subscription, resource group, region, and name. No Key Vault or Application Insights resources are needed.

1. Sign into the **Azure portal** at `https://portal.azure.com`.
2. Use the following link to open the Foundry resource creation page: `https://portal.azure.com/#create/Microsoft.CognitiveServicesAIFoundry`
3. On the **Create** page, provide the following information on the **Basics** tab:
    - **Subscription**: *Select your Azure subscription*
    - **Resource group**: *Choose or create a resource group*
    - **Region**: *Make a **random** choice from any of the following regions*\*
        - North Central US
        - Sweden Central
    - **Name**: *A unique name of your choice*
4. Select **Review + create**, then select **Create** and wait for deployment to complete.

> \* Foundry resources are constrained by regional quotas. The listed regions include default quota for the model type(s) used in this exercise. Randomly choosing a region reduces the risk of a single region reaching its quota limit in scenarios where you are sharing a subscription with other users. In the event of a quota limit being reached later in the exercise, there's a possibility you may need to create another resource in a different region.

5. Once deployment completes, go to the deployed resource. In the left pane, under **Resource Management**, select **Keys and Endpoint**, then copy the **Endpoint** — you will use it later in this exercise.

6. In the **Overview** page, select **Go to Microsoft Foundry** to open your resource in the Foundry portal (or navigate directly to `https://ai.azure.com`).

7. In **Microsoft Foundry**, create a new **project** within your Foundry resource:
    - Select the project name in the upper-left corner, then select **Create new project**.
    - Enter a **Project name** and select **Create project**.
    - Wait for the project to be created.

8. Launch Cloud Shell and run the following two commands to get temporary authorization tokens for API testing. Keep them together with the endpoint copied previously.

    ```bash
    az account get-access-token --resource https://cognitiveservices.azure.com
    az account get-access-token --resource https://management.azure.com
    ```

    >**Note**: The first token is used for OpenAI/Cognitive Services API calls; the second is used for the Azure management API (model deployment). You only need to copy the `accessToken` field value from each and **not** the entire JSON output.

## Deploy the required model

Microsoft Foundry allows you to deploy, manage, and explore models. You'll deploy a base model that will later be fine-tuned.

> **Note**: As you use Microsoft Foundry, message boxes suggesting tasks for you to perform may be displayed. You can close these and follow the steps in this exercise.

1. In **Microsoft Foundry**, in the pane on the left, select the **Models + endpoints** page and view your existing model deployments. If you don't already have one, create a new deployment of the **gpt-4.1** model with the following settings:
    - **Deployment name**: *gpt-4.1*
    - **Deployment type**: Standard
    - **Model version**: *2025-04-14*
    - **Tokens per minute rate limit**: 10K\*
    - **Content filter**: Default
    - **Enable dynamic quota**: Disabled
    
> \* A rate limit of 10,000 tokens per minute is more than adequate to complete this exercise while leaving capacity for other people using the same subscription.

## Provision an Azure Databricks workspace

> **Tip**: If you already have an Azure Databricks workspace, you can skip this procedure and use your existing workspace.

1. Sign into the **Azure portal** at `https://portal.azure.com`.
2. Create an **Azure Databricks** resource with the following settings:
    - **Subscription**: *Select the same Azure subscription that you used to create your Microsoft Foundry resource*
    - **Resource group**: *The same resource group where you created your Microsoft Foundry resource*
    - **Region**: *The same region where you created your Microsoft Foundry resource*
    - **Name**: *A unique name of your choice*
    - **Pricing tier**: *Premium*
    - **Workspace type**: *Hybrid*
    - **Managed resource group**: *Leave as default or enter a unique name*

3. Select **Review + create** and wait for deployment to complete. Then go to the resource and launch the workspace.

## Create a cluster

Azure Databricks is a distributed processing platform that uses Apache Spark *clusters* to process data in parallel on multiple nodes. Each cluster consists of a driver node to coordinate the work, and worker nodes to perform processing tasks. In this exercise, you'll create a *single-node* cluster to minimize the compute resources used in the lab environment (in which resources may be constrained). In a production environment, you'd typically create a cluster with multiple worker nodes.

> **Tip**: If you already have a cluster with a 17.3 LTS **<u>ML</u>** or higher runtime version in your Azure Databricks workspace, you can use it to complete this exercise and skip this procedure.

1. In the Azure portal, browse to the resource group where the Azure Databricks workspace was created.
2. Select your Azure Databricks Service resource.
3. In the **Overview** page for your workspace, use the **Launch Workspace** button to open your Azure Databricks workspace in a new browser tab; signing in if prompted.

> **Tip**: As you use the Databricks Workspace portal, various tips and notifications may be displayed. Dismiss these and follow the instructions provided to complete the tasks in this exercise.

4. In the sidebar on the left, select the **(+) New** task, and then select **Cluster**.
5. In the **New Cluster** page, create a new cluster with the following settings:
    - **Cluster name**: *User Name's* cluster (the default cluster name)
    - **Policy**: Unrestricted
    - **Machine learning**: Enabled
    - **Databricks runtime**: 17.3 LTS
    - **Use Photon Acceleration**: <u>Un</u>selected
    - **Worker type**: Standard_D4ds_v5
    - **Single node**: Checked
    - **Terminate after**: 30 minutes of inactivity

6. Wait for the cluster to be created. It may take a minute or two.

> **Note**: If your cluster fails to start, your subscription may have insufficient quota in the region where your Azure Databricks workspace is provisioned. See [CPU core limit prevents cluster creation](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit) for details. If this happens, you can try deleting your workspace and creating a new one in a different region.

## Create a new notebook and ingest data

1. In the sidebar, use the **(+) New** link to create a **Notebook**. In the **Connect** drop-down list, select your cluster if it is not already selected. If the cluster is not running, it may take a minute or so to start.

1. In the first cell of the notebook, enter the following SQL query to create a new volume that will be used to store this exercise's data within your default catalog:

    ```python
   %sql 
   CREATE VOLUME <catalog_name>.default.fine_tuning;
    ```

1. Replace `<catalog_name>` with the name of your default catalog. You can verify its name by selecting **Catalog** in the sidebar.
1. Use the **&#9656; Run Cell** menu option at the left of the cell to run it. Then wait for the Spark job run by the code to complete.
1. In a new cell, run the following code which uses a *shell* command to download data from GitHub into your Unity catalog.

    ```python
   %sh
   wget -O /Volumes/<catalog_name>/default/fine_tuning/training_set.jsonl https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/training_set.jsonl
   wget -O /Volumes/<catalog_name>/default/fine_tuning/validation_set.jsonl https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/validation_set.jsonl
    ```

3. In a new cell, run the following code with the access information you copied at the beginning of this exercise to assign persistent environment variables for authentication when using Microsoft Foundry:

    ```python
   import os

   os.environ["AZURE_OPENAI_ENDPOINT"] = "your_foundry_endpoint"
   os.environ["COGNITIVE_SERVICES_TOKEN"] = "your_cognitiveservices_access_token"  # from: az account get-access-token --resource https://cognitiveservices.azure.com
   os.environ["MANAGEMENT_TOKEN"] = "your_management_access_token"                # from: az account get-access-token --resource https://management.azure.com
    ```
     
## Validate token counts

Both `training_set.jsonl` and `validation_set.jsonl` are made of different conversation examples between `user` and `assistant` that will serve as data points for training and validating the fine-tuned model. While the datasets for this exercise are considered small, it is important to keep in mind when working with bigger datasets that the LLMs have a maximum context length in terms of tokens. Therefore, you can verify the token count of your datasets before training your model and revise them if necessary. 

1. In a new cell, run the following code to validate the token counts for each file:

    ```python
   import json
   import tiktoken
   import numpy as np
   from collections import defaultdict

   encoding = tiktoken.get_encoding("cl100k_base")

   def num_tokens_from_messages(messages, tokens_per_message=3, tokens_per_name=1):
       num_tokens = 0
       for message in messages:
           num_tokens += tokens_per_message
           for key, value in message.items():
               num_tokens += len(encoding.encode(value))
               if key == "name":
                   num_tokens += tokens_per_name
       num_tokens += 3
       return num_tokens

   def num_assistant_tokens_from_messages(messages):
       num_tokens = 0
       for message in messages:
           if message["role"] == "assistant":
               num_tokens += len(encoding.encode(message["content"]))
       return num_tokens

   def print_distribution(values, name):
       print(f"\n##### Distribution of {name}:")
       print(f"min / max: {min(values)}, {max(values)}")
       print(f"mean / median: {np.mean(values)}, {np.median(values)}")

   files = ['/Volumes/<catalog_name>/default/fine_tuning/training_set.jsonl', '/Volumes/<catalog_name>/default/fine_tuning/validation_set.jsonl']

   for file in files:
       print(f"File: {file}")
       with open(file, 'r', encoding='utf-8') as f:
           dataset = [json.loads(line) for line in f]

       total_tokens = []
       assistant_tokens = []

       for ex in dataset:
           messages = ex.get("messages", {})
           total_tokens.append(num_tokens_from_messages(messages))
           assistant_tokens.append(num_assistant_tokens_from_messages(messages))

       print_distribution(total_tokens, "total tokens")
       print_distribution(assistant_tokens, "assistant tokens")
       print('*' * 75)
    ```

As a reference, the model used in this exercise, GPT-4.1, has a context limit of 1,047,576 tokens (though standard deployments are capped at 300,000 tokens).

## Upload fine-tuning files to Microsoft Foundry

Before you start to fine-tune the model, you need to initialize an OpenAI client and add the fine-tuning files to its environment, generating file IDs that will be used to initialize the job.

> **Important**: When running code in Azure Databricks, `DefaultAzureCredential` authenticates as the **Databricks workspace managed identity**, not your signed-in user account. This means assigning the **Cognitive Services OpenAI Contributor** role to your own account is not sufficient — the managed identity also needs the role, or you must authenticate using a user token directly. The code below uses `azure_ad_token` with the `TEMP_AUTH_TOKEN` you obtained earlier via Cloud Shell, which runs as your own identity and avoids this issue.

1. In a new cell, run the following code:

     ```python
    import os
    from openai import AzureOpenAI

    client = AzureOpenAI(
      azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"),
      azure_ad_token = os.getenv("COGNITIVE_SERVICES_TOKEN"),
      api_version = "2025-04-01-preview"  # This API version or later is required to access seed/events/checkpoint features
    )

    training_file_name = '/Volumes/<catalog_name>/default/fine_tuning/training_set.jsonl'
    validation_file_name = '/Volumes/<catalog_name>/default/fine_tuning/validation_set.jsonl'

    training_response = client.files.create(
        file = open(training_file_name, "rb"), purpose="fine-tune"
    )
    training_file_id = training_response.id

    validation_response = client.files.create(
        file = open(validation_file_name, "rb"), purpose="fine-tune"
    )
    validation_file_id = validation_response.id

    print("Training file ID:", training_file_id)
    print("Validation file ID:", validation_file_id)
     ```

## Submit fine-tuning job

Now that the fine-tuning files have been successfully uploaded you can submit your fine-tuning training job. It isn't unusual for training to take more than an hour to complete. Once training is completed, you can see the results in Microsoft Foundry by selecting the **Fine-tuning** option in the left pane.

1. In a new cell, run the following code to start the fine-tuning training job:

    ```python
   response = client.fine_tuning.jobs.create(
       training_file = training_file_id,
       validation_file = validation_file_id,
       model = "gpt-4.1-2025-04-14",
       seed = 105 # seed parameter controls reproducibility of the fine-tuning job. If no seed is specified one will be generated automatically.
   )

   job_id = response.id
    ```

The `seed` parameter controls reproducibility of the fine-tuning job. Passing in the same seed and job parameters should produce the same results, but can differ in rare cases. If no seed is specified one will be generated automatically.

2. In a new cell, you can run the following code to monitor the status of the fine-tuning job:

    ```python
   print("Job ID:", response.id)
   print("Status:", response.status)
    ```

>**Note**: You can also monitor the job status in Microsoft Foundry by selecting **Fine-tuning** in the left sidebar.

3. Once the job status changes to `succeeded`, run the following code to get the final results:

    ```python
   response = client.fine_tuning.jobs.retrieve(job_id)

   print(response.model_dump_json(indent=2))
   fine_tuned_model = response.fine_tuned_model
    ```
   
## Deploy fine-tuned model

Now that you have a fine-tuned model, you can deploy it as a customized model and use it like any other deployed model in either the **Chat** playground of Microsoft Foundry, or via the chat completion API.

1. In a new cell, run the following code to deploy your fine-tuned model:
   
    ```python
   import json
   import requests

   token = os.getenv("MANAGEMENT_TOKEN")
   subscription = "<YOUR_SUBSCRIPTION_ID>"
   resource_group = "<YOUR_RESOURCE_GROUP_NAME>"
   resource_name = "<YOUR_AZURE_AI_SERVICES_RESOURCE_NAME>"  # The Azure AI Services resource name from your AI Foundry project settings
   model_deployment_name = "gpt-4.1-ft"

   deploy_params = {'api-version': "2023-05-01"}
   deploy_headers = {'Authorization': 'Bearer {}'.format(token), 'Content-Type': 'application/json'}

   deploy_data = {
       "sku": {"name": "standard", "capacity": 1},
       "properties": {
           "model": {
               "format": "OpenAI",
               "name": fine_tuned_model
           }
       }
   }
   deploy_data = json.dumps(deploy_data)

   request_url = f'https://management.azure.com/subscriptions/{subscription}/resourceGroups/{resource_group}/providers/Microsoft.CognitiveServices/accounts/{resource_name}/deployments/{model_deployment_name}'

   print('Creating a new deployment...')

   r = requests.put(request_url, params=deploy_params, headers=deploy_headers, data=deploy_data)

   print(r)
   print(r.reason)
   print(r.json())
    ```

2. In a new cell, run the following code to use your customized model in a chat completion call:
   
    ```python
   import os
   from openai import AzureOpenAI

   client = AzureOpenAI(
     azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"),
     azure_ad_token = os.getenv("COGNITIVE_SERVICES_TOKEN"),
     api_version = "2024-02-01"
   )

   response = client.chat.completions.create(
       model = "gpt-4.1-ft", # model = "Custom deployment name you chose for your fine-tuning model"
       messages = [
           {"role": "system", "content": "You are a helpful assistant."},
           {"role": "user", "content": "Does Microsoft Foundry support customer managed keys?"},
           {"role": "assistant", "content": "Yes, customer managed keys are supported by Microsoft Foundry."},
           {"role": "user", "content": "Do other Azure AI services support this too?"}
       ]
   )

   print(response.choices[0].message.content)
    ```
 
## Clean up

When you're done, remember to delete the deployment or the entire Microsoft Foundry project in **Microsoft Foundry** at `https://ai.azure.com`.

In Azure Databricks portal, on the **Compute** page, select your cluster and select **&#9632; Terminate** to shut it down.

If you've finished exploring Azure Databricks, you can delete the resources you've created to avoid unnecessary Azure costs and free up capacity in your subscription.