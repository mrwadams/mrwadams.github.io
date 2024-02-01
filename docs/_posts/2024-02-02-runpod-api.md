---
layout: post
title: "Using RunPod hosted LLMs for text generation via an API"
date: 2024-02-02 00:00:00 -0000
categories:
tags: [generativeai,runpod,llms,api]
---

## Introduction

Following on from my [previous post about RunPod](2024-02-01-runpod.md) I wanted to see if I could use the platform to host LLMs that I could then access via an API. This would allow me to programmatically submit inference requests to RunPod and receive the generated text back in a format that I could then use as part of a larger application.

## Prerequisites
Sign up for a RunPod account [here](https://runpod.ai/). You'll need to provide your name, email address, and password. Once you've signed up, you'll be able to access the RunPod dashboard.

## Running an LLM with Text Generation WebUI
1. From the `Pods` tab in the RunPod dashboard, click `New Pod`.
2. Select your required number and model of GPUs from the available list. RTX 4090s are powerful enough to run most quantised LLMs, but you may need to use a RTX 6000 or larger for some models.
3. Under your selected GPU, click `Deploy`.
4. I recommend using TheBloke's [One-Click UI & API template](https://github.com/TheBlokeAI/dockerLLM/blob/main/README_Runpod_LocalLLMsUIandAPI.md) as this comes pre-installed with the required dependencies for running LLMs. If it's not already selected, use the UI to search for the correct template and select it.
    ![Deploying a pod](/assets/images/runpod-deploy.png)
    > ❗ **Important:** If you want to expose the Text Generation WebUI API, you need to select the `One-Click UI & API` template. If you select the `One-Click UI` template, the API will not be exposed.
5. Click `Continue` and then `Deploy`. After a couple of minutes, your pod will be ready to use.
6. Once the pod is in a `Running` state, click `Connect` to open the `Connection Options`.
7. Click `Connect to HTTP Service [Port 7860]` to launch TGWUI.
8. In TGWUI, select the `Model` tab. You should now see a UI similar to the one below.
    ![The Model tab within TGWUI](/assets/images/runpod-models.png)
9. Within the `Download model or LoRA` section, paste in the username/model path from HuggingFace. For example, if you wanted to run the [dolphin-2_6-phi-2](https://huggingface.co/cognitivecomputations/dolphin-2_6-phi-2) model, you would paste in `cognitivecomputations/dolphin-2_6-phi-2`.
10. Click `Download` to download the model.
11. Once the model has downloaded, refresh the list of available models in the dropdown menu in the top-left of the page. 12. Select the model you want to run and click `Load`.
13. At this point you may see some error messages displayed. These are mostly caused by missing / incorrect configuration settings. See the model details page on HuggingFace or the [Common Issues](#common-issues) section below for more information on how to resolve these.
14. With the model loaded, it's time to look at how to submit inference requests to the model via the API.

## Submitting inference requests to a RunPod hosted API endpoint
Use the following code as a starting point and tailor it to your requirements. The comments in the code indicate which elements can / should be updated.

```python
import requests
import json

url = "https://<your-runpod-pod-id>-5000.proxy.runpod.net/v1/completions" # Update with your API endpoint
headers = {
    "Content-Type": "application/json"
}

message = "How can I make a cake?" # Update with your message

data = {
    # This uses the ChatML prompt format, you may need to update it based on the requirements of the model you're using
    "prompt": """<|im_start|>system
    You are Dolphin, a helpful AI assistant.<|im_end|>
    <|im_start|>user
    {}<|im_end|>
    <|im_start|>assistant""".format(message),
    "max_tokens": 200, # Update max tokens and hyperparameters as required
    "temperature": 0.7,
    "top_p": 0.9,
    "seed": 10
}

response = requests.post(url, headers=headers, data=json.dumps(data))
print(response.text)
```

## References
* [Text Generation WebUI API Documentation](https://github.com/oobabooga/text-generation-webui/wiki/12-%E2%80%90-OpenAI-APIQ)
* [A helpful post from TheBloke on HuggingFace](https://huggingface.co/TheBloke/guanaco-65B-GPTQ/discussions/18#6486ea58c1e620f0915e5b20)
