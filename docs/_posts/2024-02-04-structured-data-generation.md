---
layout: post
title: "Improving LLM Output Reliability with Structured Data Generation"
date: 2025-02-12 00:00:00 -0000
categories:
tags: [genai,llms,patterns,json,development]
---

## Introduction

When working with Large Language Models (LLMs), getting them to generate output in specific technical formats consistently and reliably can be challenging. While many models (including locally hosted ones running on Ollama) handle structured JSON output well through their native JSON modes, generating other technical formats like Mermaid diagram syntax or Gherkin test cases often leads to errors and inconsistencies. This post explores a pattern I've successfully used to improve the reliability of complex format generation by leveraging the models' strength in JSON output to separate content generation from format handling.

## The Problem

I encountered this challenge while working on generating Mermaid diagrams with LLMs. Even though the models could understand the concept and content of what needed to be generated, they would often:
* Produce invalid Mermaid syntax
* Create inconsistent node and edge definitions
* Mix different Mermaid diagram types incorrectly
* Generate overly complicated or mangled diagram structures

These issues weren't unique to Mermaid - they're common when asking LLMs to generate any output that requires strict syntax adherence. While models' built-in JSON modes work well for structured data, they don't solve the problem of generating other technical formats reliably.

## The Solution: A Two-Step Approach

Instead of asking the model to generate Mermaid (or other format-specific) syntax directly, I found success by breaking the problem into two distinct steps:

1. **Content Generation**: Have the model generate a structured JSON representation that focuses purely on the content and logic
2. **Format Conversion**: Use code to convert this structured data into the desired final format

Here's a simple example of how this pattern works in practice:

```python
def get_formatted_output(client, prompt, target_format="mermaid"):
    """
    Generate formatted output using a two-step approach:
    1. Get structured JSON from the model
    2. Convert JSON to desired format
    """
    # Step 1: Create a clear prompt for JSON structure
    system_prompt = f"""Generate a structured JSON representation of the content.
    The JSON should follow this format:
    {{
        "elements": [
            {{
                "id": "unique_id",
                "type": "node",
                "label": "Description",
                "relationships": [
                    {{
                        "target": "other_id",
                        "type": "connection_type"
                    }}
                ]
            }}
        ]
    }}
    
    Focus on the content and relationships. Don't worry about the final format."""

    # Get JSON from the model
    response = client.chat.completions.create(
        response_format={"type": "json_object"},
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": prompt}
        ]
    )

    try:
        # Parse the JSON response
        content = json.loads(response.choices[0].message.content)
        
        # Step 2: Convert to desired format
        if target_format == "mermaid":
            return convert_to_mermaid(content)
        elif target_format == "graphviz":
            return convert_to_graphviz(content)
        elif target_format == "gherkin":
            return convert_to_gherkin(content)
        else:
            raise ValueError(f"Unsupported format: {target_format}")
            
    except json.JSONDecodeError:
        # Implement appropriate error handling
        return handle_parsing_error(response.choices[0].message.content)

def convert_to_mermaid(content):
    """Convert JSON structure to Mermaid diagram syntax."""
    mermaid_lines = ["graph TD"]
    
    for element in content["elements"]:
        # Add node definition
        node_id = element["id"]
        label = element["label"]
        mermaid_lines.append(f'    {node_id}["{label}"]')
        
        # Add relationships
        for rel in element.get("relationships", []):
            mermaid_lines.append(f'    {node_id} --> {rel["target"]}')
    
    return "\n".join(mermaid_lines)

def convert_to_gherkin(content):
    """Convert JSON structure to Gherkin test case format."""
    # Similar conversion logic for Gherkin
    pass

def convert_to_graphviz(content):
    """Convert JSON structure to Graphviz DOT syntax."""
    # Similar conversion logic for Graphviz
    pass
```

## Benefits

This approach offers several significant advantages:
1. **Separation of Concerns**: The model focuses solely on content and logic, while formatting rules are handled consistently in code
2. **Improved Reliability**: JSON is a well-understood format that models are good at generating
3. **Better Error Handling**: JSON parsing provides clear error messages and makes it easier to implement fallback mechanisms
4. **Maintainability**: Formatting rules are centralized in code, making them easier to update and maintain
5. **Consistency**: The same JSON structure can be converted to multiple output formats if needed

## Implementation Tips

To get the most out of this pattern, consider the following tips:

1. **Design Clear JSON Structures**
   * Keep the structure as simple as possible while capturing all necessary information
   * Document the expected structure with examples in your prompts
   * Use meaningful field names that map clearly to your domain

2. **Use Built-in JSON Support**
   * Take advantage of OpenAI's `response_format={"type": "json_object"}` parameter
   * For other models, use the equivalent feature or include explicit instructions about JSON formatting
   * Always implement proper JSON validation

3. **Implement Robust Error Handling**
   * Add validation of the JSON structure before conversion
   * Include fallback mechanisms for when JSON parsing fails
   * Log parsing and conversion errors for debugging

4. **Keep Conversion Logic Clean**
   * Write clear, well-documented conversion functions
   * Use type hints to make the code more maintainable
   * Consider implementing unit tests for the conversion logic

## Real-World Application

I've successfully applied this pattern in my STRIDE GPT project for generating attack trees:

1. The model generates a JSON structure representing attack paths and their relationships
2. A conversion function transforms this into proper Mermaid syntax
3. The result is rendered as a clean, consistent attack tree diagram

This approach has significantly improved the reliability of the generated diagrams and made the code more maintainable.

## Example JSON Structure

Here's a simplified example of how we might structure attack tree data:

```json
{
    "nodes": [
        {
            "id": "root",
            "label": "Main Attack Vector",
            "children": [
                {
                    "id": "child1",
                    "label": "Sub-attack Vector"
                }
            ]
        }
    ]
}
```

This structured data can then be converted to Mermaid syntax or any other required format using dedicated conversion functions.

## Future Considerations

This pattern can be extended to various other use cases where structured output is needed:
* Generating test cases in Gherkin format
* Creating API specifications
* Producing documentation with specific structures
* Generating configuration files
* Creating various types of diagrams

## References
* [OpenAI JSON Mode Documentation](https://platform.openai.com/docs/guides/text-generation/json-mode)
* [JSON Schema Specification](https://json-schema.org/)
* [Mermaid JS Documentation](https://mermaid.js.org/) 