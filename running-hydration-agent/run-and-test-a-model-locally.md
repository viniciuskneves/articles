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

We will be using Anthropic SDK in our examples given it works well with local models and don't much of an abstraction from what the `curl` above was doing.

```typescript
import Anthropic from "@anthropic-ai/sdk";

const BASE_URL="http://localhost:11434" // This is the same port exposed from `ollama serve`
const API_KEY = "ollama"; // This you can later use your external API key, instead of using a local model
const client = new Anthropic({
  baseURL: BASE_URL,
  apiKey: API_KEY,
});

// Here is the equivalent to our `curl`
const MODEL = "qwen3.5:4b";
const response = await client.messages.create({
  model: MODEL,
  max_tokens: 1000,
  thinking: { type: "disabled" },
  messages: [
    { role: "user", content: "Tell me your name." },
  ],
});

console.log("Stop reason:", response.stop_reason);
console.log("\n");
for (const block of response.content) {
  console.log("Content block:", block);
}
console.log("\n");
console.log("Token usage:", response.usage);
```

Running this script through Node/Deno/Bun you should get back something like:
```bash
Stop reason: end_turn


Content block: {
  type: 'text',
  text: "My name is **Qwen3.5**. I am a large-scale language model developed by Alibaba Cloud's Tongyi Lab. How can I assist you today?"
}


Token usage: { input_tokens: 17, cache_read_input_tokens: 0, output_tokens: 34 }
```

## max_tokens

As the name suggests, this limits the maximum amount of tokens the model **generates**. It doesn't matter the input tokens, it matters the output tokens.

The impact here is that if this number is low, the model might not generate your desired output.
Using 18 as `max_tokens` in the previous example, we get something like:
```bash
Stop reason: max_tokens # Different stop reason


Content block: {
  type: 'text',
  text: 'My name is **Qwen3.5**. I am a large language model developed by' # This is truncated compared to the previous answer
}


Token usage: { input_tokens: 17, cache_read_input_tokens: 0, output_tokens: 18 }
```

If the number is too big, it might consume a lot of tokens ($).

It is all about finding the right size through testing and validation.
Given we're running a local model and are not paying for the tokens, we can keep a large number in order to keep iterating and testing.

## cache_*_tokens

As the name suggests, those are tokens coming from cache, which also means $.
They also impact speed: the time it takes for you to get the response back from the LLM.

From [Anthropic's pricing docs](https://platform.claude.com/docs/en/about-claude/pricing):
> Cache write tokens are charged when content is first stored. Cache read tokens are charged when a subsequent request retrieves the cached content. A cache hit costs 10% of the standard input price...

## Thinking... 🤔

Let's allow thinking and check what changes in our example. If we just comment the thinking configuration and run again:
```bash
Stop reason: end_turn


Content block: {
  type: 'thinking',
  thinking: 'The user is asking for my name. I need to identify myself as Qwen3.5, a large language model developed by Alibaba Cloud.\n' +
    '\n' +
    'I should respond directly and simply state my name without adding unnecessary details or capabilities since the question only asks for my identity. I will provide a clear and concise answer.'
}
Content block: { type: 'text', text: 'My name is Qwen3.5.' }


Token usage: { input_tokens: 4, cache_read_input_tokens: 11, output_tokens: 76 }
```

On top of our response, we also get a "thinking" block back, which is how the prompt was interpreted by the LLM (what is was thinking).
Bear in mind that "thinking" is also consuming output tokens ($) so we need to think (ha!) when to use it.

Let's check some more interesting examples with/without thinking before we can reason about it.
Given the following prompt:
```
A runner does 3 laps of 2.5 km, then 4 laps of 1.75 km, then walks 600 m.
Running pace is 5:30 min/km, walking pace is 12 min/km.
How many minutes does the whole session take? Reply with only the number, no units, no explanation.
```

How would you answer it?
I would start with getting a notepad and start the math:
- Running distance = 3 * 2.5km + 4 * 1.75km = 7.5km + 7km = 14.5km
- Running minutes = 14.5km * 5:30min/km = 14.5km * 5.5min/km = 79.75min
- Walking minutes = 600m * 12min/km = 7.2min
- Total = 79.75min + 7.2min = 86.95min

How would a LLM with thinking enabled answer it?
```bash
Stop reason: end_turn


Content block: {
  type: 'thinking',
  thinking: 'Thinking Process:\n' +
    '\n' +
    '1.  **Analyze the Request:**\n' +
    "    *   Input: A runner's activity description (laps, distances, running pace, walking pace).\n" +
    '    *   Task: Calculate the total time in minutes for the whole session.\n' +
    '    *   Constraint: Reply with *only* the number, no units, no explanation.\n' +
    '\n' +
    '2.  **Break Down the Activity:**\n' +
    '    *   Part 1: Running 3 laps of 2.5 km.\n' +
    '    *   Part 2: Running 4 laps of 1.75 km.\n' +
    '    *   Part 3: Walking 600 m.\n' +
    '    *   Pace: Running = 5 minutes 30 seconds per km.\n' +
    '    *   Pace: Walking = 12 minutes per km.\n' +
    '\n' +
    '3.  **Calculate Distances:**\n' +
    '    *   Part 1 Distance: $3 \\text{ laps} \\times 2.5 \\text{ km/lap} = 7.5 \\text{ km}$.\n' +
    '    *   Part 2 Distance: $4 \\text{ laps} \\times 1.75 \\text{ km/lap}$.\n' +
    '        *   $1.75 \\times 4 = 7 \\text{ km}$.\n' +
    '    *   Part 3 Distance: $600 \\text{ m} = 0.6 \\text{ km}$.\n' +
    '\n' +
    '4.  **Calculate Time for Running:**\n' +
    '    *   Total Running Distance = Part 1 + Part 2 = $7.5 \\text{ km} + 7 \\text{ km} = 14.5 \\text{ km}$.\n' +
    '    *   Running Pace: 5 min 30 sec/km.\n' +
    '        *   5 min 30 sec = 5.5 minutes per km.\n' +
    '    *   Time for Running = Total Running Distance $\\times$ Pace.\n' +
    '        *   $14.5 \\text{ km} \\times 5.5 \\text{ min/km}$.\n' +
    '        *   Calculation:\n' +
    '            *   $14.5 \\times 5 = 72.5$\n' +
    '            *   $14.5 \\times 0.5 = 7.25$\n' +
    '            *   $72.5 + 7.25 = 79.75$ minutes.\n' +
    '\n' +
    `    *   Alternatively, convert running pace to seconds first to avoid decimal issues, then convert back? Or stick to minutes with decimals. The prompt asks for "How many minutes". Usually, this implies a decimal number or potentially minutes+seconds formatted as a single number (but instruction says "no units", implying just digits). Let's keep it as decimal minutes for now.\n` +
    '    *   $14.5 \\times 5.5$:\n' +
    '        *   $145 \\times 55 = 145 \\times (50 + 5) = 7250 + 725 = 7975$.\n' +
    '        *   Decimal places: 2 ($14.5$ has 1, $5.5$ has 1). So $79.75$.\n' +
    '\n' +
    '5.  **Calculate Time for Walking:**\n' +
    '    *   Distance = $0.6 \\text{ km}$.\n' +
    '    *   Walking Pace = 12 min/km.\n' +
    '    *   Time for Walking = $0.6 \\times 12$.\n' +
    '        *   $0.6 \\times 10 = 6$\n' +
    '        *   $0.6 \\times 2 = 1.2$\n' +
    '        *   $6 + 1.2 = 7.2$ minutes.\n' +
    '\n' +
    '6.  **Calculate Total Time:**\n' +
    '    *   Total Time = Running Time + Walking Time.\n' +
    '    *   Total Time = $79.75 + 7.2$.\n' +
    '    *   Calculation:\n' +
    '        *   $79.75 + 7.00 = 86.75$\n' +
    '        *   $86.75 + 0.20 = 86.95$ minutes.\n' +
    '\n' +
    '7.  **Review the Steps:**\n' +
    '    *   Run 1: 3 * 2.5 = 7.5 km. Pace 5.5 min/km. Time = 41.25 min? Wait, let me re-read "Running pace is 5:30 min/km". Yes, 5 minutes and 30 seconds = 5.5 minutes.\n' +
    '        *   $7.5 \\times 5.5$:\n' +
    '            *   $7.5 \\times 5 = 37.5$\n' +
    '            *   $7.5 \\times 0.5 = 3.75$\n' +
    '            *   Total = $41.25$.\n' +
    '    *   Run 2: 4 * 1.75 = 7 km. Pace 5.5 min/km. Time = 38.5 min?\n' +
    '        *   $7 \\times 5.5 = 7 \\times (5 + 0.5) = 35 + 3.5 = 38.5$.\n' +
    '    *   Run Total: $41.25 + 38.5 = 79.75$ min. (Matches previous calculation).\n' +
    '    *   Walk: 0.6 km. Pace 12 min/km. Time = 7.2 min.\n' +
    '    *   Total: $79.75 + 7.2 = 86.95$.\n' +
    '\n' +
    '8.  **Check Constraint:** "Reply with only the number, no units, no explanation."\n' +
    '    *   Result: 86.95.\n' +
    "    *   Is rounding required? The prompt doesn't specify rounding. Usually, exact decimals are preferred unless it's seconds. However, 86.95 minutes is a precise answer based on the input.\n" +
    `    *   Wait, does "5:30 min/km" mean 5 minutes and 30 seconds? Yes, that's standard notation for pace (e.g., 5:30 per km). If it meant 5.30 minutes, it would usually write "5.30".\n` +
    '    *   Does the output need to be an integer? "How many minutes does the whole session take?" Usually implies exact time or rounded. Without specific rounding instructions (like "round to the nearest minute"), I should provide the calculated value. However, 86.95 minutes is 1 hour, 27 minutes, 57 seconds.\n' +
    `    *   Let's check if there's ambiguity in "5:30 min/km". Could it mean 5 minutes per km and 30% efficiency? No. It universally means 5 minutes and 30 seconds.\n` +
    '    *   Calculation Check again:\n' +
    '        *   Run 1: $3 \\times 2.5 = 7.5$ km. Time = $7.5 \\times 5.5 = 41.25$ min.\n' +
    '        *   Run 2: $4 \\times 1.75 = 7$ km. Time = $7 \\times 5.5 = 38.5$ min.\n' +
    '        *   Run Total: $41.25 + 38.5 = 79.75$ min.\n' +
    '        *   Walk: $0.6$ km. Time = $0.6 \\times 12 = 7.2$ min.\n' +
    '        *   Total: $79.75 + 7.2 = 86.95$ min.\n' +
    '\n' +
    '    *   Is there any reason to convert to seconds first?\n' +
    '        *   Run Pace = $(300 + 30)/60 = 5.5$ min.\n' +
    '        *   Walk Pace = $12$ min.\n' +
    '    *   Everything seems consistent.\n' +
    '    *   Format: Just the number "86.95".\n' +
    '\n' +
    '    *   Wait, could "5:30" mean 5 minutes 30 seconds per km? Yes.\n' +
    "    *   Could it imply integer output? Often these riddles expect an integer. Let's see if rounding makes sense. 87 minutes?\n" +
    '    *   Given the precision of inputs (2.5, 1.75, 600), decimals are acceptable unless specified "integer". However, often users want seconds converted or rounded.\n' +
    '    *   Instruction: "How many minutes..."\n' +
    '    *   If I answer 86.95, is it better than 87?\n' +
    '    *   Usually, without explicit rounding instruction, exact calculation is safer. But sometimes "5:30" implies time in a way that might yield cleaner numbers.\n' +
    "    *   Let's check the math one more time.\n" +
    '    *   $7.5 \\times 5.5 = 7.5 \\times \\frac{11}{2} = \\frac{82.5}{2} = 41.25$. Correct.\n' +
    '    *   $7 \\times 5.5 = 38.5$. Correct.\n' +
    '    *   Sum Run = 79.75.\n' +
    '    *   Walk: $0.6 \\times 12 = 7.2$.\n' +
    '    *   Sum Total = 86.95.\n' +
    '\n' +
    '    *   Is there any hidden catch? "3 laps of 2.5 km" -> 7.5km. "4 laps of 1.75 km" -> 7km. "Walks 600 m" -> 0.6km.\n' +
    '    *   Running pace 5:30 min/km = 5.5 min/km.\n' +
    '    *   Walking pace 12 min/km.\n' +
    '    *   All inputs are standard.\n' +
    '    *   Output format: "only the number, no units, no explanation".\n' +
    '\n' +
    '    *   Decision: Provide 86.95.\n' +
    "    *   Wait, sometimes these prompts come from datasets where the expected answer is an integer (e.g. rounding to nearest whole minute). Let's check if 0.95 minutes rounds up. $0.95 \\times 60 = 57$ seconds. 86 mins 57 secs. Rounding to nearest minute is 87.\n" +
    '    *   However, "How many minutes" can accept decimals. I will provide the exact decimal value to be precise, as there is no rounding instruction. If I look at similar logic puzzles, sometimes they prefer integers if the context implies it, but math problems usually prefer exactness.\n' +
    "    *   Actually, looking at the number 86.95, it's very specific.\n" +
    `    *   Let's consider if "5:30" is interpreted as 5.3 minutes (five point three minutes)? No, standard running notation is min:sec.\n` +
    "    *   Let's consider if the output should be an integer. If I say 86.95, it is unambiguous. If I say 87, it involves a rounding step not requested.\n" +
    '    *   Okay, I will stick with 86.95.\n' +
    '\n' +
    '    Wait, let me double check the arithmetic $14.5 \\times 5.5$.\n' +
    '    $14.5 \\times 5 = 72.5$\n' +
    '    $14.5 \\times 0.5 = 7.25$\n' +
    '    Sum = 79.75. Correct.\n' +
    '\n' +
    '    $0.6 \\times 12 = 7.2$. Correct.\n' +
    '    Sum = 86.95. Correct.\n' +
    '\n' +
    '    One more check on the prompt text: "Reply with only the number, no units, no explanation."\n' +
    "    Okay. I'll output 86.95.\n" +
    '\n' +
    '    Wait, is there any possibility "5:30 min/km" means 5 minutes for every half km? No, usually min per km.\n' +
    '    Is it possible "12 min/km" walking pace is fast? Jogging is common, walking is slower. Walking 12 min/km is actually quite slow (3 mph vs 60% speed). Running 5:30/min is roughly 8km/h or 12mph.Wait.\n' +
    '    Run pace 5:30 per km = 107 m/s * 3600 / (5.5) ... No.\n' +
    '    Speed in min/km is the reciprocal of seconds per distance? No.\n' +
    '    Time for 1 km.\n' +
    '    5 mins 30 secs per km -> Average speed ~ 21.8 km/h. (Very fast for 1km, usually marathon pace).\n' +
    '    Wait, standard running pace is often in min/km or min/mile. 5:30 km-1 is roughly 5.5 min/km = 97 m/min. Speed = 60/5.5 = ~10.9 m/s? No.\n' +
    '    Distance = 1 km = 1000 m. Time = 330 sec. Speed = 1000 / 330 = 3.03 m/s = ~10.9 km/h.\n' +
    '    This is a very slow jog (approx 6-7 mph). Actually, elite runners do 3 min/km. Recreational do 5-6 min/km. So this is a realistic run pace for a casual runner? No, average human run is slower or faster... Anyway, physically plausible.\n' +
    '    Walk pace 12 min/km -> 50 min walk of 1 km. Speed = 1000/720 = 1.38 m/s ~ 5 km/h. This is realistic walking speed.\n' +
    '\n' +
    "    So the physics don't invalidate the numbers. The calculation stands.\n" +
    '    Final check: 86.95.\n' +
    '\n' +
    '    Is it possible they want seconds included as part of the number (e.g. total minutes and seconds)?\n' +
    `    "How many minutes..." usually asks for a magnitude in the unit 'minutes'.\n` +
    '    If I answer 86.95, that is 86 minutes and 57 seconds.\n' +
    '    If I answer 8695, that implies milliseconds or hundredths of a minute which might be confusing.\n' +
    '    Given "no units", just the number.\n' +
    '    I will provide 86.95.\n' +
    '\n' +
    '    Wait, could there be a rounding expectation?\n' +
    '    In many competitive math contexts on LLMs, if the result is X.XX and no instruction to round, providing the exact float is correct.\n' +
    '    However, often users prefer rounded integers for "time".\n' +
    "    Let's search for similar problems online or in training data... No search capability.\n" +
    '    I will assume exactness is preferred unless it leads to infinite decimals. 86.95 terminates.\n' +
    `    Actually, let's consider the phrase "How many minutes".\n` +
    '    If a clock says 86 minutes and 57 seconds, answering "86" or "87"?\n' +
    '    If I buy an item that costs \\$0.95, I answer the price.\n' +
    '    Time is often treated as continuous.\n' +
    "    I'll stick with 86.95.\n" +
    '\n' +
    '    Wait, one detail: "3 laps of 2.5 km".\n' +
    '    Does this mean a standard track lap is 400m and they run extra? No, "laps of [distance]" implies the distance covered per lap is defined as that distance. Like "5 miles long laps" on a road course.\n' +
    '    So $3 \\times 2.5$ is correct.\n' +
    '\n' +
    '    Final check of calculation:\n' +
    '    Run Dist = $7.5 + (4 \\times 1.75) = 7.5 + 7 = 14.5$ km.\n' +
    '    Run Time = $14.5 \\times 5.5$.\n' +
    '    $14 \\times 5.5 = 77$.\n' +
    '    $0.5 \\times 5.5 = 2.75$.\n' +
    '    $77 + 2.75 = 79.75$. Correct.\n' +
    '    Walk Dist = $0.6$ km.\n' +
    '    Walk Time = $0.6 \\times 12 = 7.2$.\n' +
    '    Total = $79.75 + 7.2 = 86.95$.\n' +
    '\n' +
    '    One more thought: Is it possible "laps of 1.75 km" means the laps are shorter, so lap count is f'... 62 more characters
}
Content block: { type: 'text', text: '86.95' }


Token usage: { input_tokens: 88, cache_read_input_tokens: 0, output_tokens: 3641 }
```

Now let's run it with thinking disabled:
```bash
Stop reason: end_turn


Content block: { type: 'text', text: '94' }


Token usage: { input_tokens: 6, cache_read_input_tokens: 84, output_tokens: 3 }
```

Ehhh, that is wild 😅
Running it multiple times will result in different results...
So, what is going on here?
