---
layout: post
title: "Using RunPod hosted LLMs for text generation via an API"
date: 2024-02-03 00:00:00 -0000
categories:
tags: [generativeai,runpod,llms,api,autogen]
---

## Introduction

Continuing my recent thread of [RunPod](2024-02-01-runpod.md) related [posts](2024-02-02-runpod-api.md) - after I had worked out how to access a RunPod hosted LLM via an API, I immediated started to think about how I could use the programmatic access. One idea I had was to set up [AutoGen Studio](https://microsoft.github.io/autogen/blog/2023/12/01/AutoGenStudio/) to use a hosted LLM running on RunPod. This would enable me to try running AutoGen agents using larger models than I can fit on my local machine. This post documents what I did to get a large Mixture of Experts model running on RunPod, and then how I configured AutoGen Studio to access it.

## Prerequisites
As before, you'll need to sign up for a RunPod account [here](https://runpod.ai/) and add a minimum of $25 to your account.

## Choosing a Model
I've experimented with AutoGen before and I know that it really benefits from using a highly capable model such as GPT-4. While no open-source models have yet reached the capabilities of GPT-4, the recent reviews of [Dolphin 2.7 Mixtral 8X7B](https://huggingface.co/cognitivecomputations/dolphin-2_7-mixtral-8x7b) caught my attention, and I thought that it might be worth trying as an alternative. The unqantised model is huge though, so I decided to try [TheBloke's quantised version](https://huggingface.co/TheBloke/dolphin-2.7-mixtral-8x7b-GPTQ) as it's possible to run that on a single RTX A6000.

## Running the LLM with Text Generation WebUI
1. From the `Pods` tab in the RunPod dashboard, click `New Pod`.
2. Select an RTX A6000 from the available list of GPUs.
3. Click `Deploy`.
4. I recommend using the [RunPod TheBloke LLMs](https://github.com/TheBlokeAI/dockerLLM/blob/main/README_Runpod_LocalLLMsUI.md) template, as this comes pre-installed with the required dependencies for running LLMs. If it's not already selected, use the UI to search for the correct template and select it.
5. Click `Continue` and then `Deploy`. After a couple of minutes, your pod will be ready to use.
6. Once the pod is in a `Running` state, click `Connect` to open the `Connection Options`.
7. Click `Connect to HTTP Service [Port 7860]` to launch TGWUI.
8. In TGWUI, select the `Model` tab. You should now see a UI similar to the one below.
    ![The Model tab within TGWUI](/assets/images/runpod-models.png)
9. Within the `Download model or LoRA` section, paste in the username/model path from HuggingFace. In this case we want the [Dolphin 2.7 Mixtral 8X7B - GPTQ](https://huggingface.co/TheBloke/dolphin-2.7-mixtral-8x7b-GPTQ) model.
10. Click `Download` to download the model.
11. Once the model has downloaded, refresh the list of available models in the dropdown menu in the top-left of the page. 12. Select the model you want to run and click `Load`.
13. At this point you may see some error messages displayed. These are mostly caused by missing / incorrect configuration settings. See the model details page on HuggingFace or the [Common Issues](#common-issues) section below for more information on how to resolve these.
14. With the model loaded, we now need to install AutoGen Studio and configure it to use the RunPod hosted LLM.

## Installing AutoGen Studio
Installing AutoGen Studio is very straightforward using `pip`, although I'd recommend using a virtual environment to keep your system clean. The following commands will install AutoGen Studio and its dependencies:

```bash
pip install autogenstudio
```

Once installed, start the AutoGen Studio web UI using the following command:

```bash
autogenstudio ui --port 8081
```

You can now access AutoGen Studio by navigating to `http://localhost:8081` in your web browser.

## Configuring AutoGen Studio to use a RunPod hosted LLM

I won't go into the details of how to use AutoGen Studio here, but I will show you how to configure it to use a RunPod hosted LLM. The following steps assume that you have already created an AutoGen agent, or that you're using one of the example agents that comes with AutoGen Studio.

1. Within the AutoGen Studio UI, make sure you're in the `Build` tab and then click on `Models` in the left-hand menu.
2. Click the `New Model` button in the top-right of the page. This will open a new model configuration page similar to the one below.
    ![Model Specification](/assets/images/autogenstudio.png)
3. Complete the following fields:
    * **Model Name**: A name for your model. This can be anything you like.
    * **API Key**: You need to put something in here, but it's not used for RunPod hosted LLMs.
    * **Base URL**: This is the URL of your RunPod pod. It will be something like `https://<your-runpod-pod-id>-5000.proxy.runpod.net/v1`. The /v1 is important as it tells AutoGen Studio which version of the API to use.
    * **API Type**: Set this to `open_ai` as Text Generation WebUI's API is compatible with OpenAI's API specification.
    * **API Version**: Skip this field.
    * **Description**: A description of your model. This can be anything you like.

4. Click `Test Model` to check that AutoGen Studio can connect to your RunPod hosted LLM. If everything is configured correctly, you should see a success message.
5. Click `Save` to save your model configuration.

You can now configure your AutoGen agents to use the RunPod hosted LLM for inference by simply adding the model to each agent's specification.

## References
* [How To Run Dolphin Mixtral 8x7b In The Cloud](https://youtu.be/zGnMzysH_oI?si=S8uKvJXT55S7E7Lz)
