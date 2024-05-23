---
title: Building the aiMessages iOS App - pt. 4 (OpenAI)
author: luke
date: 2024-05-20 12:00:00 +0500
categories: [Software Engineering, aiMessages]
tags: [programming, typescript, iOS, OpenAI]
image:
  path: /assets/img/aimessages/aiMessagesIcon.png
  alt: aiMessages
---

## Overview

As I've mentioned in earlier posts, the goal of the aiMessages iOS app that I built with [Jake Taylor](https://github.com/jakee417/) was twofold:
1. Bring the ChatGPT "Large Language Model" (LLM) experience to iMessage (both in private and group chats)
2. Allow users to generate photo-realistic images with generative AI via an iMessage Extension App

For item 1, we chose to use OpenAI's ChatGPT. Now, there are many great alternative LLMs to OpenAI, but when we began development in March of 2023, OpenAI was essentially the only option. In this post, I'll walk through our backend code to call the various OpenAI APIs. 

## API

In the examples below, I'll walk through our implementation of each of the different API calls that we used from OpenAI. 

> Please note that this code was implemented March/April 2023, so it's highly likely that the APIs have been updated. Please check the source docs [here](https://platform.openai.com/docs/api-reference/introduction) for the most up-to-date references. 
{: .prompt-warning }

Here is all the code located in `openApiManager.ts`, which includes all of the different API calls we implemented for this app, to include `chat completion`, `text edit`, `image creation`, `image edit`, and `image variation`. For reasons I'll discuss below, although we implemented the API calls for OpenAI's image models (DALLE) and started with them, we ended up switching and using other image generation models for the actual app, but kept these around in case we wanted to use them again in the future. 
 
```ts
// openaiApiManager.ts

// // standard imports
import {
  ChatCompletionRequestMessage,
  CreateChatCompletionResponse,
  CreateEditResponse,
  CreateCompletionResponse,
  Configuration,
  OpenAIApi,
  ImagesResponse,
} from "openai";

// // custom imports
// utils.ts
import {
  base64StringToPngFile,
  encodeAndCountTokens,
} from "../utils/utils";
// errors.ts
import {OpenAIApiError} from "../errors/openaiErrors";
// constants.ts
import {
  BASE_DELAY,
  MAX_RETRIES,
  GPT35_TURBO_MAX_TOKENS,
  DAVINCI3_MAX_TOKENS,
  TOKEN_BUFFER,
  FinishReason,
  LoggingObject,
  LoggingEventType,
  LoggingEventProvider,
  LoggingEventStatus,
} from "../globals";

/**
 * Class for managing openai api calls
 * @class
 * @classdesc Class for managing openai api calls
 * @hideconstructor
 * @memberof module:apis
 * @category Apis
 */
export class OpenaiApiManager {
  private readonly apiKey: string | undefined;

  /**
   * @constructor
   * @param {string} openaiApiKey
   */
  constructor(openaiApiKey: string) {
    // Initialize the api key for openai
    this.apiKey = openaiApiKey;
  }

  /**
   * @param {string} uid
   * @param {string} sessionId
   * @param {ChatCompletionRequestMessage[]} messages
   * @param {number} numTokensRequired
   * @param {string} user - the uid of the user
   * @param {number} retries
   * @throws {OpenAIApiError}
   * @return {Promise<string>} A response to send back to the user. Calls
   * OpenAI's createChatCompletion api. In the case of errors we implement
   * exponential backoff until we get a successful response, or until we exceed
   * the number of MAX_RETRIES. If we never get a successful response back
   * from OpenAIApi, we send a message to the user that something
   * is wrong with OpenAI's api, and to please try again.
   */
  async generateResponseWithOpenAiChatCompletionApi(
    uid: string,
    sessionId: string,
    messages: ChatCompletionRequestMessage[],
    numTokensRequired: number,
    user: string,
    retries: number
  ): Promise<string> {
    const MODEL_ID = process.env.OPENAI_CHAT_COMPLETION_MODEL ?? "";
    const maxTokens = GPT35_TURBO_MAX_TOKENS - numTokensRequired - TOKEN_BUFFER;
    console.log("maxTokens:", maxTokens);
    console.log("numTokensRequired:", numTokensRequired);
    if (!this.apiKey) {
      throw new OpenAIApiError({
        code: 401,
        cause: "Missing Openai API key.",
      });
    }
    const configuration = new Configuration({
      apiKey: this.apiKey,
    });
    const openai = new OpenAIApi(configuration);

    const toLog: LoggingObject = {
      uid: uid,
      session_id: sessionId,
      event_type: LoggingEventType.CHAT_COMPLETION,
      event_provider: LoggingEventProvider.OPEN_AI,
      event_status: LoggingEventStatus.REQUESTED,
    };
    console.log(JSON.stringify(toLog));

    try {
      const response = await openai.createChatCompletion({
        model: MODEL_ID,
        messages: messages,
        temperature: 0.7,
        max_tokens: maxTokens,
        user: user,
      });

      const responseJSON = response.data as CreateChatCompletionResponse;

      const assistantMessageResponse =
        responseJSON.choices[0].message?.content ?? "";
      const finishReason =
        responseJSON.choices[0].finish_reason ?? FinishReason.UNKNOWN;

      // If the finishReason is content filter, it means that OpenAI API
      // response was omitted due to a flag from their content filters.
      // If finishReason is length, it means that OpenAI API response was
      // too long, so we retry with a smaller messages array. Specifically,
      // we remove the oldest user message and the corresponding assistant
      // response, compute a new number of tokens contained in this shorter
      // messages array, then recall the function with the new messages array
      // and new num of tokens required. NOTE: we only do this if the length
      // of the messages array is at least 4, since we need at least the
      // system prompt (i.e., bot personality) and one user message.
      if (finishReason === FinishReason.CONTENT_FILTER) {
        return "OpenAI's has omitted the response due " +
            "to a flag from their content filters. Please reword your " +
            "request, and try again.";
      } else if (
        finishReason === FinishReason.LENGTH &&
        retries < MAX_RETRIES &&
        messages.length >= 4
      ) {
        console.log("finishReason === length, retrying " +
            "API call with shorter array");
        // Remove oldest user message and corresponding assistant response
        // messages[0] is the personality
        // messages.slice(3) creates a new arr with elems from index 3 to end
        const shortenedMessages = [messages[0], ...messages.slice(3)];
        // Compute new number of tokens required
        const newNumTokensRequired: number =
            encodeAndCountTokens(shortenedMessages);
        return this.generateResponseWithOpenAiChatCompletionApi(
          uid,
          sessionId,
          shortenedMessages,
          newNumTokensRequired,
          user,
          retries + 1
        );
      } else {
        const toLog: LoggingObject = {
          uid: uid,
          session_id: sessionId,
          event_type: LoggingEventType.CHAT_COMPLETION,
          event_provider: LoggingEventProvider.OPEN_AI,
          event_status: LoggingEventStatus.COMPLETED,
          http_type: response.status,
        };
        console.log(JSON.stringify(toLog));
        return assistantMessageResponse;
      }
    } catch (error: any) {
      if (retries >= MAX_RETRIES) {
        const toLog: LoggingObject = {
          uid: uid,
          session_id: sessionId,
          event_type: LoggingEventType.CHAT_COMPLETION,
          event_provider: LoggingEventProvider.OPEN_AI,
          event_status: LoggingEventStatus.FAILED,
          http_type: error?.response?.status,
        };
        console.log(JSON.stringify(toLog));
        // pass the code and the error
        throw new OpenAIApiError({
          code: error?.response?.status ?? 400,
          cause: error,
        });
      }
      const delay = BASE_DELAY * 2 ** retries;
      await new Promise((resolve) => setTimeout(resolve, delay));
      console.log("Error", error.message);
      console.log(`Retry number: ${retries}`);
      return this.generateResponseWithOpenAiChatCompletionApi(
        uid,
        sessionId,
        messages,
        numTokensRequired,
        user,
        retries + 1
      );
    }
  }

  /**
   * @param {string} input
   * @param {string} instruction
   * @param {number} retries
   * @return {Promise<string>}
   * @throws {OpenAIApiError}
   * A response to send back to the user. Calls
   * OpenAI's createEdit API. In the case of errors we implement exponential
   * backoff until we get a successful response, or until we exceed
   * the number of MAX_RETRIES. If we never get a successful response back
   * from OpenAIApi, we send a message to the user that
   * something is wrong with OpenAI's api, and to please try again.
   */
  async generateResponseWithOpenAiTextEditApi(
    input: string,
    instruction: string,
    retries: number
  ): Promise<string> {
    // no max_tokens parameter for text-edit
    const MODEL_ID = process.env.OPENAI_TEXT_EDIT_MODEL ?? "";

    if (!this.apiKey) {
      throw new OpenAIApiError({
        code: 401,
        cause: "Missing Openai API key.",
      });
    }

    const configuration = new Configuration({
      apiKey: this.apiKey,
    });
    const openai = new OpenAIApi(configuration);

    try {
      const response = await openai.createEdit({
        model: MODEL_ID,
        input: input,
        instruction: instruction,
        temperature: 0.7,
      });

      const responseJSON = response.data as CreateEditResponse;
      const assistantMessageResponse = responseJSON.choices[0].text ?? "";
      return assistantMessageResponse;
    } catch (error: any) {
      if (retries >= MAX_RETRIES) {
        console.log(error.response);
        // pass the code and the error
        throw new OpenAIApiError({
          code: error.response.status ?? 400,
          cause: error,
        });
      }
      const delay = BASE_DELAY * 2 ** retries;
      await new Promise((resolve) => setTimeout(resolve, delay));
      console.log("Error", error.message);
      console.log(`Retry number: ${retries}`);
      return this.generateResponseWithOpenAiTextEditApi(
        input,
        instruction,
        retries + 1
      );
    }
  }

  /**
   * @param {string} query
   * @param {string} numTokensRequired
   * @param {string} user - the uid of the user
   * @param {number} retries
   * @throws {OpenAIApiError}
   * @return {Promise<string>} A response to send back to the user. Calls
   * OpenAI's createCompletion API. In the case of errors we implement
   * exponential backoff until we get a successful response, or until we exceed
   * the number of MAX_RETRIES. If we never get a successful response back
   * from OpenAIApi, we send a message to the user that
   * something is wrong with OpenAI's api, and to please try again.
   */
  async generateResponseWithOpenAiCompletionApi(
    query: string,
    numTokensRequired: number,
    user: string,
    retries: number
  ): Promise<string> {
    const MODEL_ID = process.env.OPENAI_COMPLETION_MODEL ?? "";
    const maxTokens = DAVINCI3_MAX_TOKENS - numTokensRequired - TOKEN_BUFFER;

    if (!this.apiKey) {
      throw new OpenAIApiError({
        code: 401,
        cause: "Missing Openai API key.",
      });
    }

    const configuration = new Configuration({
      apiKey: this.apiKey,
    });
    const openai = new OpenAIApi(configuration);

    try {
      const response = await openai.createCompletion({
        model: MODEL_ID,
        prompt: query,
        temperature: 0.7,
        max_tokens: maxTokens,
        user: user,
      });

      const responseJSON = response.data as CreateCompletionResponse;
      const assistantMessageResponse = responseJSON.choices[0].text ?? "";
      const finishReason = responseJSON.choices[0].finish_reason ?? "";

      // If the finishReason is content filter, it means that OpenAI API
      // response was omitted due to a flag from their content filters.
      // If finishReason is length, it means that OpenAI API response was
      // too long, so we retry with a large numTokensRequired, which will
      // force the API to return a shorter response (i.e., use less tokens)
      if (finishReason === FinishReason.CONTENT_FILTER) {
        return "OpenAI's has omitted the response due " +
          "to a flag from their content filters. Please reword your " +
          "request, and try again.";
      } else if (
        finishReason === FinishReason.LENGTH &&
        retries < MAX_RETRIES
      ) {
        return this.generateResponseWithOpenAiCompletionApi(
          query,
          numTokensRequired + 100,
          user,
          retries + 1
        );
      } else {
        return assistantMessageResponse;
      }
    } catch (error: any) {
      if (retries >= MAX_RETRIES) {
        // pass the code and the error
        throw new OpenAIApiError({
          code: error.response.status ?? 400,
          cause: error,
        });
      }
      const delay = BASE_DELAY * 2 ** retries;
      await new Promise((resolve) => setTimeout(resolve, delay));
      console.log("Error", error.message);
      console.log(`Retry number: ${retries}`);
      return this.generateResponseWithOpenAiCompletionApi(
        query,
        numTokensRequired,
        user,
        retries + 1
      );
    }
  }

  /**
   * @param {string} prompt
   * @param {string} image
   * @param {string} user - the uid of the user
   * @param {number} retries
   * @throws {OpenAIApiError}
   * @return {Promise<string>} A response to send back to the user. Calls
   * OpenAI's createImageEdit API. In the case of errors we implement
   * exponential backoff until we get a successful response, or until we exceed
   * the number of MAX_RETRIES. If we never get a successful response back
   * from OpenAIApi, we send a message to the user that
   * something is wrong with OpenAI's api, and to please try again.
   */
  async generateImageWithOpenAiEditImageApi(
    prompt: string,
    image: string,
    user: string,
    retries: number): Promise<string> {
    if (!this.apiKey) {
      throw new OpenAIApiError({
        code: 401,
        cause: "Missing Openai API key.",
      });
    }

    const configuration = new Configuration({
      apiKey: this.apiKey,
    });
    const openai = new OpenAIApi(configuration);

    // OpenAI API requires a PNG file, so we convert the base64 string to a PNG
    const pngImage: File = base64StringToPngFile(image);

    try {
      const response = await openai.createImageEdit(
        pngImage,
        prompt,
        undefined,
        1,
        "512x512",
        "b64_json",
        user,
      );
      const responseJSON = response.data as ImagesResponse;
      const imageResponse: string = responseJSON.data[0].b64_json ?? "";
      return imageResponse;
    } catch (error: any) {
      if (retries >= MAX_RETRIES) {
        // pass the code and the error
        throw new OpenAIApiError({
          code: error.response.status ?? 400,
          cause: error,
        });
      }
      const delay = BASE_DELAY * 2 ** retries;
      await new Promise((resolve) => setTimeout(resolve, delay));
      console.log("Error", error.message);
      console.log(`Retry number: ${retries}`);
      return this.generateImageWithOpenAiEditImageApi(
        prompt,
        image,
        user,
        retries + 1
      );
    }
  }

  /**
   * @param {string} image
   * @param {string} user - the uid of the user
   * @param {number} retries
   * @throws {OpenAIApiError}
   * @return {Promise<string>} A response to send back to the user. Calls
   * OpenAI's createImageVariation API. In the case of errors we implement
   * exponential backoff until we get a successful response, or until we exceed
   * the number of MAX_RETRIES. If we never get a successful response back
   * from OpenAIApi, we send a message to the user that
   * something is wrong with OpenAI's api, and to please try again.
   */
  async generateImageWithOpenAiImageVariationApi(
    image: string,
    user: string,
    retries: number
  ): Promise<string> {
    if (!this.apiKey) {
      throw new OpenAIApiError({
        code: 401,
        cause: "Missing Openai API key.",
      });
    }

    const configuration = new Configuration({
      apiKey: this.apiKey,
    });
    const openai = new OpenAIApi(configuration);

    // OpenAI API requires a PNG file, so we convert the base64 string to a PNG
    const pngImage: File = base64StringToPngFile(image);

    try {
      const response = await openai.createImageVariation(
        pngImage,
        1,
        "512x512",
        "b64_json",
        user,
      );
      const responseJSON = response.data as ImagesResponse;
      const imageResponse: string = responseJSON.data[0].b64_json ?? "";
      return imageResponse;
    } catch (error: any) {
      if (retries >= MAX_RETRIES) {
        // pass the code and the error
        throw new OpenAIApiError({
          code: error.response.status ?? 400,
          cause: error,
        });
      }
      const delay = BASE_DELAY * 2 ** retries;
      await new Promise((resolve) => setTimeout(resolve, delay));
      console.log("Error", error.message);
      console.log(`Retry number: ${retries}`);
      return this.generateImageWithOpenAiImageVariationApi(
        image,
        user,
        retries + 1
      );
    }
  }

  /**
   * @param {string} prompt
   * @param {string} user - the uid of the user
   * @param {number} retries
   * @throws {OpenAIApiError}
   * @return {Promise<string>} A response to send back to the user. Calls
   * OpenAI's createImage API. In the case of errors we implement exponential
   * backoff until we get a successful response, or until we exceed
   * the number of MAX_RETRIES. If we never get a successful response back
   * from OpenAIApi, we send a message to the user that
   * something is wrong with OpenAI's api, and to please try again.
   */
  async generateImageWithOpenAiCreateImageApi(
    prompt: string,
    user: string,
    retries: number
  ): Promise<string> {
    if (!this.apiKey) {
      throw new OpenAIApiError({
        code: 401,
        cause: "Missing Openai API key.",
      });
    }

    const configuration = new Configuration({
      apiKey: this.apiKey,
    });
    const openai = new OpenAIApi(configuration);

    try {
      const response = await openai.createImage({
        prompt: prompt,
        size: "512x512",
        n: 1,
        response_format: "b64_json",
        user: user,
      });
      const responseJSON = response.data as ImagesResponse;
      const imageResponse: string = responseJSON.data[0].b64_json ?? "";
      return imageResponse;
    } catch (error: any) {
      if (retries >= MAX_RETRIES) {
        // pass the code and the error
        throw new OpenAIApiError({
          code: error.response.status ?? 400,
          cause: error,
        });
      }
      const delay = BASE_DELAY * 2 ** retries;
      await new Promise((resolve) => setTimeout(resolve, delay));
      console.log("Error", error.message);
      console.log(`Retry number: ${retries}`);
      return this.generateImageWithOpenAiCreateImageApi(
        prompt,
        user,
        retries + 1
      );
    }
  }
}
```

### Chat Completion
To call this API you send a `POST` request to ` https://api.openai.com/v1/chat/completions`, which creates a model response for the provided chat conversation. As discussed in [part 3](https://lwcarani.github.io/posts/aimessages-reflections-3/#imessage) of this blog series, we stored a cache of every user's conversation history and fed this to the API calls to provide more context to the conversation and produce better results. Here's an example of what that might look like:

```sh
# .env
OPENAI_CHAT_COMPLETION_MODEL="gpt-3.5-turbo-0613"
```

```ts
import {
  ChatCompletionRequestMessage,
  CreateChatCompletionResponse,
  CreateEditResponse,
  CreateCompletionResponse,
  Configuration,
  OpenAIApi,
  ImagesResponse,
} from "openai";

const MODEL_ID = process.env.OPENAI_CHAT_COMPLETION_MODEL ?? "";
const maxTokens = GPT35_TURBO_MAX_TOKENS - numTokensRequired - TOKEN_BUFFER;

const messages: ChatCompletionRequestMessage[] = [];
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
  {role: "user", content: "How many teeth do they have?"}
);

const response = await openai.createChatCompletion({
  model: MODEL_ID,
  messages: messages,
  temperature: 0.7,
  max_tokens: maxTokens,
  user: user,
});
```

By providing the message history back to the OpenAI model, the user can just ask "How many teeth do they have?", and the model knows that the user is referring to llamas, since that is the last animal they were discussing. This provides a much more natural, conversational feel for the user, and makes the app much simpler to use. 

And here's the full code wrapper for that API call:

```ts
// openaiApiManager.ts

/**
 * @param {string} uid
 * @param {string} sessionId
 * @param {ChatCompletionRequestMessage[]} messages
 * @param {number} numTokensRequired
 * @param {string} user - the uid of the user
 * @param {number} retries
 * @throws {OpenAIApiError}
 * @return {Promise<string>} A response to send back to the user. Calls
 * OpenAI's createChatCompletion api. In the case of errors we implement
 * exponential backoff until we get a successful response, or until we exceed
 * the number of MAX_RETRIES. If we never get a successful response back
 * from OpenAIApi, we send a message to the user that something
 * is wrong with OpenAI's api, and to please try again.
 */
async generateResponseWithOpenAiChatCompletionApi(
  uid: string,
  sessionId: string,
  messages: ChatCompletionRequestMessage[],
  numTokensRequired: number,
  user: string,
  retries: number
): Promise<string> {
  const MODEL_ID = process.env.OPENAI_CHAT_COMPLETION_MODEL ?? "";
  const maxTokens = GPT35_TURBO_MAX_TOKENS - numTokensRequired - TOKEN_BUFFER;
  console.log("maxTokens:", maxTokens);
  console.log("numTokensRequired:", numTokensRequired);
  if (!this.apiKey) {
    throw new OpenAIApiError({
      code: 401,
      cause: "Missing Openai API key.",
    });
  }
  const configuration = new Configuration({
    apiKey: this.apiKey,
  });
  const openai = new OpenAIApi(configuration);

  const toLog: LoggingObject = {
    uid: uid,
    session_id: sessionId,
    event_type: LoggingEventType.CHAT_COMPLETION,
    event_provider: LoggingEventProvider.OPEN_AI,
    event_status: LoggingEventStatus.REQUESTED,
  };
  console.log(JSON.stringify(toLog));

  try {
    const response = await openai.createChatCompletion({
      model: MODEL_ID,
      messages: messages,
      temperature: 0.7,
      max_tokens: maxTokens,
      user: user,
    });

    const responseJSON = response.data as CreateChatCompletionResponse;

    const assistantMessageResponse =
      responseJSON.choices[0].message?.content ?? "";
    const finishReason =
      responseJSON.choices[0].finish_reason ?? FinishReason.UNKNOWN;

    // If the finishReason is content filter, it means that OpenAI API
    // response was omitted due to a flag from their content filters.
    // If finishReason is length, it means that OpenAI API response was
    // too long, so we retry with a smaller messages array. Specifically,
    // we remove the oldest user message and the corresponding assistant
    // response, compute a new number of tokens contained in this shorter
    // messages array, then recall the function with the new messages array
    // and new num of tokens required. NOTE: we only do this if the length
    // of the messages array is at least 4, since we need at least the
    // system prompt (i.e., bot personality) and one user message.
    if (finishReason === FinishReason.CONTENT_FILTER) {
      return "OpenAI's has omitted the response due " +
          "to a flag from their content filters. Please reword your " +
          "request, and try again.";
    } else if (
      finishReason === FinishReason.LENGTH &&
      retries < MAX_RETRIES &&
      messages.length >= 4
    ) {
      console.log("finishReason === length, retrying " +
          "API call with shorter array");
      // Remove oldest user message and corresponding assistant response
      // messages[0] is the personality
      // messages.slice(3) creates a new arr with elems from index 3 to end
      const shortenedMessages = [messages[0], ...messages.slice(3)];
      // Compute new number of tokens required
      const newNumTokensRequired: number =
          encodeAndCountTokens(shortenedMessages);
      return this.generateResponseWithOpenAiChatCompletionApi(
        uid,
        sessionId,
        shortenedMessages,
        newNumTokensRequired,
        user,
        retries + 1
      );
    } else {
      const toLog: LoggingObject = {
        uid: uid,
        session_id: sessionId,
        event_type: LoggingEventType.CHAT_COMPLETION,
        event_provider: LoggingEventProvider.OPEN_AI,
        event_status: LoggingEventStatus.COMPLETED,
        http_type: response.status,
      };
      console.log(JSON.stringify(toLog));
      return assistantMessageResponse;
    }
  } catch (error: any) {
    if (retries >= MAX_RETRIES) {
      const toLog: LoggingObject = {
        uid: uid,
        session_id: sessionId,
        event_type: LoggingEventType.CHAT_COMPLETION,
        event_provider: LoggingEventProvider.OPEN_AI,
        event_status: LoggingEventStatus.FAILED,
        http_type: error?.response?.status,
      };
      console.log(JSON.stringify(toLog));
      // pass the code and the error
      throw new OpenAIApiError({
        code: error?.response?.status ?? 400,
        cause: error,
      });
    }
    const delay = BASE_DELAY * 2 ** retries;
    await new Promise((resolve) => setTimeout(resolve, delay));
    console.log("Error", error.message);
    console.log(`Retry number: ${retries}`);
    return this.generateResponseWithOpenAiChatCompletionApi(
      uid,
      sessionId,
      messages,
      numTokensRequired,
      user,
      retries + 1
    );
  }
}
```


### Edit Text
This API call creates a model response for a provided body of text. It scans the text, fixing any typos that might have been present, and then returns the corrected response. For example, "walruses like to eet fish" would be corrected to "walruses like to *eat* fish." This functionality was not actually integrated anywhere in our app, but we implemented the API call in case we wanted to incorporate it in the future. 

```ts
// openaiApiManager.ts

/**
 * @param {string} input
 * @param {string} instruction
 * @param {number} retries
 * @return {Promise<string>}
 * @throws {OpenAIApiError}
 * A response to send back to the user. Calls
 * OpenAI's createEdit API. In the case of errors we implement exponential
 * backoff until we get a successful response, or until we exceed
 * the number of MAX_RETRIES. If we never get a successful response back
 * from OpenAIApi, we send a message to the user that
 * something is wrong with OpenAI's api, and to please try again.
 */
async generateResponseWithOpenAiTextEditApi(
  input: string,
  instruction: string,
  retries: number
): Promise<string> {
  // no max_tokens parameter for text-edit
  const MODEL_ID = process.env.OPENAI_TEXT_EDIT_MODEL ?? "";

  if (!this.apiKey) {
    throw new OpenAIApiError({
      code: 401,
      cause: "Missing Openai API key.",
    });
  }

  const configuration = new Configuration({
    apiKey: this.apiKey,
  });
  const openai = new OpenAIApi(configuration);

  try {
    const response = await openai.createEdit({
      model: MODEL_ID,
      input: input,
      instruction: instruction,
      temperature: 0.7,
    });

    const responseJSON = response.data as CreateEditResponse;
    const assistantMessageResponse = responseJSON.choices[0].text ?? "";
    return assistantMessageResponse;
  } catch (error: any) {
    if (retries >= MAX_RETRIES) {
      console.log(error.response);
      // pass the code and the error
      throw new OpenAIApiError({
        code: error.response.status ?? 400,
        cause: error,
      });
    }
    const delay = BASE_DELAY * 2 ** retries;
    await new Promise((resolve) => setTimeout(resolve, delay));
    console.log("Error", error.message);
    console.log(`Retry number: ${retries}`);
    return this.generateResponseWithOpenAiTextEditApi(
      input,
      instruction,
      retries + 1
    );
  }
}
```

### Create Image

To create an image with OpenAI, you simply send a `POST` request to `https://api.openai.com/v1/images/generation` with a text prompt. The model then creates an image based on the prompt. It's even easier than creating an `axios` POST request, however, because this API call is already integrated into the Node.js library that OpenAI maintains. The code for our implementation of this API call can be seen below.

> Although we initially started with using this API for image generation, we quickly switched to Stability AI and Clipdrop once we discovered they were cheaper, just as easy to use, more flexible, and at the time, seemed to produce better results than DALLE. We kept the code around in case we wanted to integrate these API calls back into our app again in the future, or allow users to select which model they want to use to generate images (OpenAI, Stability AI, Clipdrop, etc.).
{: .prompt-info }

```ts
// openaiApiManager.ts

/**
 * @param {string} prompt
 * @param {string} user - the uid of the user
 * @param {number} retries
 * @throws {OpenAIApiError}
 * @return {Promise<string>} A response to send back to the user. Calls
 * OpenAI's createImage API. In the case of errors we implement exponential
 * backoff until we get a successful response, or until we exceed
 * the number of MAX_RETRIES. If we never get a successful response back
 * from OpenAIApi, we send a message to the user that
 * something is wrong with OpenAI's api, and to please try again.
 */
async generateImageWithOpenAiCreateImageApi(
  prompt: string,
  user: string,
  retries: number
): Promise<string> {
  if (!this.apiKey) {
    throw new OpenAIApiError({
      code: 401,
      cause: "Missing Openai API key.",
    });
  }

  const configuration = new Configuration({
    apiKey: this.apiKey,
  });
  const openai = new OpenAIApi(configuration);

  try {
    const response = await openai.createImage({
      prompt: prompt,
      size: "512x512",
      n: 1,
      response_format: "b64_json",
      user: user,
    });
    const responseJSON = response.data as ImagesResponse;
    const imageResponse: string = responseJSON.data[0].b64_json ?? "";
    return imageResponse;
  } catch (error: any) {
    if (retries >= MAX_RETRIES) {
      // pass the code and the error
      throw new OpenAIApiError({
        code: error.response.status ?? 400,
        cause: error,
      });
    }
    const delay = BASE_DELAY * 2 ** retries;
    await new Promise((resolve) => setTimeout(resolve, delay));
    console.log("Error", error.message);
    console.log(`Retry number: ${retries}`);
    return this.generateImageWithOpenAiCreateImageApi(
      prompt,
      user,
      retries + 1
    );
  }
}
```


### Image Edit
To edit an image with OpenAI, you simply send a `POST` request to `https://api.openai.com/v1/images/edits` with an image and text prompt. The model then creates a new image based on the image provided, and the prompt. Just like the `image create` API, it's even easier than creating an `axios` POST request, however, because this API call is already integrated into the Node.js library that OpenAI maintains. The code for our implementation of this API call can be seen below. Once again, this code is not actually integrated into our app, but we kept it around in case we wanted to plug it in at some point in the future. 

```ts
// openaiApiManager.ts

/**
 * @param {string} prompt
 * @param {string} image
 * @param {string} user - the uid of the user
 * @param {number} retries
 * @throws {OpenAIApiError}
 * @return {Promise<string>} A response to send back to the user. Calls
 * OpenAI's createImageEdit API. In the case of errors we implement
 * exponential backoff until we get a successful response, or until we exceed
 * the number of MAX_RETRIES. If we never get a successful response back
 * from OpenAIApi, we send a message to the user that
 * something is wrong with OpenAI's api, and to please try again.
 */
async generateImageWithOpenAiEditImageApi(
  prompt: string,
  image: string,
  user: string,
  retries: number): Promise<string> {
  if (!this.apiKey) {
    throw new OpenAIApiError({
      code: 401,
      cause: "Missing Openai API key.",
    });
  }

  const configuration = new Configuration({
    apiKey: this.apiKey,
  });
  const openai = new OpenAIApi(configuration);

  // OpenAI API requires a PNG file, so we convert the base64 string to a PNG
  const pngImage: File = base64StringToPngFile(image);

  try {
    const response = await openai.createImageEdit(
      pngImage,
      prompt,
      undefined,
      1,
      "512x512",
      "b64_json",
      user,
    );
    const responseJSON = response.data as ImagesResponse;
    const imageResponse: string = responseJSON.data[0].b64_json ?? "";
    return imageResponse;
  } catch (error: any) {
    if (retries >= MAX_RETRIES) {
      // pass the code and the error
      throw new OpenAIApiError({
        code: error.response.status ?? 400,
        cause: error,
      });
    }
    const delay = BASE_DELAY * 2 ** retries;
    await new Promise((resolve) => setTimeout(resolve, delay));
    console.log("Error", error.message);
    console.log(`Retry number: ${retries}`);
    return this.generateImageWithOpenAiEditImageApi(
      prompt,
      image,
      user,
      retries + 1
    );
  }
}
```

### Image Variation
To create an image variation with OpenAI, you simply send a `POST` request to `https://api.openai.com/v1/images/variations` with an image. The model then creates a variation of the provided image. Just like the `image create` and `image edit` APIs, you don't have to go through the hassle of setting up an `axios` POST request, because this API call is already integrated into the Node.js library that OpenAI maintains. The code for our implementation of this API call can be seen below. Once again, this code is not actually integrated into our app, but we kept it around in case we wanted to plug it in at some point in the future. 

```ts
// openaiApiManager.ts

/**
 * @param {string} image
 * @param {string} user - the uid of the user
 * @param {number} retries
 * @throws {OpenAIApiError}
 * @return {Promise<string>} A response to send back to the user. Calls
 * OpenAI's createImageVariation API. In the case of errors we implement
 * exponential backoff until we get a successful response, or until we exceed
 * the number of MAX_RETRIES. If we never get a successful response back
 * from OpenAIApi, we send a message to the user that
 * something is wrong with OpenAI's api, and to please try again.
 */
async generateImageWithOpenAiImageVariationApi(
  image: string,
  user: string,
  retries: number
): Promise<string> {
  if (!this.apiKey) {
    throw new OpenAIApiError({
      code: 401,
      cause: "Missing Openai API key.",
    });
  }

  const configuration = new Configuration({
    apiKey: this.apiKey,
  });
  const openai = new OpenAIApi(configuration);

  // OpenAI API requires a PNG file, so we convert the base64 string to a PNG
  const pngImage: File = base64StringToPngFile(image);

  try {
    const response = await openai.createImageVariation(
      pngImage,
      1,
      "512x512",
      "b64_json",
      user,
    );
    const responseJSON = response.data as ImagesResponse;
    const imageResponse: string = responseJSON.data[0].b64_json ?? "";
    return imageResponse;
  } catch (error: any) {
    if (retries >= MAX_RETRIES) {
      // pass the code and the error
      throw new OpenAIApiError({
        code: error.response.status ?? 400,
        cause: error,
      });
    }
    const delay = BASE_DELAY * 2 ** retries;
    await new Promise((resolve) => setTimeout(resolve, delay));
    console.log("Error", error.message);
    console.log(`Retry number: ${retries}`);
    return this.generateImageWithOpenAiImageVariationApi(
      image,
      user,
      retries + 1
    );
  }
}
```

## Takeaways

Overall, the OpenAI APIs were extremely easy to use (especially with the officially support Node.js `openai` library), and provided great performance. Using OpenAI was very start-up / developer friendly - at our peak we had around 3,000 users, and we never paid more than $1 - $2 a month for using the service. Throughout the development pf our app, OpenAI was continually increasing the amount of tokens you could include in a request (i.e., allowing us to increase the context length, and produce longer responses), making the API calls cheaper, and improving the performance of the models.  

As I've mentioned several times already, OpenAI also supports an official Node.js library for calling the OpenAI APIs, and since I was already using TypeScript for backend development, it was seamless to just integrate the supported library vice something like writing custom `axios` HTTP requests (which also would have worked just fine, but using the `openai` library was easier). 

Finally, the API documentation was fantastic - very comprehensive, easy to read, understand, and follow. This is what you would expect from a massive company / product like OpenAI ChatGPT, but it's still worth noting.

Thanks for reading! In part 5 I plan to cover Stability AI's API and Clipdrop's API in more detail. 
