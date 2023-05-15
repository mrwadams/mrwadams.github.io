---
layout: post
title: "Autonomously Generating Security Awareness Newsletters"
date: 2023-5-15 00:00:00 -0000
categories:
tags: [gpt,llm,security,ai,agents]
---

In this post I'll be sharing a Python script I recently developed that automates the generation of security awareness newsletters. It's an interesting project that combines web scraping, natural language processing, and even a touch of web development. Let's dive right in!

## Overview of the Script

The script I've developed performs the following tasks:

1. Load the configuration file and set up the environment.
2. Initialise the toolkit with Bing search and requests tools.
3. Initialise the chat language model and agent.
4. Find a recent article URL about a cyber attack.
5. Extract the article text.
6. Generate security awareness newsletter content using the GPT-4 language model.
7. Generate an HTML newsletter using the generated content.
8. Write the HTML newsletter to a file.

Here's a diagram I've drawn that shows how all of the components interact through the various requests:

![Request flow diagram](/assets/images/ai-newsletter-1.png)

Now let's walk through the script step-by-step.

### Step 1: Load Configuration File and Set Up Environment

First, the script reads a configuration file (`config.json`) containing API keys for Bing and OpenAI. It then sets up the environment by storing these API keys as environment variables. This makes it easier to access them later in the script.

    ```python
    config = load_config('config.json')
    setup_environment(config)
    ```

### Step 2: Initialise the Toolkit

Next, the script initialises a LangChain toolkit with Bing search and requests tools. This toolkit will be used later to search for relevant articles and make requests to websites.

    ```python
    num_search_results = 1
    toolkit = initialize_toolkit(num_search_results)
    ```

### Step 3: Initialise the Chat Language Model and Agent

In this step, we initialise a chat language model (GPT-4) and a LangChain agent that will use the toolkit and the chat language model to perform various tasks.

    ```python
    chat_language_model = ChatOpenAI(
        temperature=0.7,
        model_name='gpt-4',
        openai_api_key=config['API_KEYS']['OPENAI_API_KEY']
    )

    agent = initialize_agent(
        toolkit,
        chat_language_model,
        agent="zero-shot-react-description",
        verbose=True,
        return_intermediate_steps=True
    )
    ```

### Step 4: Find a Recent Article URL

Now it's time to find a recent article about a cyber attack. We do this by using the initialised agent to search for an article published during the current month. The search query is designed to find articles from reputable sources that provide valuable information about the attack, its impact, and the attack vector.

    ```python
    month = datetime.today().strftime('%B %Y')
    search_query = """As a security awareness professional, you want to inform your readers about recent cyber attack news. Find a news article about a recent cyber attack published during {month}. The article should provide valuable information about the attack, its impact, and the attack vector. Consider reputable sources like Bleeping Computer, Dark Reading, Ars Technica, and The Register.

    Once you have found a suitable article, provide the source URL in the following format: 'The source URL is: <URL>'. Remember, the goal is to select a news story that will help users understand and avoid similar attack vectors.
    """

    url = get_recent_article_url(agent, search_query, month)
    ```

### Step 5: Extract the Article Text

With the article URL in hand, we can now extract the article's text using the `newspaper3k` library.

    ```python
    article_text = get_article_text(url)
    ```

### Step 6: Generate Security Awareness Newsletter Content

In this step, we use the GPT-4 language model to generate the security awareness newsletter content based on the extracted article text. The generated content will include a summary of the article, an explanation of the attack vector(s), tips and recommendations for users, and a call to action for readers to stay vigilant and informed about cybersecurity threats.

    ```python
    gpt4_language_model = OpenAI(
        temperature=0.7,
        model_name='gpt-4',
        openai_api_key=config['API_KEYS']['OPENAI_API_KEY']
    )

    prompt_template = """You are a security awareness professional looking to inform your colleagues about a recent cyber attack covered in the following article: {article_text}. Craft a well-written and engaging internal company newsletter content that is focussed on cyber security awareness. In preparing the newsletter content you should:

    1. Summarise the main points of the article.
    2. Explains the attack vector(s) and its potential impact.
    3. Provides tips and recommendations for users to avoid falling for similar attack vectors.
    4. Encourages readers to stay vigilant and informed about cybersecurity threats.

    Your content should be written in a clear and concise manner while maintaining a professional and informative tone. You must use British English.

    Additionally, include a link to the original article {url} at the end of your content.

    Remember, your goal is to help users understand the risks and protect themselves from similar cyber attacks.
    """

    newsletter_content = generate_newsletter_content(gpt4_language_model, prompt_template, article_text, url)
    ```
![Generated newsletter content](/assets/images/ai-newsletter-2.jpg)

### Step 7: Generate the HTML Newsletter

Now that we have the newsletter content, we can generate an HTML version of the newsletter using the GPT-4 language model. The generated HTML will be responsive and visually appealing, with a clean layout and design.

    ```python
    html_prompt_template = """
    Act as a web developer and create a responsive HTML newsletter for the following content: {newsletter_content}. The newsletter should be written in a clear and concise manner while maintaining a professional and informative tone. You must use British English.

    Style the newsletter using HTML and Bootstrap CSS. The newsletter should be responsive and render correctly on desktop and mobile devices. Use Bootstrap's grid system to layout the content. The newsletter should be visually appealing and include a header and footer.

    Some additional suggestions for the design and layout are:

    1. Choose a clean and professional color scheme with a primary and secondary color. Use these colors consistently throughout the newsletter, including headings, links, and buttons.
    2. Use a visually appealing and easy-to-read font for the body text and a complementary font for headings.
    3. Include a header with the company logo, newsletter title, and a brief introduction. Use a placeholder image for the logo.
    4. Organise the content into different sections with appropriate headings, such as "Summary", "Attack Vector", "Tips and Recommendations", and "Stay Informed".
    5. Use relevant icons or images (e.g., cyber security themed) to support the content and enhance the visual appeal.
    6. Ensure there is adequate white space and visual separation between sections to improve readability.
    7. Add a footer with important links, contact information, and a copyright notice.
    8. Make sure all links, including the link to the original article, are easily identifiable and styled consistently.

    By following these suggestions, you should be able to generate an attractive and high-quality HTML newsletter.
    """

    html_newsletter = generate_newsletter_html(gpt4_language_model, html_prompt_template, newsletter_content)
    ```

![Generated newsletter with HTML and CSS](/assets/images/ai-newsletter-3.jpg)

### Step 8: Write the HTML Newsletter to a File

Finally, the script writes the generated HTML newsletter to a file. This file can then be used to send the newsletter via email or host it on a website.

    ```python
    with open('newsletter.html', 'w') as f:
        f.write(html_newsletter)
    ```

## Wrapping Up

And that's it! We've just walked through a Python script that creates a HTML security awareness newsletter from scratch, with no intervention from the user. The pattern of passing the output from autonomous search engine queries to a Large Language Model (LLM) is a powerful one that has many potential use cases - I'll be looking for more opportunities to build on this in the coming weeks.