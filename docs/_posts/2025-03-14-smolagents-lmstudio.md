---
layout: post
title: "Building AI Agents with Local LLMs: Using smolagents with LM Studio"
date: 2025-03-14 00:00:00 -0000
categories:
tags: [genai,smolagents,agents,development,llms]
---

## Introduction

The world of AI agents has been dominated by cloud-based models like GPT-4o and Claude Sonnet, but there's a growing movement toward running these capabilities locally. In this post, I'll show you how to create powerful AI agents using the `smolagents` library with local Large Language Models (LLMs) running on LM Studio. This approach offers several advantages: enhanced privacy, reduced costs, lower latency, and the ability to work offline.

I've created a [GitHub repository](https://github.com/mrwadams/smolagents-lmstudio-examples) with complete examples that you can use as a starting point for your own projects.

## What is smolagents?

[smolagents](https://github.com/smol-ai/smol-agent) is a lightweight Python library for creating AI agents that can use tools to accomplish tasks. It's designed to be simple, flexible, and compatible with various LLM backends, including local models. The library provides several agent types:

- **CodeAgent**: The default agent that writes and executes Python code to solve tasks
- **ToolCallingAgent**: A general-purpose agent that uses tools via function calling (JSON-like blobs)
- **MultiStepAgent**: A base class for agents that solve tasks in multiple steps

For most use cases with local LLMs, either the `CodeAgent` or `ToolCallingAgent` is recommended, depending on whether you want the agent to execute Python code.

## What is LM Studio?

[LM Studio](https://lmstudio.ai/) is a desktop application that allows you to download, run, and chat with local LLMs on your computer. It provides a user-friendly interface for managing models and offers an OpenAI-compatible API server, making it easy to integrate with libraries like `smolagents`.

## Prerequisites

Before we begin, you'll need:

- Python 3.8+
- smolagents with OpenAI support (`pip install 'smolagents[openai]'`)
- requests (`pip install requests`)
- LM Studio installed on your computer

## Setting Up LM Studio

1. Download and install LM Studio from [lmstudio.ai](https://lmstudio.ai/)
2. Launch LM Studio and download a model of your choice. For the best experience with function calling (which is essential for tools), I recommend:
   - Gemma 3 27B Instruct (gemma-3-27b-it) - On par with Gemini 1.5 Pro
   - Gemma 3 12B Instruct (gemma-3-12b-it) - Better than older Gemma 2 27B models
3. Load your chosen model in LM Studio
4. Click on the "Local Server" tab
5. Click "Start Server"
6. The server should now be running at `http://localhost:1234`

## Testing the Connection

Before diving into agent creation, let's verify that we can connect to LM Studio's API. I've created a simple test script that you can run:

```python
import requests
import json

def test_lm_studio_connection():
    """
    Test if the connection to LM Studio's API is working properly.
    """
    print("Testing connection to LM Studio API...")
    
    # LM Studio API endpoint
    api_url = "http://localhost:1234/v1/chat/completions"
    
    # Simple test message
    payload = {
        "model": "local-model",
        "messages": [
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "Hello, are you working?"}
        ],
        "temperature": 0.7,
        "max_tokens": 50
    }
    
    try:
        # Send a request to the API
        response = requests.post(
            api_url,
            headers={"Content-Type": "application/json"},
            data=json.dumps(payload),
            timeout=10
        )
        
        # Check if the request was successful
        if response.status_code == 200:
            result = response.json()
            assistant_message = result["choices"][0]["message"]["content"]
            print("\n✅ Connection successful!")
            print("\nLM Studio API response:")
            print(f"Assistant: {assistant_message}")
            return True
        else:
            print(f"\n❌ Error: Received status code {response.status_code}")
            print(f"Response: {response.text}")
            return False
            
    except requests.exceptions.ConnectionError:
        print("\n❌ Connection Error: Could not connect to LM Studio API.")
        print("Make sure LM Studio is running and the server is started.")
        print("Check that the server is running on http://localhost:1234")
        return False
        
    except Exception as e:
        print(f"\n❌ Error: {str(e)}")
        return False

if __name__ == "__main__":
    test_lm_studio_connection()
```

Save this as `test_lm_studio_connection.py` and run it. If everything is set up correctly, you should see a success message and a response from the model.

## Creating a Basic Agent

Now that we've confirmed our connection to LM Studio, let's create a simple agent using `smolagents`. Here's a basic example:

```python
import os
from smolagents import OpenAIServerModel, ToolCallingAgent

# Configure the model to use LM Studio's local API endpoint
model = OpenAIServerModel(
    model_id="local-model",  # This can be any name, LM Studio will use whatever model you have loaded
    api_base="http://localhost:1234/v1",  # Default LM Studio API endpoint
    api_key="not-needed",  # LM Studio doesn't require an API key by default
)

# Create a simple agent using the local model
agent = ToolCallingAgent(
    name="LocalLLMAgent",
    model=model,
    tools=[],  # Empty list of tools
    # You can also add the default toolbox with add_base_tools=True
)

# Example conversation with the agent
response = agent.run("Hello! Can you tell me what you are and how you're running?")
print(f"Agent response: {response}")
```

Save this as `local_llm_example.py` and run it. You should see a response from your local LLM.

## Adding Tools to Your Agent

The real power of `smolagents` comes from its ability to use tools. Let's create a coding assistant agent with a tool for searching Python documentation:

```python
import os
from smolagents import OpenAIServerModel, CodeAgent, tool

# Configure the model to use LM Studio's local API endpoint
model = OpenAIServerModel(
    model_id="local-model",
    api_base="http://localhost:1234/v1",
    api_key="not-needed",
)

# Define a tool that the agent can use to search for Python documentation
@tool
def search_python_docs(query: str) -> str:
    """
    Search for Python documentation based on the query.
    This is a simplified example - in a real application, you might
    use a web API or local documentation.
    
    Args:
        query: The search query
        
    Returns:
        str: Information about the Python feature
    """
    # This is a mock implementation
    python_docs = {
        "list": "Lists are used to store multiple items in a single variable. Lists are ordered, changeable, and allow duplicate values.",
        "dict": "Dictionaries are used to store data values in key:value pairs. A dictionary is a collection which is ordered, changeable and do not allow duplicates.",
        "set": "Sets are used to store multiple items in a single variable. Set is one of 4 built-in data types in Python used to store collections of data.",
        "function": "A function is a block of code which only runs when it is called. You can pass data, known as parameters, into a function.",
    }
    
    for key, value in python_docs.items():
        if key in query.lower():
            return value
    
    return "No specific information found for that query. Try asking about lists, dictionaries, sets, or functions."

# Example usage of the agent
def chat_with_coding_assistant():
    print("Python Coding Assistant (powered by your local LLM)")
    print("Type 'exit' to end the conversation\n")
    
    # Store conversation history for context (since agents don't maintain state between runs)
    conversation_history = []
    
    while True:
        user_input = input("You: ")
        if user_input.lower() == 'exit':
            print("Goodbye!")
            break
        
        # Add user input to conversation history
        conversation_history.append(f"User: {user_input}")
        
        # Create a new agent for each interaction
        coding_assistant = CodeAgent(
            name="PythonCodingAssistant",
            model=model,
            tools=[search_python_docs],
        )
        
        # Include conversation history in the prompt
        full_prompt = "\n".join(conversation_history) + "\n\nPlease respond to the latest message."
        
        # Use reset=False to continue the conversation without resetting the agent's memory
        response = coding_assistant.run(full_prompt, reset=False)
        print(f"Assistant: {response}")
        
        # Add assistant response to conversation history
        conversation_history.append(f"Assistant: {response}")
        
        # You can inspect the agent's logs to see what happened
        # print(coding_assistant.logs)

if __name__ == "__main__":
    print("Reminder: Ensure LM Studio is running with a model loaded on http://localhost:1234\n")
    chat_with_coding_assistant()
```

Save this as `coding_assistant_agent.py` and run it. You can now chat with your coding assistant and ask questions about Python.

## Using the Default Toolbox

smolagents comes with a default toolbox that you can add to your agent:

```python
from smolagents import OpenAIServerModel, CodeAgent

model = OpenAIServerModel(
    model_id="local-model",
    api_base="http://localhost:1234/v1",
    api_key="not-needed",
)

# Create an agent with the default toolbox
agent = CodeAgent(
    name="AssistantWithDefaultTools",
    model=model,
    tools=[],  # Your custom tools would go here
    add_base_tools=True  # This adds DuckDuckGo search, Python interpreter, and Transcriber
)

response = agent.run("What is the current weather in London?")
print(f"Agent response: {response}")
```

The default toolbox includes:
- DuckDuckGo web search
- Python code interpreter (for ToolCallingAgent)
- Transcriber (speech-to-text using Whisper-Turbo)

## Key Concepts to Understand

When working with `smolagents` and local LLMs, there are a few important concepts to keep in mind:

### 1. Tool Creation

There are two ways to create tools:

#### Using the @tool decorator:

```python
@tool
def my_tool(param: str) -> str:
    """
    Tool description.
    
    Args:
        param: Parameter description
        
    Returns:
        str: Return value description
    """
    # Tool implementation
    return result
```

#### Creating a Tool subclass:

This gives you more flexibility, especially for initializing heavy class attributes.

### 2. Conversation State

To create a conversational interface, you can:
- Store conversation history manually
- Use `reset=False` when calling `agent.run()` to continue the conversation
- Create a new agent for each interaction if needed

### 3. Inspecting Agent Runs

After running an agent, you can inspect what happened:
- `agent.logs` stores fine-grained logs of the agent's run
- `agent.write_memory_to_messages()` writes the agent's memory as a list of chat messages

### 4. Multi-Agent Systems

You can create hierarchical multi-agent systems where one agent manages others:

```python
from smolagents import CodeAgent, OpenAIServerModel, DuckDuckGoSearchTool

model = OpenAIServerModel(
    model_id="local-model",
    api_base="http://localhost:1234/v1",
    api_key="not-needed",
)

web_agent = CodeAgent(
    tools=[DuckDuckGoSearchTool()],
    model=model,
    name="web_search",
    description="Runs web searches for you. Give it your query as an argument."
)

manager_agent = CodeAgent(
    tools=[], 
    model=model, 
    managed_agents=[web_agent]
)

manager_agent.run("Who is the CEO of Hugging Face?")
```

### 5. Interactive UI

You can use GradioUI to interact with your agent:

```python
from smolagents import CodeAgent, OpenAIServerModel, GradioUI

model = OpenAIServerModel(
    model_id="local-model",
    api_base="http://localhost:1234/v1",
    api_key="not-needed",
)

agent = CodeAgent(tools=[], model=model, add_base_tools=True)

GradioUI(agent).launch()
```

### 6. Model Performance

The quality of responses depends on the model you have loaded in LM Studio. For the best experience:
- Use larger models (12B+ parameters) for better results
- Choose models that support function calling (like Gemma 3)
- Be aware that some models may not support all features

## Common Issues and Troubleshooting

- **Connection Error**: Make sure LM Studio server is running and the port matches your configuration
- **Poor Responses**: Try loading a different/larger model in LM Studio
- **Tool Execution Issues**: Some local models may not support function calling properly. Try a different model or simplify your prompts.
- **Import Errors**: If you get an error about missing 'openai', install the OpenAI dependency: `pip install 'smolagents[openai]'`

## Sharing Your Agent

Once you've configured your agent to your needs, you can share it to the Hugging Face Hub:

```python
agent.push_to_hub("your-username/your-agent-name")
```

And to load an agent from the Hub:

```python
agent.from_hub("your-username/your-agent-name", trust_remote_code=True)
```

## Conclusion

Running AI agents with local LLMs offers a compelling alternative to cloud-based solutions. With `smolagents` and LM Studio, you can create powerful agents that respect your privacy, work offline, and don't incur ongoing API costs.

The examples I've shared are just the beginning. You can extend these agents with custom tools, create multi-agent systems, add interactive UIs, and even share your agents with others. The full code for these examples, along with additional resources, is available in my [GitHub repository](https://github.com/mrwadams/smolagents-lmstudio-examples).

As local LLMs continue to improve, the gap between cloud and local capabilities will narrow, making this approach increasingly viable for a wide range of applications. I encourage you to experiment with different models and tools to find the combination that works best for your specific needs.

Happy building!