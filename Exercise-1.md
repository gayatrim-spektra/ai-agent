## Exercise 1: Foundry Hosted Agents Setup

### Estimated Duration: 40 Minutes

## Overview

In this exercise, you will explore the core concepts of **Foundry Agent Service** and prepare the environment you will use throughout this lab. You will sign in to the **Microsoft Foundry** portal, create a Foundry project, and deploy a chat model. You will then build your first agent directly in the portal and validate its behavior in the agent playground before moving to code-first deployment in later exercises.

## Objectives

In this exercise, you will complete the following tasks:

   - Task 1: Environment Preparation
   - Task 2: Build Your First Agent

### Task 1: Environment Preparation

In this task, you will review the key concepts behind Foundry Agent Service, then create a Foundry project and deploy a model that every later exercise builds on.

#### Understand Foundry Agent Service concepts

Before you create any resources, review the building blocks you will work with in this lab:

* **Microsoft Foundry**: A unified platform for building, deploying, and governing AI applications and agents. You access it through the Foundry portal at `https://ai.azure.com`.
* **Foundry resource (account)**: The top-level Azure resource that hosts model deployments and projects. Billing, networking, and content safety policies are managed at this level.
* **Foundry project**: A workspace inside a Foundry resource where you create agents, connections, and evaluations. Each project has its own endpoint and its own **managed identity**.
* **Agent**: A combination of a model, instructions, and optional tools that can reason over user requests and take actions. Foundry supports two main agent types:
    - **Prompt agents**: Defined declaratively (model + instructions + tools) and run entirely on Foundry-managed infrastructure. You will build one of these in this exercise.
    - **Hosted agents**: Your own containerized agent code (for example, built with the **Microsoft Agent Framework**) that Foundry runs for you in per-session, sandboxed compute with its own **Microsoft Entra ID** identity, scale-to-zero, and built-in observability. You will deploy one of these in Exercise 2.
* **Model deployment**: A specific model (for example, **gpt-5-mini**) made available under a deployment name that agents and applications call.

The resources you create in this lab fit together like this:

```
Foundry resource (account)
├── Model deployment (gpt-5-mini)
└── Foundry project
    ├── Prompt agent  (created in the portal, Exercise 1)
    ├── Hosted agent  (deployed with azd, Exercise 2)
    │   └── Versions 1, 2, ... (each version gets sandboxed compute)
    ├── Connections (Container Registry, Application Insights)
    └── Observability (traces, evaluations)
```

Some concrete examples of what agents built this way can do:

* A customer support agent that answers product questions from grounded documentation.
* An IT helpdesk agent that files tickets through a tool call.
* A voice concierge that answers spoken questions in real time (you will build this channel in Exercise 4).
* A data assistant that runs calculations with a code interpreter tool.

#### **Create a Foundry project and deploy a model**

1. Open a browser in the lab environment and navigate to the following URL:

   ```
   https://ai.azure.com
   ```

1. Sign in with the following credentials:

   - **Email/Username:** <inject key="AzureAdUserEmail"></inject>

   - **Password:** <inject key="AzureAdUserPassword"></inject>

   ![Image](./media/Ex1-Task1-Step2.png)

1. If a **Stay signed in?** pop-up appears, click on **No**.

1. On the Microsoft Foundry home page, click on **+ Create new** and select **Foundry project**.

   ![Image](./media/Ex1-Task1-Step4.png)

   > **Note (Author review needed):** Verify the exact entry point in the current Foundry portal. The portal may instead show a **Create project** button on first sign-in, or a welcome wizard that creates the first project automatically.

1. On the **Create a project** pane, expand **Advanced options** and provide the following values, then click on **Create (5)**:

   - **Project name:** **agent-project (1)**

   - **Foundry resource:** **foundry-<inject key="Suffix"></inject> (2)**

   - **Subscription:** Leave the default subscription selected **(3)**

   - **Resource group / Region:** Select the existing resource group and choose **East US 2 (4)**

   ![Image](./media/Ex1-Task1-Step5.png)

   > **Note:** Project creation takes around **1-2 minutes** to complete. Once it finishes, you land on the project home page.

1. From the project home page, copy the **Foundry project endpoint** value and paste it into a notepad file. You will need this value in later exercises.

   ![Image](./media/Ex1-Task1-Step6.png)

1. In the left navigation pane, select **Build (1)**, then select **Deployments (2)**.

   ![Image](./media/Ex1-Task1-Step7.png)

1. Click on **+ Deploy model (1)** and select **Deploy base model (2)**.

   ![Image](./media/Ex1-Task1-Step8.png)

1. In the model list, search for **gpt-5-mini (1)**, select it from the results **(2)**, and click on **Confirm (3)**.

   ![Image](./media/Ex1-Task1-Step9.png)

1. On the deployment pane, keep the default **Deployment name** as **gpt-5-mini** and the default **Deployment type**, then click on **Deploy**.

   ![Image](./media/Ex1-Task1-Step10.png)

   > **Note:** If **gpt-5-mini** is not available in your region, select another chat-capable model such as **gpt-4.1-mini** and use its deployment name wherever this lab guide refers to **gpt-5-mini**.

1. Once the deployment completes, verify that the model appears in the **Deployments** list with the state **Succeeded**.

   ![Image](./media/Ex1-Task1-Step11.png)

> **Congratulations** on completing the task! Now, it's time to validate it. Here are the steps:
> - Hit the Validate button for the corresponding task. If you receive a success message, you can proceed to the next task. 
> - If not, carefully read the error message and retry the step, following the instructions in the lab guide.
> - If you need any assistance, please contact us at cloudlabs-support@spektrasystems.com. We are available 24/7 to help you out.

<validation step="REPLACE-WITH-VALIDATION-GUID" />

### Task 2: Build Your First Agent

In this task, you will create a prompt agent in the Foundry portal and test its behavior interactively in the agent playground.

1. In the left navigation pane of your project, select **Build (1)**, then select **Agents (2)**.

   ![Image](./media/Ex1-Task2-Step1.png)

1. Click on **+ New agent**.

   ![Image](./media/Ex1-Task2-Step2.png)

1. On the agent creation pane, provide the following values:

   - **Agent name:** **support-agent (1)**

   - **Model:** Select the **gpt-5-mini** deployment you created in Task 1 **(2)**

   ![Image](./media/Ex1-Task2-Step3.png)

1. In the **Instructions** field, enter the following system prompt and save the agent:

   ```
   You are the Contoso customer support assistant. You help customers with questions about Contoso products, orders, returns, and shipping policies. Answer politely and concisely. If a question is unrelated to Contoso customer support, politely decline and redirect the user to a support topic.
   ```

   ![Image](./media/Ex1-Task2-Step4.png)

1. Note down the **Agent name** exactly as shown in the agent details. You will use this value to connect **Voice Live** to this agent in Exercise 4.

1. Open the agent **Playground** tab. In the chat pane, enter the following prompt and press **Enter**:

   ```
   What is your return policy for damaged items?
   ```

   ![Image](./media/Ex1-Task2-Step6.png)

   > **Note:** The agent has no grounding data yet, so it answers from general knowledge in the persona you defined. You will add governance, voice, and evaluation layers on top of agents in later exercises.

1. Verify that the agent responds in the Contoso support persona, then test the instruction boundary with an off-topic prompt:

   ```
   Write me a poem about the ocean.
   ```

1. Confirm that the agent politely declines and redirects the conversation back to support topics.

   ![Image](./media/Ex1-Task2-Step8.png)

1. Click on **New chat** to reset the conversation, and try one more support question of your choice to confirm the behavior is consistent across sessions.

> **Congratulations** on completing the task! Now, it's time to validate it. Here are the steps:
> - Hit the Validate button for the corresponding task. If you receive a success message, you can proceed to the next task. 
> - If not, carefully read the error message and retry the step, following the instructions in the lab guide.
> - If you need any assistance, please contact us at cloudlabs-support@spektrasystems.com. We are available 24/7 to help you out.

<validation step="REPLACE-WITH-VALIDATION-GUID" />

## Review

In this exercise, you have completed the following

   - Reviewed Foundry Agent Service concepts, created a Foundry project, and deployed the gpt-5-mini model
   - Built your first agent in the Foundry portal and tested it in the agent playground

### You have successfully completed the exercise!
### In the Lab Guide section, click the **Next >>** button to proceed to Exercise 2.

![](media/up4.png)
