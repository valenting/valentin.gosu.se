---
layout: post
title: Local voice transcription in Obsidian with Whisper.cpp
---
For a while now I've been a fan of Obsidian.  It's really great. I use it for all kinds of note taking and brainstorming. But long form writing isn't exactly my strong point. Especially when I'm concentrating on a specific piece of code, it's often difficult to start typing my thoughts into a document without losing my train of thought.

This is where Whisper comes in. [Whisper](https://github.com/openai/whisper) is an automatic speech recognition model that was open-sourced by OpenAI. It's actually several models of different sizes that allow you to convert speech to text with surprising accuracy. Of course OpenAI provides this as a web service, and of course someone has already created an [Obsidian plugin](https://github.com/nikdanilov/whisper-obsidian-plugin) that uses it for transcription.

But since the models are open source there's very little preventing you from running it on your own computer.
In my case, I'm using [whisper.cpp](https://github.com/ggerganov/whisper.cpp) - a C++ port of the model that performs much better in my albeit limited experience. Luckily this project also provides a simple server implementation that is API compatible, and the plugin allows us to specify the endpoint, it should've just worked.

However, the Obsidian plugin insists you provide an OpenAI key, which causes requests to generate a [CORS preflight](https://developer.mozilla.org/en-US/docs/Glossary/Preflight_request) - which is an OPTIONS request that checks it's allowed to POST to that endpoint with an "Authorization" header. In this case the fix was [really simple](https://github.com/valenting/whisper.cpp/commit/b942bfdbadc89c0b1a6629ae37a709d877da3660) so I submitted a [PR](https://github.com/ggerganov/whisper.cpp/pull/1850/)

So, if you want to deploy this setup for yourself, here's a quick guide:

```bash
git clone https://github.com/ggerganov/whisper.cpp.git
cd whisper.cpp
make server
bash ./models/download-ggml-model.sh small
# Make sure to install ffmpeg for conversion
# Use --host 0.0.0.0 if you host this on a separate machine on your network
./server -debug --convert -l auto --host 0.0.0.0 -m models/ggml-small.bin -t 8 --port 8080
```

In my case I'm using the small model, but feel free to try out [different ones](https://github.com/ggerganov/whisper.cpp/blob/master/models/README.md) depending on the speed of your computer.

So that's it. Now just make sure to put `http://localhost:8080/inference` in the settings for the Whisper plugin, and a random API key, and you should be ready to go. If you are running the server on a different machine on your network, use `http://hostOrIP:8080/inference` instead.

The hotkey is `Alt-Q`. Press it once to start recording, and again to get the transcription.
