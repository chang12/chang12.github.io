---
layout: post
title: "Python 에서 async 를 처음으로 써봤다."
tags: []
---

openai api 를 써서 gpt 로 뭔가를 생성하는 program 을 작성했다. test data 를 순회하며, prompt 를 완성하고, api 를 호출한다. 그런데 체감 시간이 길다.

```python
client = get_openai_client()

def process(datum: ...) -> ...:
    content = fill(prompt, datum) 
    message = {"role": "user", "content": content}
    chat_completion = client.chat.completions.create(messages=[message], model=model)
    chat_completion_message_content = chat_completion.choices[0].message.content
    ...
    
results = [process(datum) for datum in data]
...
```

순차적으로 api 를 호출하는 대신, parallel 하게 호출하면 더 빨라질 것이다.

```python
import asyncio


client = get_openai_async_client()

async def process(datum: ...) -> ...:
    content = fill(prompt, datum) 
    message = {"role": "user", "content": content}
    chat_completion = await client.chat.completions.create(messages=messages, model=model)
    chat_completion_message_content = chat_completion.choices[0].message.content
    ...

tasks = [process(datum) for datum in data]
results = await asyncio.gather(*tasks)
...
```

elapsed time 이 ~240 seconds 에서 ~20 seconds 로 감소했다. 
