## Exercise 4: Voice Live Integration

### Estimated Duration: 50 Minutes

## Overview

In this exercise, you will add a real-time voice channel to an existing agent using **Voice Live** in **Microsoft Foundry**. Voice Live combines speech recognition, generative AI, and text-to-speech into a single unified interface, so you do not need to orchestrate separate services. You will connect Voice Live to the agent you built in Exercise 1 by its agent name, run a speech-to-speech quickstart client with your microphone, and then tune the voice experience with semantic turn detection, barge-in handling, and noise suppression.

## Objectives

In this exercise, you will complete the following tasks:

   - Task 1: Enable Voice Live
   - Task 2: Voice Experience Tuning

### Task 1: Enable Voice Live

In this task, you will connect Voice Live to your existing agent and run a speech-to-speech quickstart so you can talk to the agent and hear it respond.

The key idea in this task: **Voice Live does not require you to redeploy or modify the agent**. It is a real-time connection layered on top of an existing Foundry agent, identified by its **agent name** and **project name**. The agent's instructions and configuration stay exactly where they are, in Foundry, and the same agent keeps serving the text channel in the playground.

> **Note:** This task requires a working microphone and speakers in the lab environment. If you are working on a lab virtual machine, confirm that audio redirection is enabled in your remote session before you begin.

1. In the Azure portal, navigate to the **foundry-<inject key="Suffix"></inject>** resource, select **Keys and Endpoint (1)** under Resource Management, and copy the **Endpoint** value **(2)** into your notepad file. This is the endpoint the Voice Live client connects to.

   ![Image](./media/Ex4-Task1-Step1.png)

   > **Note (Author review needed):** Voice Live is supported in a limited set of regions. Confirm that the lab's Foundry resource region supports Voice Live agent connections; if it does not, provision a second Foundry resource in a supported region and use its endpoint here. The agent itself does not need to move.

1. In the Visual Studio Code terminal, create a separate folder for the voice client and set up a Python virtual environment:

   ```powershell
   mkdir C:\LabFiles\voice-live-client
   cd C:\LabFiles\voice-live-client
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```

1. Install the required packages:

   ```powershell
   pip install "azure-ai-voicelive>=1.2.0b4" azure-identity python-dotenv pyaudio
   ```

   > **Note:** This process takes around **2-3 minutes** to complete.

1. In Visual Studio Code, create a new file named **.env** in the **voice-live-client** folder with the following content, replacing the endpoint placeholder with the value you copied in step 1:

   ```
   VOICELIVE_ENDPOINT=<your-foundry-resource-endpoint>
   AGENT_NAME=support-agent
   PROJECT_NAME=agent-project
   VOICE_NAME=en-US-Ava:DragonHDLatestNeural
   ```

   > **Note:** **AGENT_NAME** and **PROJECT_NAME** identify the agent you created in Exercise 1. You can optionally pin a specific version with an **AGENT_VERSION** entry; when omitted, Voice Live uses the latest version.

1. Create a new file named **voice_agent.py** in the same folder with the following content:

   ```python
   import asyncio
   import base64
   import os
   import queue
   import threading

   import pyaudio
   from azure.ai.voicelive.aio import connect
   from azure.ai.voicelive.models import (
       InputAudioFormat,
       Modality,
       OutputAudioFormat,
       RequestSession,
       ServerEventType,
   )
   from azure.identity.aio import AzureCliCredential
   from dotenv import load_dotenv

   load_dotenv()

   RATE = 24000
   CHUNK = 1024


   class AudioIO:
       """Captures microphone audio and plays agent audio using PyAudio."""

       def __init__(self, connection, loop):
           self.connection = connection
           self.loop = loop
           self.audio = pyaudio.PyAudio()
           self.playback_queue = queue.Queue()
           self.running = True

       def start(self):
           threading.Thread(target=self._capture, daemon=True).start()
           threading.Thread(target=self._playback, daemon=True).start()

       def _capture(self):
           stream = self.audio.open(format=pyaudio.paInt16, channels=1,
                                    rate=RATE, input=True,
                                    frames_per_buffer=CHUNK)
           while self.running:
               data = stream.read(CHUNK, exception_on_overflow=False)
               audio_b64 = base64.b64encode(data).decode("ascii")
               asyncio.run_coroutine_threadsafe(
                   self.connection.input_audio_buffer.append(audio=audio_b64),
                   self.loop,
               )

       def _playback(self):
           stream = self.audio.open(format=pyaudio.paInt16, channels=1,
                                    rate=RATE, output=True,
                                    frames_per_buffer=CHUNK)
           while self.running:
               try:
                   chunk = self.playback_queue.get(timeout=0.1)
                   stream.write(chunk)
               except queue.Empty:
                   continue

       def play(self, audio_bytes):
           self.playback_queue.put(audio_bytes)

       def interrupt(self):
           while not self.playback_queue.empty():
               self.playback_queue.get_nowait()


   async def main():
       agent_config = {
           "agent_name": os.environ["AGENT_NAME"],
           "project_name": os.environ["PROJECT_NAME"],
       }

       credential = AzureCliCredential()
       async with connect(
           endpoint=os.environ["VOICELIVE_ENDPOINT"],
           credential=credential,
           agent_config=agent_config,
       ) as connection:
           session_config = RequestSession(
               modalities=[Modality.TEXT, Modality.AUDIO],
               input_audio_format=InputAudioFormat.PCM16,
               output_audio_format=OutputAudioFormat.PCM16,
           )
           await connection.session.update(session=session_config)

           audio_io = AudioIO(connection, asyncio.get_running_loop())

           print("=" * 60)
           print("VOICE ASSISTANT READY - start talking (Ctrl+C to exit)")
           print("=" * 60)

           async for event in connection:
               if event.type == ServerEventType.SESSION_UPDATED:
                   audio_io.start()
               elif event.type == ServerEventType.INPUT_AUDIO_BUFFER_SPEECH_STARTED:
                   print("Listening...")
                   audio_io.interrupt()
               elif event.type == ServerEventType.INPUT_AUDIO_BUFFER_SPEECH_STOPPED:
                   print("Processing...")
               elif event.type == ServerEventType.RESPONSE_AUDIO_DELTA:
                   audio_io.play(event.delta)
               elif event.type == ServerEventType.RESPONSE_AUDIO_TRANSCRIPT_DONE:
                   print(f"Agent: {event.transcript}")
               elif event.type == ServerEventType.ERROR:
                   print(f"Error: {event.error.message}")


   if __name__ == "__main__":
       try:
           asyncio.run(main())
       except KeyboardInterrupt:
           print("\nVoice assistant stopped.")
   ```

   > **Note (Author review needed):** Verify this client against the current **azure-ai-voicelive** SDK sample for agent connections, since preview SDK event and property names can change between releases.

1. Make sure you are signed in with the Azure CLI, since agent connections require **Microsoft Entra ID** authentication (API keys are not supported in agent mode):

   ```powershell
   az login
   ```

1. Run the voice client:

   ```powershell
   python voice_agent.py
   ```

1. Wait for the **VOICE ASSISTANT READY** banner, then ask a question out loud, for example: "What is your return policy for damaged items?" Verify that:

   - The console prints **Listening...** while you speak and **Processing...** when you stop
   - You hear the agent's spoken answer through your speakers
   - The response transcript appears in the console

   ![Image](./media/Ex4-Task1-Step8.png)

1. Ask the same question you used in the Exercise 1 playground and compare the answers. This confirms the key takeaway: **one agent, two channels**. The same instructions and configuration serve both text and voice, with no redeployment.

1. Press **Ctrl+C** to stop the client.

   > **Note:** If the connection fails with an authentication error, re-run `az login` and confirm your account has the **Foundry User** role on the Foundry project. If you see a region or feature-not-found error, revisit the region note at the start of this task.

> **Congratulations** on completing the task! Now, it's time to validate it. Here are the steps:
> - Hit the Validate button for the corresponding task. If you receive a success message, you can proceed to the next task. 
> - If not, carefully read the error message and retry the step, following the instructions in the lab guide.
> - If you need any assistance, please contact us at cloudlabs-support@spektrasystems.com. We are available 24/7 to help you out.

<validation step="REPLACE-WITH-VALIDATION-GUID" />

### Task 2: Voice Experience Tuning

In this task, you will tune how the voice session detects turns, handles interruptions, and suppresses background noise, then test each behavior.

Three settings shape how natural a voice agent feels:

* **Turn detection**: Decides when you have finished speaking and the agent should respond. Basic **server VAD** uses audio energy (silence) to end a turn. **Azure semantic VAD** also considers the meaning of what was said, so a thinking pause like "my order number is... umm..." does not cut you off mid-sentence.
* **Barge-in (interruption)**: Lets the user talk over the agent. When speech is detected during playback, the service raises a speech-started event; the client stops playback immediately and the agent listens to the new input instead of finishing its sentence.
* **Noise suppression and echo cancellation**: **Azure deep noise suppression** removes background noise from the microphone signal, and **server echo cancellation** prevents the agent from hearing its own voice through your speakers and interrupting itself.

1. In Visual Studio Code, open **voice_agent.py** and add the following imports below the existing **azure.ai.voicelive.models** import block:

   ```python
   from azure.ai.voicelive.models import (
       AudioEchoCancellation,
       AudioNoiseReduction,
       AzureSemanticVad,
   )
   ```

1. Locate the **session_config** definition inside **main()** and replace it with the following configuration, which enables semantic turn detection, deep noise suppression, and server echo cancellation:

   ```python
   session_config = RequestSession(
       modalities=[Modality.TEXT, Modality.AUDIO],
       input_audio_format=InputAudioFormat.PCM16,
       output_audio_format=OutputAudioFormat.PCM16,
       turn_detection=AzureSemanticVad(),
       input_audio_noise_reduction=AudioNoiseReduction(
           type="azure_deep_noise_suppression"
       ),
       input_audio_echo_cancellation=AudioEchoCancellation(),
   )
   ```

   ![Image](./media/Ex4-Task2-Step2.png)

   > **Note (Author review needed):** Verify the exact model class names for semantic VAD, noise reduction, and echo cancellation in the installed **azure-ai-voicelive** version, and adjust if they differ.

1. Run the client again:

   ```powershell
   python voice_agent.py
   ```

1. Test the improved turn detection. Ask a question that includes a natural thinking pause, for example: "I want to ask about... hmm... your shipping options to Canada." Verify the agent waits for you to finish instead of responding at the first pause.

   ![Image](./media/Ex4-Task2-Step4.png)

1. Test barge-in. Ask a question that produces a long answer, for example: "Explain all of your shipping options in detail." While the agent is still speaking, interrupt it out loud with: "Actually, just tell me the fastest one." Verify that:

   - Playback stops as soon as you start speaking
   - The console prints **Listening...**
   - The agent answers your new question instead of finishing the old response

1. Test noise suppression. Play some background noise near your microphone (for example, music from your phone at low volume) and ask a normal question. Verify that the agent still transcribes and answers your question correctly.

   > **Note:** With deep noise suppression enabled, steady background noise is filtered before speech recognition runs. Loud speech-like noise, such as a nearby conversation, can still trigger turn detection; production deployments tune VAD thresholds for their acoustic environment.

1. Press **Ctrl+C** to stop the client when you finish testing.

> **Congratulations** on completing the task! Now, it's time to validate it. Here are the steps:
> - Hit the Validate button for the corresponding task. If you receive a success message, you can proceed to the next task. 
> - If not, carefully read the error message and retry the step, following the instructions in the lab guide.
> - If you need any assistance, please contact us at cloudlabs-support@spektrasystems.com. We are available 24/7 to help you out.

<validation step="REPLACE-WITH-VALIDATION-GUID" />

## Review

In this exercise, you have completed the following

   - Enabled Voice Live by connecting it to your existing agent and running the speech-to-speech quickstart
   - Tuned the voice experience with semantic turn detection, barge-in, and noise suppression, and tested each behavior

### You have successfully completed the exercise!
### In the Lab Guide section, click the **Next >>** button to proceed to Exercise 5.

![](media/up4.png)
