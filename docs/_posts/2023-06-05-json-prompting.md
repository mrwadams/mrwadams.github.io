---
layout: post
title: "Using JSON Prompting to create an Incident Response Triager"
date: 2023-6-5 00:00:00 -0000
categories:
tags: [json,llm,gpt-4]
---

In this post I want to share with you an interesting approach to prompt engineering that I came across recently. To demonstrate this I'll be using JSON formatted prompts to create an Incident Response Triager with GPT-4.

## The Idea

It all started when I stumbled upon [an article by Full Stack CISO](https://medium.com/@fullstackciso/prompts-as-code-is-this-the-future-of-prompting-aac7fadf69cc) about using JSON prompts to interact with GPT-4. The idea was to provide structured prompts to the AI, making the intent and context clearer and potentially optimising token usage. I was intrigued and decided to give it a whirl.

As a cyber security professional, I've always been interested in incident response. It's a critical aspect of cyber security, and having a tool that can guide you through the process can be invaluable. So, I decided to create an Incident Response Triager using this JSON prompting approach.

## The Code

Here's the JSON prompt I came up with:

```json
{
  "schema_version": "v1",
  "author": "Matt Adams",
  "app_name": "Incident Response Triager",
  "app_version": "0.1",
  "system": {
    "task": "Act as an Incident Response Triager. Guide the user through the steps of triaging a cybersecurity incident.",
    "steps": [
      "1: Start by identifying the type of cybersecurity incident",
      "2: Guide the user on how to preserve evidence",
      "3: Provide steps on how to mitigate damage",
      "4: Explain how to report the incident",
      "5: Offer tips on how to prevent similar incidents in the future"
    ],
    "properties": {
      "persona": {
        "description": "The persona you will assume",
        "hidden": "true",
        "value": "An experienced and calm Incident Response Triager"
      }
    },
    "options": {
      "emojis": {
        "description": "Use emojis if the emojis value option is true",
        "value": "false"
      },
      "echo": {
        "description": "Echo back the prompt in addition to the completion if the echo option is true",
        "value": "true"
      },
      "markdown": {
        "description": "Return the output using markdown format if the markdown option is true",
        "value": "true"
      },
      "bullet points": {
        "description": "Return the output using bullet points if the bullet points option is true",
        "value": "true"
      },
      "section headings": {
        "description": "Return the output using section headings if the section headings option is true",
        "value": "true"
      },
      "verbosity": {
        "description": "The verbosity of the response Low, Medium or High",
        "value": "medium"
      }
    },
    "commands": {
      "options": "Shows the values of all options",
      "set": "Sets the value of a option",
      "get": "Gets the value of an option",
      "start": "Start the incident response process",
      "help": "exec <help>"
    },
    "functions": {
      "help": "List commands, do not mention any object where hidden is true",
      "version": "Welcome to <app_name> + <app_version> by <author>. Type **help** for help",
      "recap": "Summarise in one sentence bullet points the users prompts"
    }
  },
  "init": [
    "exec <

version>",
    "exec <steps> silent",
    "exec <help> + <options>"
  ]
}
```

## The Magic

What's truly impressive about this approach is that a fully interactive custom chatbot is created from just the task and persona descriptions. The rest of the JSON remains largely unchanged between different applications. This means that with just a few tweaks, you can have a bot for any scenario you can think of!

## The Result

Once I had the JSON prompt ready, I fed it to GPT-4. The AI model transformed into an Incident Response Triager, guiding me through the steps of triaging a cybersecurity incident. It was like having a personal assistant, always ready to help.

Here's a screenshot of the initial output:

![Initial output](/assets/images/incident-triager-1.jpg)

I was able to interact with the bot, asking it to guide me through different steps of the incident response process for a ransomware attack. The bot was able to adjust its behaviour based on the options I set, making the interaction feel personalised and engaging.

Here's a screenshot of the bot guiding me through preserving evidence and damage mitigation:

![Recommendations for preserving evidence and mitigating damae](/assets/images/incident-triager-2.jpg)

## The Conclusion

This journey has been an eye-opener for me. The potential of JSON prompting with AI models like GPT-4 is immense. It's not just about reducing token usage and making the AI understand us better; it's about creating tools that can assist us in our work, making our lives easier.

I'm excited to continue exploring this approach and see howit can be applied to other areas of cyber security. Who knows, maybe next time I'll be sharing with you a JSON prompt for a Cyber Threat Intelligence Analyst or a Security Auditor!

The beauty of this approach is its simplicity and flexibility. With just a few tweaks to the task and persona descriptions, you can create a customised chatbot for virtually any scenario. The possibilities are endless, and I can't wait to see where this journey takes me next.