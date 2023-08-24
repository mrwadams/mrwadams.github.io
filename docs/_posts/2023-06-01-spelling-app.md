---
layout: post
title: "Creating a Spelling Practice App for my children"
date: 2023-6-1 00:00:00 -0000
categories:
tags: [development,streamlit,python]
---

In today's post I'm going to share a side project that I've been working on for the last couple of days - a spelling practice app for children.

You may already know that I'm not a software developer by profession, but I've always had a keen interest in creating applications that can solve real-world problems. This project was born out of a desire to make spelling practice more engaging and interactive for my own children and their classmates.

The Spelling Practice App is a web-based application that helps children improve their spelling skills in a fun and interactive way. The app provides a list of words which the children can listen to, spell out, and then check if they've got it right. The app also keeps a score, providing a sense of progress and achievement.

Here's a peek at some of the key sections of the code:

[Link](http://s3.amazonaws.com/prtcthisisatestbucket)

```python
# Function to convert text to audio using Google Text to Speech
def text_to_audio(word):
    tts = gTTS(word, lang='en')
    audio_data = BytesIO()
    tts.write_to_fp(audio_data)
    audio_data.seek(0)
    return audio_data
```

This function uses Google's Text-to-Speech service to convert the selected word into audio, which is then played back to the learner. This auditory input is not only engaging but also helps children understand the correct pronunciation of words.

![A screenshot of the app](/assets/images/spelling-app.jpg)

The heart of the app is the Practise Spellings page, where children interact with the word list, listen to the words, and attempt to spell them correctly. Here's a snippet of the code that handles this:

```python
# Page to practice spellings
elif page == "Practise Spellings":
    st.header("Practise Spellings 📝")
    if "word_list" in st.session_state:
        words = read_word_list(f"{st.session_state.word_list}.txt")
        ...
        # Check answer button logic
        if check_pressed:
            if user_input.strip() == '':
                st.info("Please enter a spelling before checking the answer.")
            else:
                st.session_state.total_attempts += 1
                if user_input.strip().lower() == st.session_state.selected_word.strip().lower():
                    st.success("Correct!")
                    st.session_state.score += 1
                else:
                    st.error(f"Oops! You typed '{user_input}'. The correct spelling is '{st.session_state.selected_word}'")
```

This section of the code controls how the words are chosen from the list, how the audio is generated, and how the user's input is checked for correctness. It also keeps track of the score, providing immediate feedback and a sense of progress.

The entire code for the Spelling Practice App is available on GitHub for anyone interested in understanding the workings of the app or wishing to contribute to it. You can access the repository [here](https://github.com/mrwadams/spelling-practice-app).

So, if you're a parent, teacher, or simply someone interested in educational tools, I invite you to try out the Spelling Practice App. Your feedback will be invaluable in refining the app and making it even better for our young learners.

I'm already working on adding new features, like a function to create a list of words to review based on any incorrect answers, so stay tuned for more updates!

Happy spelling!