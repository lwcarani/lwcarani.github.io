---
title: Conversational Memory for LLMs
author: luke
date: 2024-10-15 12:00:00 +0500
categories: [Software Engineering, aiMessages]
tags: [programming, large-language-models]
mermaid: true
image:
  path: /assets/img/conversational memory.png
---

## Overview

One of the key technical problems Jake and I needed to solve when building [aiMessages](https://lwcarani.github.io/posts/aimessages-reflections-1/) was support for conversational memory. Conversational memory is how a chatbot (like aiMessages) can respond to multiple queries in a chat-like manner. Effective implementation enables coherent conversation. Without it, every interaction with the large language model (LLM) would be treated as an independent query without considering past questions or context. 

To illustrate this point, let's say you sent a text from your phone to aiMessages saying "What color are walruses?". The LLM then responds "Walruses are brown!". In a text message conversation, you want to be able to say, "and where do they live?", and you want the LLM to *know* that you are referring to walruses - you don't want to have to specify that every time. By providing the cached message history, that's exactly what we achieved with aiMessages. 

Consider the following:

<div style="font-family: Arial, sans-serif; line-height: 1.6; margin: 20px 0; padding: 20px; background-color: #f0f0f0; max-width: 100%; overflow-x: auto;">
  <style scoped>
    .container { display: flex; justify-content: space-between; }
    .column { width: 48%; background-color: white; padding: 15px; border-radius: 10px; box-shadow: 0 0 10px rgba(0,0,0,0.1); }
    h3 { color: blue; text-align: center; text-decoration: underline; }
    .user-input { background-color: #e6f7ff; border-radius: 10px; padding: 10px; margin-bottom: 10px; color: black; }
    .llm-response { background-color: #f0f0f0; border-radius: 10px; padding: 10px; margin-bottom: 10px; color: black; }
    .conversation-history { border-top: 2px dotted #999; margin-top: 20px; padding-top: 20px; }
    .no-memory { color: red; font-style: italic; }
    @media (max-width: 768px) {
      .container { flex-direction: column; }
      .column { width: 100%; margin-bottom: 20px; }
    }
  </style>
  <div class="container">
    <div class="column">
      <h3>With conversational memory</h3>
      <div class="user-input">What color are walruses, and what do they like to eat?</div>
      <div class="llm-response">Walruses are brown, and like to eat clams and mollusks.</div>
      <div class="user-input">Where do they live?</div>
      <div class="llm-response">They live in the Arctic and subarctic regions!</div>
      <div class="conversation-history">
        <div class="user-input">What were we talking about?</div>
        <div class="llm-response">You were asking me about walruses.</div>
      </div>
    </div>
    <div class="column">
      <h3>Without conversational memory</h3>
      <div class="no-memory">(No conversation history is stored)</div>
      <div class="user-input">What color are walruses, and what do they like to eat?</div>
      <div class="llm-response">Walruses are brown, and like to eat clams and mollusks.</div>
      <div class="user-input">Where do they live?</div>
      <div class="llm-response">Sorry I have no idea what you're talking about. Where does who live?</div>
      <div class="conversation-history">
        <div class="user-input">What were we talking about?</div>
        <div class="llm-response">Sorry I have no idea what you're talking about!</div>
      </div>
    </div>
  </div>
</div>

Clearly, no conversational memory is rather useless. It would be no good to require the user to regurgitate all relevant context every time they ask a follow-up question. Unfortunately, by default LLMs are *stateless*, meaning each query is processed independently of all other interactions, so there is no inherent sense of a continuous session. Since aiMessages was integrated into iMessages in a texting environment, remembering previous interactions was crucial for a seamless user experience. Conversational memory allowed us to give this experience to the user.

## Implementation

To achieve conversational memory, we automatically cached the `N` previous messages from the user's conversation, to provide context to the LLM when generating new responses. `N` is set in [globals.ts](https://github.com/lwcarani/aiMessages-backend-public/blob/main/functions/src/globals.ts), typically between `5` and `10`.

```ts
// globals.ts
export const MAX_MESSAGE_HISTORY: number = process.env.NODE_ENV === "test" ? 5 : 10;
```

Without this message caching, the LLM (in this case, OpenAI's API), would not be able to provide high fidelity responses. As discussed previously, each API call is independent, so there's no way for the OpenAI API to differentiate between users and "remember" the conversation. To achieve this experience, we would pass along the most recent messages from the user (along with the previous response generated by the LLM) to provide context, improving the quality of responses. This was tricky, because we had to send enough previous messages to provide adequate context, while also ensuring we didn't exceed the max tokens for the API we were using.

Fortunately, the OpenAI [API docs](https://platform.openai.com/docs/api-reference/chat/create) make this rather simple to implement. As part of the request body, you can simply add a `messages` parameter with a list of messages comprising the current state of the conversation. The `system` role is where you can specify how you want the LLM to respond (for aiMessages, this is where we included the `personality` of the bot, as specified by the user). Then, you simply append previous user and LLM responses in alternating order, recreating the conversation history so that the LLM has complete context of the most recent part of the conversation, and can produce the optimal next response.

```ts
 messages.push(
      {role: "system", content: "a helpful assistant"},
      {role: "user", content: "Hello there!"},
      {role: "assistant", content: "Hi, nice to meet you!"},
      {role: "user", content: "What do walruses eat?"},
      {role: "assistant", content: "Fish, seaweed, and other marine things"},
      {role: "user", content: "What do rhinos eat?"},
      {role: "assistant", content: "Plants, grass, and other land things"},
      {role: "user", content: "What do elephants eat?"},
      {role: "assistant", content: "Elephant food"},
      {role: "user", content: "What color are llamas?"},
      {role: "assistant", content: "Llamas are gray"},
      {role: "user", content: "How many tentacles does a squid have?"}
    );
```

#### Private

To see all of the code on how we implemented conversational memory for private iMessage conversations with aiMessages, you can check out the [privateMessageHandler](https://github.com/lwcarani/aiMessages-backend-public/blob/main/functions/src/handlers.ts#L272) 
function inside [handlers.ts](https://github.com/lwcarani/aiMessages-backend-public/blob/main/functions/src/handlers.ts)

#### Group

For group chats, the `messages` parameter was organized very similarly to the private message chat, with one small adjustment. Instead of the `content` parameter only containing the raw text from the conversation, we prepend the phone number or email address of the user who sent that message, so that the LLM has improved context as to which user sent which message:

```ts
 messages.push(
      {role: "system", content: "You are a helpful concierge helping us find a restaurant."},
      {role: "user", content: "+12241112233: Hello there!"},
      {role: "assistant", content: "Hi, nice to meet you +12241112233!"},
      {role: "user", content: "+12241112233: Where should we go to dinner in Buffalo New York?"},
      {role: "user", content: "+18004445566: Make sure it has vegan options"},
      {role: "user", content: "luke@foobar.net: aiMessages, can you also make sure they serve good cocktails?"},
      {role: "assistant", content: "+12241112233 'foobar-YUMYUM' is a great choice with plenty of seating in Buffalo. +18004445566, they have plenty of great vegan options, and don't worry luke@foobar.net, they have world famous cocktails too, recommended by Bobby Flay himself."},
      {role: "user", content: "+18004445566: Who is Bobby Flay?"},
      {role: "assistant", content: "+18004445566 it makes sense you wouldn't know who Bobby Flay is, he loves to grill meat, and since you are a vegan, you probably wouldn't follow his cooking shows or recipes. Rest assured, you can trust his opinion, and mine!"},
      {role: "user", content: "+12241112233: Thanks for all of the help, aiMessages!"}
    );
```

To see all of the code on how we implemented conversational memory for group iMessage conversations with aiMessages, you can check out the [groupMessageHandler](https://github.com/lwcarani/aiMessages-backend-public/blob/main/functions/src/handlers.ts#L494) 
function inside [handlers.ts](https://github.com/lwcarani/aiMessages-backend-public/blob/main/functions/src/handlers.ts)

There were several key distinctions in how we implemented group chat responses compared to private chat responses. Whenever we received a private message from the user, we would always respond, because the assumption is that if the user sends a text to the aiMessages chatbot, they intend to receive a response back. However, for group chats, we defaulted to only responding if someone in the group: 
1. Had a valid, active aiMessages account
2. Had message credits remaining
3. Actually mentioned the **name** of their bot in the text message. Sometimes people fire off several texts in a group without actually asking any meaningful questions - we didn't want to generate an LLM response every single time, only when the user specifically wanted the bot's input with something like "hey aiMessages..."

## Takeaways

Thanks for reading, and hopefully you learned something!

> Our implementation is probably most similar to what is available from LangChain today as [ConversationBufferMemory](https://api.python.langchain.com/en/latest/memory/langchain.memory.buffer.ConversationBufferMemory.html). For other ways to implement "Conversational Memory" with LLMs, I recommend searching terms like `ConversationSummaryMemory`, `ConversationBufferWindowMemory`, `ConversationSummaryBufferMemory`, `ConversationKnowledgeGraphMemory` on LangChain's site.
{: .prompt-info }
