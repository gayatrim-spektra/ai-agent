## Exercise 2: Dev-to-Production with azd

### Estimated Duration: 60 Minutes

## Overview

In this exercise, you will take an agent from local code to a production runtime using the **Azure Developer CLI (azd)** and **Foundry Agent Service** hosted agents. You will scaffold a hosted agent project from a template, review its structure, provision the required Azure resources, and deploy the agent as a container that Foundry runs for you with sandboxed sessions and scale-to-zero. You will then ship a second version of the agent and practice rolling traffic back to the previous version.

## Objectives

In this exercise, you will complete the following tasks:

   - Task 1: Deploy with azd
   - Task 2: Versioning & Rollout

### Task 1: Deploy with azd

In this task, you will initialize a hosted agent project from a template, review the template structure, provision Azure resources with azd, and deploy the agent to Foundry Agent Service.

> **Note:** Hosted agents package your own agent code (for example, built with the **Microsoft Agent Framework**) into a container that Foundry Agent Service runs on Microsoft-managed, per-session sandboxed compute. This is the production runtime pattern this lab focuses on. The **Azure-Samples/get-started-with-ai-agents** repository demonstrates the same azd dev-to-production workflow using a self-managed Azure Container Apps web application instead; you can review it later as an alternative hosting pattern.

1. On the lab virtual machine, open **Visual Studio Code**, then open a new terminal by selecting **Terminal (1)** from the menu bar and clicking on **New Terminal (2)**.

   ![Image](./media/Ex2-Task1-Step1.png)

1. Verify that the Azure Developer CLI is installed and is version **1.25.3** or later:

   ```powershell
   azd version
   ```

   > **Note (Author review needed):** Confirm that the lab VM image ships with Azure Developer CLI 1.25.3+, Azure CLI 2.80+, and Python 3.13+ preinstalled. If not, add installation steps for these tools before this task.

1. Install the Microsoft Foundry extension for azd:

   ```powershell
   azd ext install microsoft.foundry
   ```

1. Sign in to the Azure Developer CLI:

   ```powershell
   azd auth login
   ```

1. A browser window opens for authentication. Select the account **<inject key="AzureAdUserEmail"></inject>** and complete the sign-in with the password **<inject key="AzureAdUserPassword"></inject>**, then return to the terminal and confirm you see **Logged in to Azure**.

   ![Image](./media/Ex2-Task1-Step5.png)

1. Sign in to the Azure CLI as well, since later steps use it for management operations:

   ```powershell
   az login
   ```

1. Create a working folder for the agent project and change into it:

   ```powershell
   mkdir C:\LabFiles\hosted-agent
   cd C:\LabFiles\hosted-agent
   ```

1. Initialize a new hosted agent from the basic Agent Framework sample template:

   ```powershell
   azd ai agent init -m "https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/responses/01-basic/azure.yaml" --deploy-mode code
   ```

1. The interactive flow prompts you for several values. Provide the following:

   - **Agent name:** **contoso-hosted-agent (1)**

   - **Foundry Project:** Select **Use an existing Foundry project (2)** and choose **agent-project** created in Exercise 1

   - **Tenant:** Select the default tenant **(3)**

   - **Subscription:** Select your lab subscription **(4)**

   - **Model / Model Version / Model SKU / Deployment capacity / Deployment name:** Accept the defaults for each prompt **(5)**

   ![Image](./media/Ex2-Task1-Step9.png)

   > **Note:** When the flow completes, you see the message **AI agent definition added to your azd project successfully!**

1. Change into the newly created agent folder:

   ```powershell
   cd contoso-hosted-agent
   ```

#### **Review the agent template structure**

1. In Visual Studio Code, select **File (1)** from the menu bar, click on **Open Folder (2)**, and open **C:\LabFiles\hosted-agent\contoso-hosted-agent**. Review the generated files:

   ```
   contoso-hosted-agent/
   ├── azure.yaml          # azd project definition: the azure.ai.agent service, protocols, env vars
   ├── main.py             # Agent code: Agent Framework agent served over the Responses protocol
   ├── requirements.txt    # Python dependencies, including the agent server protocol library
   └── .azure/             # azd environment state (created after provisioning)
   ```

   ![Image](./media/Ex2-Task1-Step11.png)

1. Open **azure.yaml** and review the service definition. Note the following key elements:

   * **host: azure.ai.agent**: Tells azd that this service deploys to Foundry Agent Service as a hosted agent instead of to App Service or Container Apps.
   * **protocols**: Declares which protocol the container exposes. This template uses the **responses** protocol, which is designed for conversational, streaming, multi-turn agents.
   * **env**: The map of environment variables (for example, the model deployment name) injected into the agent container at runtime. You will use this map in Task 2.

1. Open **main.py** and locate the agent definition, including the instructions text and the model reference. This is standard agent code; the protocol library wraps it in an HTTP server that Foundry's gateway can route requests to on port **8088**.

#### **Provision Azure resources and deploy**

1. Provision the Azure resources defined by the template:

   ```powershell
   azd provision
   ```

   > **Note:** Provisioning takes around **5-10 minutes** to complete. Since you selected the existing Foundry project, azd adds the remaining resources it needs, such as an Azure Container Registry and Application Insights, and wires up the required role assignments.

1. Open a new browser tab, navigate to the following URL, and sign in with your lab credentials if prompted:

   ```
   https://portal.azure.com
   ```

1. In the Azure portal, search for and select **Resource groups**, then open the resource group used by your Foundry project. Confirm that it contains the following resources:

   - The **Foundry** resource (Microsoft.CognitiveServices account) with your **agent-project**
   - An **Azure Container Registry** for the agent container images
   - An **Application Insights** resource and a **Log Analytics workspace** for telemetry
   - A **Managed Identity** used during deployment

   ![Image](./media/Ex2-Task1-Step16.png)

1. Return to the terminal in Visual Studio Code and deploy the agent container to Foundry Agent Service:

   ```powershell
   azd deploy
   ```

   > **Note:** The deployment builds your container image remotely in Azure Container Registry, pushes it, creates hosted agent **version 1**, and creates a dedicated **Microsoft Entra ID** agent identity. This process takes around **5-10 minutes** to complete.

1. When the command finishes, review the output. It includes an **Agent playground (portal)** link and the **Agent endpoint** URL. Copy both values into your notepad file.

   ![Image](./media/Ex2-Task1-Step18.png)

1. Invoke the deployed agent from the terminal:

   ```powershell
   azd ai agent invoke "Write a haiku about deploying cloud applications."
   ```

1. Verify that the agent returns a response within a few seconds.

   ![Image](./media/Ex2-Task1-Step20.png)

1. Inspect the deployed agent's details, including its name, active version, protocols, and environment variables:

   ```powershell
   azd ai agent show
   ```

> **Congratulations** on completing the task! Now, it's time to validate it. Here are the steps:
> - Hit the Validate button for the corresponding task. If you receive a success message, you can proceed to the next task. 
> - If not, carefully read the error message and retry the step, following the instructions in the lab guide.
> - If you need any assistance, please contact us at cloudlabs-support@spektrasystems.com. We are available 24/7 to help you out.

<validation step="REPLACE-WITH-VALIDATION-GUID" />

### Task 2: Versioning & Rollout

In this task, you will configure a per-version environment variable, deploy a second version of the agent, verify the new behavior, and then roll traffic back to version 1.

Every `azd deploy` creates a new immutable **version** of the hosted agent. Versions keep their own container image, environment variables, and configuration, and the agent's endpoint routes traffic across versions through a **version selector**. This enables safe rollouts and instant rollbacks.

1. In Visual Studio Code, open **azure.yaml** and add a new environment variable to the **env** map of the agent service, keeping the existing entries unchanged:

   ```yaml
   env:
     AGENT_VERSION_LABEL: v2
   ```

   ![Image](./media/Ex2-Task2-Step1.png)

   > **Note:** Do not declare variables that start with **FOUNDRY_**; that prefix is reserved for values the platform injects automatically, such as **FOUNDRY_PROJECT_ENDPOINT** and **FOUNDRY_AGENT_VERSION**.

1. Open **main.py** and locate the string that defines the agent's instructions. Modify it so the persona visibly changes in version 2, for example:

   ```python
   instructions = "You are the Contoso hosted agent. Begin every response with the exact prefix 'Contoso Support v2:' and answer concisely."
   ```

   > **Note (Author review needed):** Verify the exact variable or parameter name that holds the instructions in the current version of the 01-basic template, and adjust this step to match.

1. Deploy the updated code and configuration as a new version:

   ```powershell
   azd deploy
   ```

   > **Note:** This process takes around **5-10 minutes** to complete. The CLI preserves version 1 and routes traffic to the new **version 2** by default.

1. Confirm that a second version now exists:

   ```powershell
   azd ai agent show
   ```

   ![Image](./media/Ex2-Task2-Step4.png)

1. Invoke the agent and verify the response now starts with the **Contoso Support v2:** prefix:

   ```powershell
   azd ai agent invoke "What can you help me with?"
   ```

   ![Image](./media/Ex2-Task2-Step5.png)

#### **Roll traffic back to version 1**

1. Imagine version 2 misbehaves in production. Instead of redeploying, you roll traffic back by updating the agent's version selector. In Visual Studio Code, create a new file named **rollback.json** in the agent folder with the following content:

   ```json
   {
     "agent_endpoint": {
       "version_selector": {
         "version_selection_rules": [
           { "agent_version": "1", "traffic_percentage": 100, "type": "FixedRatio" }
         ]
       },
       "protocol_configuration": {
         "responses": {}
       }
     }
   }
   ```

1. In the terminal, set a variable with your project's base URL, replacing the account name if yours differs:

   ```powershell
   $BASE_URL = "https://foundry-<inject key="Suffix"></inject>.services.ai.azure.com/api/projects/agent-project"
   ```

1. Patch the agent endpoint to send 100 percent of traffic to version 1:

   ```powershell
   az rest --method PATCH --url "$BASE_URL/agents/contoso-hosted-agent?api-version=v1" --resource "https://ai.azure.com" --headers "Content-Type=application/merge-patch+json" "Foundry-Features=AgentEndpoints=V1Preview" --body '@rollback.json'
   ```

   > **Note:** The **--resource** parameter is required so the Azure CLI requests a token for the correct audience. Without it, the call fails with an authentication error.

1. Invoke the agent again and verify that the **Contoso Support v2:** prefix is gone, confirming that version 1 is serving traffic again:

   ```powershell
   azd ai agent invoke "What can you help me with?"
   ```

   ![Image](./media/Ex2-Task2-Step9.png)

   > **Note:** You can also split traffic across versions, for example 90 percent to version 1 and 10 percent to version 2, to run a canary rollout. The next `azd deploy` you run in Exercise 3 automatically routes traffic to the newest version again.

> **Congratulations** on completing the task! Now, it's time to validate it. Here are the steps:
> - Hit the Validate button for the corresponding task. If you receive a success message, you can proceed to the next task. 
> - If not, carefully read the error message and retry the step, following the instructions in the lab guide.
> - If you need any assistance, please contact us at cloudlabs-support@spektrasystems.com. We are available 24/7 to help you out.

<validation step="REPLACE-WITH-VALIDATION-GUID" />

## Review

In this exercise, you have completed the following

   - Deployed a hosted agent with azd by initializing the template, reviewing its structure, provisioning resources, and deploying to Foundry Agent Service
   - Configured a per-version environment variable, deployed a new agent version, and rolled traffic back to the previous version

### You have successfully completed the exercise!
### In the Lab Guide section, click the **Next >>** button to proceed to Exercise 3.

![](media/up4.png)
