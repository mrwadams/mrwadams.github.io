---
layout: post
title: "Basic template for Selenium scripts"
date: 2022-07-08 00:00:00 -0000
categories:
tags: [python,selenium]
---

## Background
I've recently been working through [Automate the Boring Stuff with Python](https://automatetheboringstuff.com/) and found the chapters on using Python for web scraping / automatically interacting with web pages particularly interesting.

One of the modules mentioned in the book (`selenium`) allows Python to directly control a web browser so that it can programatically find and scrape specific elements in the page, click on links, populate form fields, etc. This has the potential to be a very useful capability, but when testing the example scripts from the book I ran into some issues.

It seems that the code in the book is a little outdated, which caused IDLE to return several errors and warnings when I tried to run the example scripts.

## Code
The following is a basic template that can be used as a basis for creating new Selenium scripts. It was tested on a Windows 10 machine.

```python
#! python3

from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from webdriver_manager.chrome import ChromeDriverManager
from selenium.webdriver.common.keys import Keys # required if you want to insert key presses to complete forms
from selenium.webdriver.common.by import By # required to search for elements in a page

import os # required to use environment variables

# Install the latest version of WebDriver (if not cached)
driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()))

# Open browser and navigate to a given web page
driver.get("https://www.example.com")

# The following code will find two form fields (called "email" and "password" and populate them with the values of two environment variables stored by the OS)
# Refer to https://selenium-python.readthedocs.io/ for examples other possible interactions (e.g. clicking links)
email = driver.find_element(By.NAME,"email")
email.send_keys(os.environ.get('EMAIL'))

password = driver.find_element(By.NAME,"password")
password.send_keys(os.environ.get('PASSWORD'))
password.send_keys(Keys.RETURN)

# Close browser window
driver.quit()

```