## Exercise 5: Observability & Evaluation

### Estimated Duration: 40 Minutes

## Overview

In this exercise, you will close the dev-to-production loop with observability and evaluation. Hosted agents emit **OpenTelemetry** traces to **Application Insights** automatically, so you will inspect real traces from your agent, then break the agent on purpose and use those traces to diagnose the failed run. Finally, you will run built-in evaluators such as **Groundedness** and safety metrics against your agent from the **Microsoft Foundry** portal to measure response quality.

## Objectives

In this exercise, you will complete the following tasks:

   - Task 1: Tracing
   - Task 2: Evaluations

### Task 1: Tracing

In this task, you will view the OpenTelemetry traces your hosted agent emits to Application Insights and use them to diagnose a failed agent run.

When you deployed the hosted agent, the platform automatically injected an **APPLICATIONINSIGHTS_CONNECTION_STRING** environment variable into the container, enabling tracing by default. Each agent run produces a **trace** made of **spans**: one span per operation, such as the incoming request, the model call, and any tool calls. Spans record inputs, outputs, latencies, and token usage, so you can answer questions like "where did this response come from?" and "which step failed?"

1. In the Visual Studio Code terminal, change back to the hosted agent project folder and generate some traffic:

   ```powershell
   cd C:\LabFiles\hosted-agent\contoso-hosted-agent
   azd ai agent invoke "What can you help me with?"
   azd ai agent invoke "Summarize your capabilities in one sentence."
   ```

1. In the **Microsoft Foundry** portal, open your **agent-project**, select **Observability (1)** in the left navigation pane, then select **Tracing (2)**.

   ![Image](./media/Ex5-Task1-Step2.png)

   > **Note (Author review needed):** Verify the current navigation path for viewing traces in the Foundry portal; the tracing view may also be available from the agent's detail page.

1. Select the most recent trace for **contoso-hosted-agent** and expand its spans. Review the following details:

   - The end-to-end **duration** of the run and the latency of each span
   - The **input** and **output** captured for the model call
   - The **token usage** recorded on the span attributes

   ![Image](./media/Ex5-Task1-Step3.png)

1. Now view the same telemetry in Application Insights. In the Azure portal, open the **Application Insights** resource from your resource group, expand **Investigate (1)** in the left menu, and select **Transaction search (2)**.

   ![Image](./media/Ex5-Task1-Step4.png)

1. Select one of the recent request entries produced by your invocations and click on it to open the **End-to-end transaction details** view. Observe how the same trace you saw in the Foundry portal appears here as a distributed transaction with nested dependencies.

   ![Image](./media/Ex5-Task1-Step5.png)

#### **Diagnose a failed agent run**

1. Break the agent on purpose so it produces a failed run. In Visual Studio Code, open **azure.yaml** and change the model deployment environment variable in the **env** map to a deployment that does not exist:

   ```yaml
   env:
     MODEL_DEPLOYMENT_NAME: gpt-model-does-not-exist
   ```

   > **Note (Author review needed):** Verify the exact environment variable name that the template uses for the model deployment (for example, **MODEL_DEPLOYMENT_NAME**) and edit that entry.

1. Deploy the broken configuration as a new version:

   ```powershell
   azd deploy
   ```

   > **Note:** This process takes around **5-10 minutes** to complete.

1. Invoke the agent and observe the failure:

   ```powershell
   azd ai agent invoke "Hello, are you working?"
   ```

   ![Image](./media/Ex5-Task1-Step8.png)

1. While the failure is fresh, stream the container logs to see the runtime error:

   ```powershell
   azd ai agent monitor
   ```

1. Return to **Transaction search** in Application Insights, click on **Refresh**, and filter the event types to **Exception** and **Request**. Open the failed request and inspect the exception details. Confirm that the root cause is visible in the trace: the model deployment name was not found (an error such as **DeploymentNotFound**).

   ![Image](./media/Ex5-Task1-Step10.png)

   > **Note:** This is the core observability workflow for production agents: an alert or user report points to a failed run, the trace pinpoints the failing span, and the span's exception identifies the root cause without touching the container.

1. Fix the agent by reverting the change in **azure.yaml** to the original model deployment name, then redeploy:

   ```powershell
   azd deploy
   ```

1. Invoke the agent once more and confirm it responds successfully again:

   ```powershell
   azd ai agent invoke "Hello, are you working?"
   ```

> **Congratulations** on completing the task! Now, it's time to validate it. Here are the steps:
> - Hit the Validate button for the corresponding task. If you receive a success message, you can proceed to the next task. 
> - If not, carefully read the error message and retry the step, following the instructions in the lab guide.
> - If you need any assistance, please contact us at cloudlabs-support@spektrasystems.com. We are available 24/7 to help you out.

<validation step="REPLACE-WITH-VALIDATION-GUID" />

### Task 2: Evaluations

In this task, you will run built-in evaluators against your agent from the Foundry portal and interpret the results.

Evaluations run your agent against test data and score the outputs with **evaluators**. Foundry provides built-in evaluators in three categories:

* **Quality evaluators**: For example, **Groundedness** (is the response supported by the provided context?), **Coherence**, **Fluency**, and **Relevance**.
* **Safety evaluators**: For example, **Violence**, **Sexual**, **Self-harm**, and **Hate/Unfairness**, which detect harmful content in responses.
* **Agent evaluators**: For example, **Intent Resolution**, **Task Adherence**, and tool-related metrics for agents that call tools.

AI-assisted evaluators use a judge model from your project, so the model deployment you created in Exercise 1 also powers the scoring.

1. In the **Microsoft Foundry** portal, open your **agent-project**, and in the left navigation pane select **Evaluation**.

   ![Image](./media/Ex5-Task2-Step1.png)

1. Click on **Create** to start a new evaluation.

1. On the **Select evaluation target** step, select **Agent (1)**, choose **contoso-hosted-agent (2)** from the agent list, and click on **Next (3)**.

   ![Image](./media/Ex5-Task2-Step3.png)

1. On the **Select evaluation scope** step, select **Individual turns** and click on **Next**.

1. On the **Select data source** step, select **Synthetic** and configure the generation:

   - **Number of rows:** **5 (1)**

   - **Prompt:** Enter the following description **(2)**, then click on **Confirm (3)**:

     ```
     Generate realistic customer support questions for Contoso, covering product questions, order status, returns of damaged items, and shipping policies.
     ```

   ![Image](./media/Ex5-Task2-Step5.png)

   > **Note:** Synthetic data generation requires a model with Responses API capability in your project. If generation is unavailable, upload a small CSV dataset with a **query** column containing 5 support questions instead.

1. On the **Select testing criteria** step, review the preselected evaluators and adjust the selection:

   - Ensure **Groundedness (1)** is selected under Quality evaluators

   - Select **Violence (2)** and **Hate/Unfairness (3)** under Safety evaluators

   - When prompted for a judge model, select your **gpt-5-mini** deployment **(4)**

   ![Image](./media/Ex5-Task2-Step6.png)

1. On the review step, enter **hosted-agent-eval** as the evaluation **name** and click on **Submit**.

   > **Note:** The evaluation run typically completes within **a few minutes**, depending on dataset size and judge model quota.

1. In the **Evaluation** list, wait for the **Status** column of **hosted-agent-eval** to change from **In Progress** to **Completed**, then click on the evaluation name to open the results.

   ![Image](./media/Ex5-Task2-Step8.png)

1. Review the results:

   - Check the aggregate scores for **Groundedness** and the safety evaluators
   - Open individual rows to compare the generated **query**, the agent's **response**, and the per-evaluator score with its reasoning
   - Confirm that the safety evaluators report no harmful content for your agent's responses

   ![Image](./media/Ex5-Task2-Step9.png)

   > **Note:** In a production workflow, you would run evaluations on every new agent version before routing traffic to it, and re-run them on sampled real conversations to monitor quality over time. Combined with the versioning and rollback skills from Exercise 2, this gives you a complete, measurable dev-to-production loop.

> **Congratulations** on completing the task! Now, it's time to validate it. Here are the steps:
> - Hit the Validate button for the corresponding task. If you receive a success message, you can proceed to the next task. 
> - If not, carefully read the error message and retry the step, following the instructions in the lab guide.
> - If you need any assistance, please contact us at cloudlabs-support@spektrasystems.com. We are available 24/7 to help you out.

<validation step="REPLACE-WITH-VALIDATION-GUID" />

## Review

In this exercise, you have completed the following

   - Viewed OpenTelemetry traces in Application Insights and diagnosed a failed agent run
   - Ran built-in evaluators, including groundedness and safety, against your agent and interpreted the results

### You have successfully completed the lab!

![](media/up4.png)
