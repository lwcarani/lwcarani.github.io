---
title: Building the aiMessages iOS App - pt. 2 (Typescript)
author: luke
date: 2024-05-08 12:00:00 +0500
categories: [Software Engineering, aiMessages]
tags: [programming, typescript, iOS]
image:
  path: /assets/img/aiMessagesIcon.png
  alt: aiMessages
---

## Overview

As I mentioned in my [first blog post](https://lwcarani.github.io/posts/aimessages-reflections-1/) of this series, all of our backend code had to be written in TypeScript, because all that was available was Cloud Functions (1st gen). In the 1st generation of cloud functions. the only integrated option was to deploy TypeScript code. With the release of 2nd gen, you can now use Python. I had never used TypeScript before, but I was excited to learn a new language! Writing this app and using TypeScript, I learned a lot new concepts (both TypeScript syntax and general software design principles), to include (but not limited to) `Buffers`, `FormData`, exponential backoff, `.env` (`dotenv`), `enums`, `interfaces`, custom error classes, logging instrumentation or server side logging, the `optional chaining operator`, `nullish coalescing` (as opposed to `nil coalescing`, which can be found in the `Swift` programming language), `async/await`, and HTTP event handling with `axios`.

## Code Example
The following piece of code from the backend of our app actually illustrates all of these coding concepts. It is a function that passes a text `prompt` and image `sketch` from the client to the `Clipdrop` API, which then turns that doodle + prompt into a photo-realistic image, which is then returned to the client. 

```ts
// clipdropApiManager.ts

// // standard imports
// Package for handling Webhooks / POST and other https requests
import axios from "axios";
import FormData from "form-data";

// // custom imports
import {
  base64StringToPngFile,
} from "../utils/utils";
// constants.ts
import {
  BASE_DELAY,
  MAX_RETRIES,
  ImageRequestType,
  LoggingObject,
  LoggingEventType,
  LoggingEventProvider,
  LoggingEventStatus,
  NUM_TRAINING_STEPS,
} from "../globals";
import {
  ClipdropError,
} from "../errors/clipdropErrors";

/**
 * Class for managing clipdrop api calls
 * @class
 * @classdesc Class for managing clipdrop api calls
 * @hideconstructor
 * @memberof module:apis
 * @category Apis
 */
export class ClipdropApiManager {
  private readonly clipdropApiKey: string | undefined;
  private readonly clipdropApiHost: string | undefined;

  /**
   * @constructor
   * @param {string | undefined} clipdropApiKey
   */
  constructor(
    clipdropApiKey: string | undefined
  ) {
    // Initialize the api key for clipdrop
    this.clipdropApiKey = clipdropApiKey;
    this.clipdropApiHost = process.env.CLIPDROP_API_HOST ?? undefined;
  }

  /**
   * Generates an image from the appropriate clipdrop api
   * @param {string} uid
   * @param {string} sessionId
   * @param {ImageRequestType} requestType
   * @param {string} caption
   * @param {string | undefined} image
   * @return {Promise<string>}
   * @throws {ClipdropError}
   */
  async callClipdropApi(
    uid: string,
    sessionId: string,
    requestType: ImageRequestType,
    caption: string,
    image?: string | undefined
  ): Promise<string[]> {
    let imageResponseArray: string[] = [];
    let httpType = 0;
    const userProvidedImage = image ?? "";

    switch (requestType) {
    case ImageRequestType.DOODLE: {
      const toLog: LoggingObject = {
        uid: uid,
        session_id: sessionId,
        event_type: LoggingEventType.DOODLE,
        event_provider: LoggingEventProvider.CLIPDROP,
        event_status: LoggingEventStatus.REQUESTED,
      };
      console.log(JSON.stringify(toLog));
      try {
        ({imageResponseArray, httpType} =
          await this.generateResponseWithClipdropSketchToImageApi(
            caption,
            userProvidedImage,
            0
          ));
        const toLog: LoggingObject = {
          uid: uid,
          session_id: sessionId,
          event_type: LoggingEventType.DOODLE,
          event_provider: LoggingEventProvider.CLIPDROP,
          event_status: LoggingEventStatus.COMPLETED,
          http_type: httpType,
        };
        console.log(JSON.stringify(toLog));
      } catch (error: any) {
        // if error, log status, then throw error to populate up the stack
        const toLog: LoggingObject = {
          uid: uid,
          session_id: sessionId,
          event_type: LoggingEventType.DOODLE,
          event_provider: LoggingEventProvider.CLIPDROP,
          event_status: LoggingEventStatus.FAILED,
          http_type: error.code,
          num_steps: NUM_TRAINING_STEPS,
        };
        console.log(JSON.stringify(toLog));
        throw new ClipdropError({
          message: error.message,
          code: error.code,
          cause: error.cause,
        });
      }
      break;
    }
    default: {
      console.log("Entering default case statement for ImageRequestType");
      imageResponseArray = [];
    }
    }
    return imageResponseArray;
  }

  /**
   * @param {string} prompt
   * @param {string} image
   * @param {number} retries
   * @throws {StabilityError}
   * @return {Promise<{string, number}>} A response to send back to the user.
   * Calls Clipdrop's sketch-to-image API (Clipdrop is by Stability AI).
   * In the case of errors we implement exponential backoff until we get a
   * successful response, or until we exceed the number of MAX_RETRIES.
   */
  private async generateResponseWithClipdropSketchToImageApi(
    prompt: string,
    image: string,
    retries: number
  ): Promise<{imageResponseArray: string[], httpType: number}> {
    const imageResponse = [];

    if (!this.clipdropApiKey) {
      throw new ClipdropError({
        message: "Missing Clipdrop API key.",
        code: 401,
        cause: "Missing Clipdrop API key.",
      });
    }
    const pngImage: File = base64StringToPngFile(image);
    const form = new FormData();
    form.append("sketch_file", pngImage);
    form.append("prompt", prompt);

    try {
      const response =
        await axios
          .post(
            `${this.clipdropApiHost}/sketch-to-image/v1/sketch-to-image`,
            form, {
              headers: {
                "x-api-key": this.clipdropApiKey,
                ...form.getHeaders(),
              },
              responseType: "arraybuffer",
            });

      const buffer: Buffer = Buffer.from(response.data, "binary");
      const base64String: string = buffer.toString("base64");
      imageResponse.push(base64String);

      return {imageResponseArray: imageResponse, httpType: response.status};
    } catch (error: any) {
      if (retries >= MAX_RETRIES) {
        console.log(`Maximum retries exceeded: ${retries}`);
        console.log("Error cause: ", error?.response?.data?.error);
        throw new ClipdropError({
          message: "Non-200 response: maximum retries exceeded.",
          code: error?.response?.status,
          cause: error?.response?.data?.error,
        });
      }
      const delay = BASE_DELAY * 2 ** retries;
      await new Promise((resolve) => setTimeout(resolve, delay));
      console.log(
        `Error: Non-200 response (${error?.response?.status}).
        Retry number: ${retries}`
      );
      return this.generateResponseWithClipdropSketchToImageApi(
        prompt,
        image,
        retries + 1
      );
    }
  }
} // end `ClipdropApiManager` class

```

Let's walk through some of the components.

### Buffers

```ts
const buffer: Buffer = Buffer.from(response.data, "binary");
const base64String: string = buffer.toString("base64");
```

`Buffer` is a way to store and manipulate binary data in Node.js. More specifically, `Buffer` objects are used to represent a fixed-length sequence of bytes, where each buffer corresponds to some raw memory allocated outside of V8's heap (when we say outside of V8's heap, we're referring to memory that's managed directly by the operating system, not by the V8 engine) (V8 is the JavaScript engine inside of node.js that parses and executes JavaScript code). For this project, I frequently used `Buffer` to switch between encoding and decoding image data to and from base64 encoded strings. The first line of code above creates a new `Buffer` object from `response.data` using the "binary" encoding. The second line converts that `Buffer` object into a string using base64 encoding. Base64 encoding schemes are commonly used when there is a need to encode binary data, and I found it was the default choice used when interfacing with different AI image generation APIs (Stability, Clipdrop, OpenAI, etc.). 


### Axios

```ts
const response =
  await axios
    .post(
      `${this.clipdropApiHost}/sketch-to-image/v1/sketch-to-image`,
      form, {
      headers: {
        "x-api-key": this.clipdropApiKey,
        ...form.getHeaders(),
      },
      responseType: "arraybuffer",
    });
```

[Axios](https://axios-http.com/docs/intro) is a JavaScript library used to make HTTP requests from node.js (or XMLHttpRequests from the browser). It is similar to the `fetch` API, but offers a bit more functionality. `Fetch` is also built-in whereas `axios` is a stand-along third party package that needs to be installed. For this project, I used `axios` because it greatly simplified the process of sending asynchronous HTTP requests from node.js to other servers, and also made it very simple to handle the responses. 


### Async/Await

Modern JavaScript/TypeScript added a way to handle function callbacks in an elegant way by adding a Promise-based API that has special syntax to allow us to treat asynchronous code as though it is synchronous. This new language feature does add a bit of complexity - making a function `async` means your values are wrapped in Promises. So what used to return a `string` now returns a `Promise<string>`. 

```ts
const func = () => ":wave:";
const asyncFunc = async () => ":wave:";

const myString = func();
const myPromiseString = asyncFunc();

// myString is a string
myString.length;  // does NOT throw an error

// myPromiseString is a Promise, not the string
myPromiseString.length;  // throws error
```

Using the `await` keyword allows us to easily convert a `promise` into its value. Note that `await` must be used within an `async` function:

```ts
// You can use the await keyword to convert a promise
// into its value. Today, these only work inside an async
// function.

const myWrapperFunction = async () => {
  const myString = func();
  const myResolvedPromiseString = await asyncFunc();

  // Via the await keyword, now myResolvedPromiseString
  // is a string
  myString.length;
  myResolvedPromiseString.length;
```

Pulling directly from the [TypeScript site](https://www.typescriptlang.org/play/?#example/async-await), we see that the `async/await` pattern allows us to greatly simplify code and make it much more readable:

```ts
// Async/Await took code which looked like this:

// getResponse(url, (response) => {
//   getResponse(response.url, (secondResponse) => {
//     const responseData = secondResponse.data
//     getResponse(responseData.url, (thirdResponse) => {
//       ...
//     })
//   })
// })

// And let it become linear like:

// const response = await getResponse(url)
// const secondResponse = await getResponse(response.url)
// const responseData = secondResponse.data
// const thirdResponse = await getResponse(responseData.url)
// ...

// Which can make the code sit closer to left edge, and
// be read with a consistent rhythm.
```

### FormData

```ts
const form = new FormData();
form.append("sketch_file", pngImage);
form.append("prompt", prompt);

try {
  const response =
    await axios
      .post(
        `${this.clipdropApiHost}/sketch-to-image/v1/sketch-to-image`,
        form, {
          headers: {
            "x-api-key": this.clipdropApiKey,
            ...form.getHeaders(),
          },
          responseType: "arraybuffer",
        });
  ...
}
```

And here's another example of using `FormData` from the Stability AI API call of our backend code:

```ts
// stabilityApiManager.ts
const imageBuffer = Buffer.from(image, "base64");
const formData = new FormData();
formData.append("init_image", imageBuffer);
formData.append("init_image_mode", "IMAGE_STRENGTH");
formData.append("image_strength", 0.35);
formData.append("text_prompts[0][text]", prompt);
formData.append("cfg_scale", 7);
formData.append("clip_guidance_preset", "FAST_BLUE");
formData.append("samples", numSamples);
formData.append("steps", NUM_TRAINING_STEPS);
```

The [FormData](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_FormData_Objects) object integrates seamlessly with `axios`. It allows you to compile a set of key/value pairs to send using the `axios` API (or `Fetch` or `XMLHttpRequest`), so it makes it easy to call APIs with keyed data or send keyed data via a POST request.

### Interfaces and Enums

It's not immediately obvious from the code, but `LoggingObject` is actually an interface I defined in the `globals.ts` file of this project:

```ts
// globals.ts
export interface LoggingObject {
  session_id: string;
  uid?: string | undefined;
  event_type?: LoggingEventType | undefined;
  event_provider?: LoggingEventProvider | undefined;
  event_status?: LoggingEventStatus | undefined;
  http_type?: number | undefined;
  http_info?: string | undefined;
  num_steps?: number | undefined;
}
```

An `interface` in TypeScript is similar to other languages, in that it is a syntactical contract that defines the expected structure of an object. By allowing us to describe objects in this manner, the type checker in TypeScript can catch more errors, give better warning messages, and allow us to write more efficient, readable code. A trailing `?` indicates an optional property, so in the code above we see that every property is optional, except for `session_id`. Further specifying the type, you'll see that the optional `event_type` property must be of type `LoggingEventType` (which is an `enum`), or `undefined`. `LoggingEventType` is also defined in the `globals.ts` file of my project:

```ts
// globals.ts
export enum LoggingEventType {
  IMAGE_CREATE = "image_create",
  IMAGE_EDIT = "image_edit",
  IMAGE_EDIT_WITH_MASK = "image_edit_with_mask",
  DOODLE = "doodle",
  TEXT_EDIT = "text_edit",
  CHAT_COMPLETION = "chat_completion",
  COMPLETION = "completion",
  INCOMING_WEBHOOK = "incoming_webhook",
  MESSAGE_AUTH = "message_auth",
  SEND_MESSAGE = "send_message",
  FIREBASE_QUERY = "firebase_query",
  FIREBASE_STORAGE_UPLOAD = "firebase_storage_upload",
  FIREBASE_CLOUD_URL_RETRIEVAL = "firebase_cloud_url_retrieval",
  PUBLICATION = "publication",
}
```

Where `interfaces` specify a syntactic contract, `enums` allow you to define a set of named constants. This makes it easier to document the intent of your code to other developers, improves the modularity of the code, and improves readability. 

### Optional Chaining

```ts
throw new ClipdropError({
  message: "Non-200 response: maximum retries exceeded.",
  code: error?.response?.status,
  cause: error?.response?.data?.error,
});
```

The optional chaining operator (`?.`) accesses an object's property or calls a function. If the object accessed or function called using this operator is `undefined` or `null` the expression short circuits and evaluates to `undefined` instead of throwing an error. This results in much shorter and simpler expressions when accessing chained properties where the possibility exists that a reference may be missing. 

Before optional chaining, the `cause` property value would have to be unwrapped in the following way:

```ts
const cause = error.response && error.response.data && error.response.data.error
```

This used to be idiomatic JavaScript (TypeScript), but it is much more verbose, and actually unsafe. Where optional chaining only checks for `undefined` or `null`, `&&` checks for any `falsy` value, like `0`. Short-circuiting and setting the `cause` property equal to `0` may not be desirable, so optional chaining with `?.` helps prevent these unexpected errors. 

### Nullish Coalescing

```ts
const userProvidedImage = image ?? "";
```

The nullish coalescing operator (`??`) is a logical operator that returns the right-hand side when the left-hand side is `null` or `undefined`, and otherwise returns the left-hand. There is a subtle and important distinction with using `??` compared to `||`, which is the logical OR operator. Before `??` was introduced, a common pattern was to use the boolean logical operator `||` to check if a value was *falsy*, like so:

```ts
const count = 0;
const text = "";

const qty = count || 42;
const message = text || "hi!";
console.log(qty); // 42 and not 0
console.log(message); // "hi!" and not ""
```

This could cause unexpected consequences and behavior if in this case, you consider `0` and `""` valid values. `??` helps avoid this pitfall by only short-circuiting on `null` or `undefined`. 

```ts
const count = 0;
const text = "";

const qty = count ?? 42;
const message = text ?? "hi!";
console.log(qty); // 0 and not 42
console.log(message); // "" and not "hi!"
```


### Exponential Backoff

Exponential backoff is a standard error handling strategy for network applications in which a procedure periodically retries a failed request with increasing delays between requests. It can be a very useful algorithm that uses some sort of feedback to multiplicatively decrease the rate of the same process or request, until an OK response is received. Below is an example:

```ts
try {
  const response =
    await axios
      .post(
        `${this.clipdropApiHost}/sketch-to-image/v1/sketch-to-image`,
        form, {
          headers: {
            "x-api-key": this.clipdropApiKey,
            ...form.getHeaders(),
          },
          responseType: "arraybuffer",
        });

  const buffer: Buffer = Buffer.from(response.data, "binary");
  const base64String: string = buffer.toString("base64");
  imageResponse.push(base64String);

  return { imageResponseArray: imageResponse, httpType: response.status };
} catch (error: any) {
  if (retries >= MAX_RETRIES) {
    console.log(`Maximum retries exceeded: ${retries}`);
    console.log("Error cause: ", error?.response?.data?.error);
    throw new ClipdropError({
      message: "Non-200 response: maximum retries exceeded.",
      code: error?.response?.status,
      cause: error?.response?.data?.error,
    });
  }
  const delay = BASE_DELAY * 2 ** retries;
  await new Promise((resolve) => setTimeout(resolve, delay));
  console.log(
    `Error: Non-200 response (${error?.response?.status}).
    Retry number: ${retries}`
  );
  return this.generateResponseWithClipdropSketchToImageApi(
    prompt,
    image,
    retries + 1
  );
}
```

Here we see that if our `axios` POST request to Clipdrop fails, we `catch` the bad response, wait some `delay` amount that exponentially increases proportionally to the number of retry attempts we've made so far, and then retries the HTTP request again. If we exceed our `MAX_RETRIES` limit then we simply return an error to the client. 

### Custom Errors

In the above code example we see that we have a custom `ClipdropError` implemented:

```ts
throw new ClipdropError({
  message: "Non-200 response: maximum retries exceeded.",
  code: error?.response?.status,
  cause: error?.response?.data?.error,
});
```

And the full `error class` is here:

```ts
// clipdropErrors.ts

type ErrorName =
  | "CLIPDROP_ERROR";

/**
 * Error class for Clipdrop API errors
 */
export class ClipdropError extends Error {
  name: ErrorName;
  message: string;
  cause: string;
  code: number;

  /**
   * @param {{code: number, cause: any}} code - The error code and cause
   */
  constructor({
    message,
    code,
    cause,
  }: {
    message: string
    code: number,
    cause: string
  }) {
    super();
    this.code = code;
    this.cause = cause;
    this.message = message;
    this.name = "CLIPDROP_ERROR";
  }
}
```

The custom error class here allows us to differentiate between error types throughout our program and attach extra data to the errors being thrown, which in turn let's us provide more detailed error messages back to the client and to the console log output, increasing the effectiveness and efficiency of debugging. 

Custom error classes can be even more complicated, like the one below, again, allowing us to capture more nuanced information about the nature of the error and share better error messages to the client and output better error messages to console log output on the server:

```ts
// openaiErrors.ts

type ErrorName =
  | "OPENAI_API_ERROR"
  | "OPENAI_API_ERROR_CODE_401"
  | "OPENAI_API_ERROR_CODE_429"
  | "OPENAI_API_ERROR_CODE_500"
  | "OPENAI_API_ERROR_CODE_UNKNOWN";

type ErrorMessages = {
  [name in ErrorName]: string;
};

export const OPENAI_ERROR_MESSAGES: ErrorMessages = {
  OPENAI_API_ERROR: "OpenAI error.",
  OPENAI_API_ERROR_CODE_401: "OpenAI servers had an error " +
    "while processing your request. Please retry your request " +
    "after a brief wait, and contact us if the issue persists.",
  OPENAI_API_ERROR_CODE_429: "OpenAI servers are currently " +
    "experiencing higher than normal traffic. Please retry " +
    "your request after a brief wait.",
  OPENAI_API_ERROR_CODE_500: "OpenAI servers had an error " +
    "while processing your request. Please retry your request " +
    "after a brief wait, and contact us if the issue persists.",
  OPENAI_API_ERROR_CODE_UNKNOWN: "OpenAI servers had an unknown error " +
    "while processing your request. Please retry your request " +
    "after a brief wait, and contact us if the issue persists.",
};

/**
 * Error class for Openai API errors
 */
export class OpenAIApiError extends Error {
  name: ErrorName;
  message: string;
  cause: any;
  code: number;

  /**
   * @param {{code: number, cause: any}} code - The error code and cause
   */
  constructor({
    code,
    cause,
  }: {
    code: number,
    cause?: any
  }) {
    super();
    this.code = code;
    this.cause = cause;
    // check to see if OpenAI returned an error message
    // otherwise, just use one of the generic error messages
    const openaiErrorMessage: string =
      cause?.response?.data?.error?.message ?? "";
    if (openaiErrorMessage) {
      this.name = "OPENAI_API_ERROR";
      this.message = openaiErrorMessage;
    } else {
      switch (code) {
        case 401:
          this.name = "OPENAI_API_ERROR_CODE_401";
          this.message = OPENAI_ERROR_MESSAGES.OPENAI_API_ERROR_CODE_401;
          break;
        case 429:
          this.name = "OPENAI_API_ERROR_CODE_429";
          this.message = OPENAI_ERROR_MESSAGES.OPENAI_API_ERROR_CODE_429;
          break;
        case 500:
          this.name = "OPENAI_API_ERROR_CODE_500";
          this.message = OPENAI_ERROR_MESSAGES.OPENAI_API_ERROR_CODE_500;
          break;
        default:
          this.name = "OPENAI_API_ERROR_CODE_UNKNOWN";
          this.message = OPENAI_ERROR_MESSAGES.OPENAI_API_ERROR_CODE_UNKNOWN;
      }
    }
  }
}
```

### dotenv

`dotenv` or `.env` is a zero-dependency module that loads environment variables from a `.env` file into `process.env`. Then during runtime, these variables are loaded, and can be accessed throughout the application, like so:

```ts
this.clipdropApiHost = process.env.CLIPDROP_API_HOST ?? undefined;
```

Specifying a `.env` file is simple. For the aiMessages project, it looks like this:

```sh
# .env
LOOP_SENDER_NAME="aimessages@imsg.chat"
LOOP_API_HOST="https://server.loopmessage.com/api/v1/message/send/"
LOOP_API_AUTH_HOST="https://iauth.loopmessage.com/auth/api/v1/init/"
STABILITY_API_HOST="https://api.stability.ai"
STABILITY_DIFFUSION_ENGINE="stable-diffusion-v1-5"
STABILITY_DIFFUSION_XL_BETA_ENGINE="stable-diffusion-xl-beta-v2-2-2"
STABILITY_DIFFUSION_XL_ENGINE="stable-diffusion-xl-1024-v1-0"
STABILITY_INPAINTING_ENGINE="stable-inpainting-512-v2-0"
CLIPDROP_API_HOST="https://clipdrop-api.co"
OPENAI_TEXT_EDIT_MODEL="text-davinci-edit-001"
OPENAI_COMPLETION_MODEL="text-davinci-003"
OPENAI_CHAT_COMPLETION_MODEL="gpt-3.5-turbo-0613"
```


> These variables are not secure, so it is important NOT to store API keys or other secrets and/or passwords in `.env`. For API keys and other sensitive bits of data, a secret manager or password manager should be used. For aiMessages, we used Google/Firebase's [Secret Manager](https://firebase.google.com/docs/functions/config-env?gen=2nd).
{: .prompt-danger }


### Logging Instrumentation

To track the status of the Clipdrop API call, I included structured `console.log()` statements throughout the function call. You can see them below scattered throughout the `switch` statement:

```ts
// clipdropApiManager.ts

switch (requestType) {
case ImageRequestType.DOODLE: {
  const toLog: LoggingObject = {
    uid: uid,
    session_id: sessionId,
    event_type: LoggingEventType.DOODLE,
    event_provider: LoggingEventProvider.CLIPDROP,
    event_status: LoggingEventStatus.REQUESTED,
  };
  console.log(JSON.stringify(toLog));
  try {
    ({imageResponseArray, httpType} =
      await this.generateResponseWithClipdropSketchToImageApi(
        caption,
        userProvidedImage,
        0
      ));
    const toLog: LoggingObject = {
      uid: uid,
      session_id: sessionId,
      event_type: LoggingEventType.DOODLE,
      event_provider: LoggingEventProvider.CLIPDROP,
      event_status: LoggingEventStatus.COMPLETED,
      http_type: httpType,
    };
    console.log(JSON.stringify(toLog));
  } catch (error: any) {
    // if error, log status, then throw error to populate up the stack
    const toLog: LoggingObject = {
      uid: uid,
      session_id: sessionId,
      event_type: LoggingEventType.DOODLE,
      event_provider: LoggingEventProvider.CLIPDROP,
      event_status: LoggingEventStatus.FAILED,
      http_type: error.code,
      num_steps: NUM_TRAINING_STEPS,
    };
    console.log(JSON.stringify(toLog));
    throw new ClipdropError({
      message: error.message,
      code: error.code,
      cause: error.cause,
    });
  }
  break;
}
default: {
  console.log("Entering default case statement for ImageRequestType");
  imageResponseArray = [];
}
}
```

If you trace the functional call logic, you can see that we update the event status logging when the API call is initiated (or requested), when it is completed, or when it fails. By conducting structured logging in this manner, we can conduct analysis of our logs to see where bottlenecks might be occurring. Specifically, we used Google's BigQuery to analyze our logs, which was only possible due to the structured manner in which we placed our `console.log()` statements. In fact, in a different place in our backend logic, this server side logging was crucial in identifying a massive slowdown in performance due to sub-optimal code, allowing us to immediately identify the bad section of code, fix the issue, and eliminate the latency.


## Final Thoughts

I have a strong preference for TypeScript over vanilla JavaScript, as I found the type checker to be very helpful. Using TypeScript with this project was my first foray into the `async/await` paradigm and HTTP event handling, and I found the experience to very enjoyable, and rewarding!

Thanks for reading! In part 3 I plan to cover our use of the LoopMessage API for communicating with users via iMessage.
