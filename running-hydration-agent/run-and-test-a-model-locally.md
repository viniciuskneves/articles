# Run and test a model locally

Before I even start talking about agents and all the "it is just a loop", we should look into the basics first.

Instead of connecting to Anthropic/OpenAI/..., running a less powerful model locally helps us to understand a few concepts that are important on the long run.

## Install dependencies

We need to install + run the model locally before proceeding.

### Installing Ollama

We will be using Ollama to run the model locally.
I recommend following their install guide on https://ollama.com/download or `brew install ollama` if you're a Mac user with Homebrew.

After that you can start Ollama through `ollama serve`.

Now we need to pull the model we want to use. You can find a long list at https://ollama.com/search.
I picked `qwen3.5:4b`: https://ollama.com/library/qwen3.5
It is not too small and not too big, enough for us to play with.

You can test it through (assuming the port `11434` is correct): 
```bash
curl -s http://localhost:11434/v1/messages -H "content-type: application/json" -d '{
  "model":"qwen3.5:4b","max_tokens":1000,"thinking":{"type":"disabled"},
  "messages":[{"role":"user","content":"Tell me your name."}]}'
```

This should return something like:
```json
{"id":"msg_bed9c38990c8d65af63c7508","type":"message","role":"assistant","model":"qwen3.5:4b","content":[{"type":"text","text":"My name is **Qwen3.5**. I am a large-scale language model developed by Tongyi Lab. How can I assist you today?"}],"stop_reason":"end_turn","usage":{"input_tokens":17,"cache_read_input_tokens":0,"output_tokens":31}}
```

We can already see things that are interesting in the request above: `max_tokens` and `thinking`.
We will explore them later, let's first make sure we write some code that uses our model.

## Prompting the model through code
