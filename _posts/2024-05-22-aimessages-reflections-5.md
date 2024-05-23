---
title: Building the aiMessages iOS App - pt. 5 (Stability AI + Clipdrop)
author: luke
date: 2024-05-22 12:00:00 +0500
categories: [Software Engineering, aiMessages]
tags: [programming, typescript, iOS, Stability AI, Clipdrop]
image:
  path: /assets/img/aimessages/aiMessagesIcon.png
  alt: aiMessages
---

## Overview

As I've mentioned in earlier posts, the goal of the aiMessages iOS app that I built with [Jake Taylor](https://github.com/jakee417/) was twofold:
1. Bring the ChatGPT "Large Language Model" (LLM) experience to iMessage (both in private and group chats)
2. Allow users to generate photo-realistic images with generative AI via an iMessage Extension App

For item 2, we needed to identify a generative AI image generation API to use. We started with OpenAI's DALLE, but quickly switched to Stability AI and Clipdrop because they were cheaper to use, had great API docs, and produced equally stunning images. In this post, I'll walk through our backend code to call the various Stability AI and Clipdrop APIs. 

## API

In the examples below, I'll walk through our implementation of each of the different API calls that we used from Stability AI and Clipdrop. 

> Please note that this code was implemented March/April 2023, so it's highly likely that the APIs have been updated. Please check the source docs [here](https://platform.stability.ai/docs/getting-started) (for Stability) and [here](https://clipdrop.co/apis/docs/sketch-to-image) (for Clipdrop) for the most up-to-date references. 
{: .prompt-warning }

Here is all the code located in `clipdropApiManager.ts` and `stabilityApiManager.ts` which include all of the different image API calls we implemented for this app, to include `sketch to image`, `text to image`, `image to image`, and `image to image with mask`. As I mentioned above, we actually started with just using OpenAI's image generation APIs (DALLE), but quickly switched to Stability AI and Clipdrop.

`stabilityApiManager.ts`
```ts
// stabilityApiManager.ts

// // standard imports
// Package for handling Webhooks / POST and other https requests
import axios from "axios";
import FormData from "form-data";

// // custom imports
// constants.ts
import {
  BASE_DELAY,
  MAX_RETRIES,
  ImageRequestType,
  GenerationResponse,
  LoggingObject,
  LoggingEventType,
  LoggingEventProvider,
  LoggingEventStatus,
  NUM_TRAINING_STEPS,
} from "../globals";
import {
  StabilityError,
} from "../errors/stabilityErrors";

/**
 * Class for managing stability.ai api calls
 * @class
 * @classdesc Class for managing stability.ai api calls
 * @hideconstructor
 * @memberof module:apis
 * @category Apis
 */
export class StabilityApiManager {
  private readonly stabilityApiKey: string | undefined;
  private readonly stabilityApiHost: string | undefined;

  /**
   * @constructor
   * @param {string | undefined} stabilityApiKey
   */
  constructor(
    stabilityApiKey: string | undefined
  ) {
    // Initialize the api key for stability.ai
    this.stabilityApiKey = stabilityApiKey;
    this.stabilityApiHost = process.env.STABILITY_API_HOST ?? undefined;
  }

  /**
   * Generates an image from the appropriate Stability.ai api
   * @param {string} uid
   * @param {string} sessionId
   * @param {ImageRequestType} requestType
   * @param {string} caption
   * @param {number} numSamples
   * @param {string | undefined} image
   * @return {Promise<string>}
   * @throws {StabilityError}
   */
  async callStabilityAiApi(
    uid: string,
    sessionId: string,
    requestType: ImageRequestType,
    caption: string,
    numSamples: number,
    image?: string | undefined
  ): Promise<string[]> {
    let imageResponseArray: string[] = [];
    let httpType = 0;
    const userProvidedImage = image ?? "";

    switch (requestType) {
    case ImageRequestType.CREATE: {
      const toLog: LoggingObject = {
        uid: uid,
        session_id: sessionId,
        event_type: LoggingEventType.IMAGE_CREATE,
        event_provider: LoggingEventProvider.STABILITY_AI,
        event_status: LoggingEventStatus.REQUESTED,
        num_steps: NUM_TRAINING_STEPS,
      };
      console.log(JSON.stringify(toLog));
      try {
        ({imageResponseArray, httpType} =
          await this.generateResponseWithStabilityAiTextToImageApi(
            caption,
            numSamples,
            0
          ));
        const toLog: LoggingObject = {
          uid: uid,
          session_id: sessionId,
          event_type: LoggingEventType.IMAGE_CREATE,
          event_provider: LoggingEventProvider.STABILITY_AI,
          event_status: LoggingEventStatus.COMPLETED,
          http_type: httpType,
          num_steps: NUM_TRAINING_STEPS,
        };
        console.log(JSON.stringify(toLog));
      } catch (error: any) {
        // if error, log status, then throw error to populate up the stack
        const toLog: LoggingObject = {
          uid: uid,
          session_id: sessionId,
          event_type: LoggingEventType.IMAGE_CREATE,
          event_provider: LoggingEventProvider.STABILITY_AI,
          event_status: LoggingEventStatus.FAILED,
          http_type: error.code,
          num_steps: NUM_TRAINING_STEPS,
        };
        console.log(JSON.stringify(toLog));
        throw new StabilityError({
          message: error.message,
          code: error.code,
          cause: error.cause,
        });
      }
      break;
    }
    case ImageRequestType.EDIT_WITH_MASK: {
      const toLog: LoggingObject = {
        uid: uid,
        session_id: sessionId,
        event_type: LoggingEventType.IMAGE_EDIT_WITH_MASK,
        event_provider: LoggingEventProvider.STABILITY_AI,
        event_status: LoggingEventStatus.REQUESTED,
        num_steps: NUM_TRAINING_STEPS,
      };
      console.log(JSON.stringify(toLog));
      try {
        ({imageResponseArray, httpType} =
          await this.generateResponseWithStabilityAiImageToImageWithMaskApi(
            caption,
            userProvidedImage,
            numSamples,
            0
          ));
        const toLog: LoggingObject = {
          uid: uid,
          session_id: sessionId,
          event_type: LoggingEventType.IMAGE_EDIT_WITH_MASK,
          event_provider: LoggingEventProvider.STABILITY_AI,
          event_status: LoggingEventStatus.COMPLETED,
          http_type: httpType,
          num_steps: NUM_TRAINING_STEPS,
        };
        console.log(JSON.stringify(toLog));
      } catch (error: any) {
        // if error, log status, then throw error to populate up the stack
        const toLog: LoggingObject = {
          uid: uid,
          session_id: sessionId,
          event_type: LoggingEventType.IMAGE_EDIT_WITH_MASK,
          event_provider: LoggingEventProvider.STABILITY_AI,
          event_status: LoggingEventStatus.FAILED,
          http_type: error.code,
          num_steps: NUM_TRAINING_STEPS,
        };
        console.log(JSON.stringify(toLog));
        throw new StabilityError({
          message: error.message,
          code: error.code,
          cause: error.cause,
        });
      }
      break;
    }
    case ImageRequestType.EDIT: {
      const toLog: LoggingObject = {
        uid: uid,
        session_id: sessionId,
        event_type: LoggingEventType.IMAGE_EDIT,
        event_provider: LoggingEventProvider.STABILITY_AI,
        event_status: LoggingEventStatus.REQUESTED,
        num_steps: NUM_TRAINING_STEPS,
      };
      console.log(JSON.stringify(toLog));
      try {
        ({imageResponseArray, httpType} =
          await this.generateResponseWithStabilityAiImageToImageApi(
            caption,
            userProvidedImage,
            numSamples,
            0
          ));
        const toLog: LoggingObject = {
          uid: uid,
          session_id: sessionId,
          event_type: LoggingEventType.IMAGE_EDIT,
          event_provider: LoggingEventProvider.STABILITY_AI,
          event_status: LoggingEventStatus.COMPLETED,
          http_type: httpType,
          num_steps: NUM_TRAINING_STEPS,
        };
        console.log(JSON.stringify(toLog));
      } catch (error: any) {
        // if error, log status, then throw error to populate up the stack
        const toLog: LoggingObject = {
          uid: uid,
          session_id: sessionId,
          event_type: LoggingEventType.IMAGE_EDIT,
          event_provider: LoggingEventProvider.STABILITY_AI,
          event_status: LoggingEventStatus.FAILED,
          http_type: error.code,
          num_steps: NUM_TRAINING_STEPS,
        };
        console.log(JSON.stringify(toLog));
        throw new StabilityError({
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
   * @param {number} numSamples
   * @param {number} retries
   * @throws {StabilityError}
   * @return {Promise<{string, number}>} A response to send back to user. Calls
   * Stability.AI's text-to-image REST API. In the case of errors we implement
   * exponential backoff until we get a successful response, or until we exceed
   * the number of MAX_RETRIES.
   */
  private async generateResponseWithStabilityAiTextToImageApi(
    prompt: string,
    numSamples: number,
    retries: number
  ): Promise<{imageResponseArray: string[], httpType: number}> {
    const engineId = process.env.STABILITY_DIFFUSION_XL_BETA_ENGINE;

    if (!this.stabilityApiKey) {
      throw new StabilityError({
        message: "Missing Stability API key.",
        code: 401,
        cause: "Missing Stability API key.",
      });
    }

    try {
      const response = await axios.post(
        `${this.stabilityApiHost}/v1/generation/${engineId}/text-to-image`,
        {
          text_prompts: [
            {
              text: prompt,
            },
          ],
          cfg_scale: 7,
          clip_guidance_preset: "FAST_BLUE",
          height: 512,
          width: 512,
          samples: numSamples,
          steps: NUM_TRAINING_STEPS,
        },
        {
          headers: {
            "Content-Type": "application/json",
            "Accept": "application/json",
            "Authorization": `Bearer ${this.stabilityApiKey}`,
          },
        }
      );
      const responseJSON = response.data as GenerationResponse;
      const imageResponse: string[] = [];
      responseJSON.artifacts.forEach((image, index) => {
        imageResponse[index] = image.base64 ?? "";
      });

      return {imageResponseArray: imageResponse, httpType: response.status};
    } catch (error: any) {
      if (retries >= MAX_RETRIES) {
        console.log(`Maximum retries exceeded: ${retries}`);
        console.log("Error cause: ", error?.response?.data?.message);
        throw new StabilityError({
          message: "Non-200 response: maximum retries exceeded.",
          code: error?.response?.status,
          cause: error?.response?.data?.message,
        });
      }
      const delay = BASE_DELAY * 2 ** retries;
      await new Promise((resolve) => setTimeout(resolve, delay));
      console.log(
        `Error: Non-200 response (${error?.response?.status}).
        Retry number: ${retries}`
      );
      return this.generateResponseWithStabilityAiTextToImageApi(
        prompt,
        numSamples,
        retries + 1
      );
    }
  }

  /**
   * @param {string} prompt
   * @param {string} image
   * @param {number} numSamples
   * @param {number} retries
   * @throws {StabilityError}
   * @return {Promise<{string, number}>}
   * A response to send back to the user. Calls
   * Stability.AI's image-to-image REST API. In the case of errors we implement
   * exponential backoff until we get a successful response, or until we exceed
   * the number of MAX_RETRIES.
   */
  private async generateResponseWithStabilityAiImageToImageApi(
    prompt: string,
    image: string,
    numSamples: number,
    retries: number
  ): Promise<{imageResponseArray: string[], httpType: number}> {
    const engineId = process.env.STABILITY_DIFFUSION_XL_BETA_ENGINE;

    if (!this.stabilityApiKey) {
      throw new StabilityError({
        message: "Missing Stability API key.",
        code: 401,
        cause: "Missing Stability API key.",
      });
    }

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

    try {
      const response =
        await axios
          .post(
            this.stabilityApiHost +
            `/v1/generation/${engineId}/image-to-image`,
            formData, {
              headers: {
                ...formData.getHeaders(),
                "Accept": "application/json",
                "Authorization": `Bearer ${this.stabilityApiKey}`,
              },
            });

      const responseJSON = response.data as GenerationResponse;
      const imageResponse: string[] = [];
      responseJSON.artifacts.forEach((image, index) => {
        imageResponse[index] = image.base64 ?? "";
      });

      return {imageResponseArray: imageResponse, httpType: response.status};
    } catch (error: any) {
      if (retries >= MAX_RETRIES) {
        console.log(`Maximum retries exceeded: ${retries}`);
        console.log("Error cause: ", error?.response?.data?.message);
        throw new StabilityError({
          message: "Non-200 response: maximum retries exceeded.",
          code: error?.response?.status,
          cause: error?.response?.data?.message,
        });
      }
      const delay = BASE_DELAY * 2 ** retries;
      await new Promise((resolve) => setTimeout(resolve, delay));
      console.log(
        `Error: Non-200 response (${error?.response?.status}).
        Retry number: ${retries}`
      );
      return this.generateResponseWithStabilityAiImageToImageApi(
        prompt,
        image,
        numSamples,
        retries + 1
      );
    }
  }

  /**
   * @param {string} prompt
   * @param {string} image
   * @param {number} numSamples
   * @param {number} retries
   * @throws {StabilityError}
   * @return {Promise<{string, number}>} A response to send back to the user.
   * Calls Stability.AI"s image-to-image w/ mask REST API.
   * In the case of errors we implement exponential backoff until
   * we get a successful response, or until we exceed
   * the number of MAX_RETRIES.
   */
  private async generateResponseWithStabilityAiImageToImageWithMaskApi(
    prompt: string,
    image: string,
    numSamples: number,
    retries: number
  ): Promise<{imageResponseArray: string[], httpType: number}> {
    const engineId = process.env.STABILITY_DIFFUSION_XL_BETA_ENGINE;

    if (!this.stabilityApiKey) {
      throw new StabilityError({
        message: "Missing Stability API key.",
        code: 401,
        cause: "Missing Stability API key.",
      });
    }
    const imageBuffer = Buffer.from(image, "base64");
    const formData = new FormData();
    formData.append("init_image", imageBuffer);
    formData.append("mask_source", "INIT_IMAGE_ALPHA");
    formData.append("text_prompts[0][text]", prompt);
    formData.append("cfg_scale", "7");
    formData.append("clip_guidance_preset", "FAST_BLUE");
    formData.append("samples", numSamples);
    formData.append("steps", NUM_TRAINING_STEPS);

    try {
      const response =
        await axios
          .post(
            this.stabilityApiHost +
            `/v1/generation/${engineId}/image-to-image/masking`,
            formData, {
              headers: {
                ...formData.getHeaders(),
                "Accept": "application/json",
                "Authorization": `Bearer ${this.stabilityApiKey}`,
              },
            });

      const responseJSON = response.data as GenerationResponse;
      const imageResponse: string[] = [];
      responseJSON.artifacts.forEach((image, index) => {
        imageResponse[index] = image.base64 ?? "";
      });

      return {imageResponseArray: imageResponse, httpType: response.status};
    } catch (error: any) {
      if (retries >= MAX_RETRIES) {
        console.log(`Maximum retries exceeded: ${retries}`);
        console.log("Error cause: ", error?.response?.data?.message);
        throw new StabilityError({
          message: "Non-200 response: maximum retries exceeded.",
          code: error?.response?.status,
          cause: error?.response?.data?.message,
        });
      }
      const delay = BASE_DELAY * 2 ** retries;
      await new Promise((resolve) => setTimeout(resolve, delay));
      console.log(
        `Error: Non-200 response (${error?.response?.status}).
        Retry number: ${retries}`
      );
      return this.generateResponseWithStabilityAiImageToImageWithMaskApi(
        prompt,
        image,
        numSamples,
        retries + 1
      );
    }
  }
} // end `StabilityApiManager` class
```
`clipdropApiManager.ts`
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
    // const imageBuffer = Buffer.from(image, "base64");
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

### Text to Image (Stability AI)

The text to image API was the most popular functionality in the app. In it, users could simply pass a text `prompt`, and we would return an `image` back to the client, which could then be saved directly as a photo or sticker on the device. To make a request, our backend code simply sends an HTTPS POST request to `https://api.stability.ai/v1/generation/${engineId}/text-to-image`. For this API, we used `stable-diffusion-xl-beta-v2-2-2` for the `engineId`. This API was very simple to use, and the documentation was very clear and helpful.

Below you can see an example of the functionality in action. The user generated an image with the prompt "A cool sun wearing blue sunglasses," and the resultant image is shown below. This image can then easily be shared with friends directly through iMessage, or by converting into a sticker.

![alt](/assets/img/aimessages/exampleExtension.png){: w="300" h="600" .normal .shadow}

And here's the full `generateResponseWithStabilityAiTextToImageApi` function we used to call the API and convert a user's `prompt` into a new `image`:

```ts
// stabilityApiManager.ts

/**
 * @param {string} prompt
 * @param {number} numSamples
 * @param {number} retries
 * @throws {StabilityError}
 * @return {Promise<{string, number}>} A response to send back to user. Calls
 * Stability.AI's text-to-image REST API. In the case of errors we implement
 * exponential backoff until we get a successful response, or until we exceed
 * the number of MAX_RETRIES.
 */
private async generateResponseWithStabilityAiTextToImageApi(
  prompt: string,
  numSamples: number,
  retries: number
): Promise<{imageResponseArray: string[], httpType: number}> {
  const engineId = process.env.STABILITY_DIFFUSION_XL_BETA_ENGINE;

  if (!this.stabilityApiKey) {
    throw new StabilityError({
      message: "Missing Stability API key.",
      code: 401,
      cause: "Missing Stability API key.",
    });
  }

  try {
    const response = await axios.post(
      `${this.stabilityApiHost}/v1/generation/${engineId}/text-to-image`,
      {
        text_prompts: [
          {
            text: prompt,
          },
        ],
        cfg_scale: 7,
        clip_guidance_preset: "FAST_BLUE",
        height: 512,
        width: 512,
        samples: numSamples,
        steps: NUM_TRAINING_STEPS,
      },
      {
        headers: {
          "Content-Type": "application/json",
          "Accept": "application/json",
          "Authorization": `Bearer ${this.stabilityApiKey}`,
        },
      }
    );
    const responseJSON = response.data as GenerationResponse;
    const imageResponse: string[] = [];
    responseJSON.artifacts.forEach((image, index) => {
      imageResponse[index] = image.base64 ?? "";
    });

    return {imageResponseArray: imageResponse, httpType: response.status};
  } catch (error: any) {
    if (retries >= MAX_RETRIES) {
      console.log(`Maximum retries exceeded: ${retries}`);
      console.log("Error cause: ", error?.response?.data?.message);
      throw new StabilityError({
        message: "Non-200 response: maximum retries exceeded.",
        code: error?.response?.status,
        cause: error?.response?.data?.message,
      });
    }
    const delay = BASE_DELAY * 2 ** retries;
    await new Promise((resolve) => setTimeout(resolve, delay));
    console.log(
      `Error: Non-200 response (${error?.response?.status}).
      Retry number: ${retries}`
    );
    return this.generateResponseWithStabilityAiTextToImageApi(
      prompt,
      numSamples,
      retries + 1
    );
  }
}
```

### Image to Image (Stability AI)

For the image to image functionality, users could pass a text `prompt` and an `image` to be altered, and we would return a new image back to the client, which could then be saved directly as an image or sticker on the iPhone/iPad. To make a request, our backend code simply sends an HTTPS POST request to `https://api.stability.ai/v1/generation/${engineId}/image-to-image`. For this API, we used `stable-diffusion-xl-beta-v2-2-2` for the `engineId`.

Here's the full `generateResponseWithStabilityAiImageToImageApi` function we used to call the API and convert a user's `image` and `prompt` into a new `image`:

```ts
// stabilityApiManager.ts

/**
 * @param {string} prompt
 * @param {string} image
 * @param {number} numSamples
 * @param {number} retries
 * @throws {StabilityError}
 * @return {Promise<{string, number}>}
 * A response to send back to the user. Calls
 * Stability.AI's image-to-image REST API. In the case of errors we implement
 * exponential backoff until we get a successful response, or until we exceed
 * the number of MAX_RETRIES.
 */
private async generateResponseWithStabilityAiImageToImageApi(
  prompt: string,
  image: string,
  numSamples: number,
  retries: number
): Promise<{imageResponseArray: string[], httpType: number}> {
  const engineId = process.env.STABILITY_DIFFUSION_XL_BETA_ENGINE;

  if (!this.stabilityApiKey) {
    throw new StabilityError({
      message: "Missing Stability API key.",
      code: 401,
      cause: "Missing Stability API key.",
    });
  }

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

  try {
    const response =
      await axios
        .post(
          this.stabilityApiHost +
          `/v1/generation/${engineId}/image-to-image`,
          formData, {
            headers: {
              ...formData.getHeaders(),
              "Accept": "application/json",
              "Authorization": `Bearer ${this.stabilityApiKey}`,
            },
          });

    const responseJSON = response.data as GenerationResponse;
    const imageResponse: string[] = [];
    responseJSON.artifacts.forEach((image, index) => {
      imageResponse[index] = image.base64 ?? "";
    });

    return {imageResponseArray: imageResponse, httpType: response.status};
  } catch (error: any) {
    if (retries >= MAX_RETRIES) {
      console.log(`Maximum retries exceeded: ${retries}`);
      console.log("Error cause: ", error?.response?.data?.message);
      throw new StabilityError({
        message: "Non-200 response: maximum retries exceeded.",
        code: error?.response?.status,
        cause: error?.response?.data?.message,
      });
    }
    const delay = BASE_DELAY * 2 ** retries;
    await new Promise((resolve) => setTimeout(resolve, delay));
    console.log(
      `Error: Non-200 response (${error?.response?.status}).
      Retry number: ${retries}`
    );
    return this.generateResponseWithStabilityAiImageToImageApi(
      prompt,
      image,
      numSamples,
      retries + 1
    );
  }
}
```

### Image to Image with Mask (Stability AI)

The image to image with mask API allows you to selectively modify portions of an image using a mask, where the mask is the same size and shape as the initial image. To use it, users draw / shade the portion of the image they want to modify, and only that portion of the image will be updated. Along with the `image` parameter, users also pass in a `prompt`. To make a request, our backend code simply sends an HTTPS POST request to `https://api.stability.ai/v1/generation/${engineId}/image-to-image/masking`. For this API, we used `stable-diffusion-xl-beta-v2-2-2` for the `engineId`.

Below you can see an example of the functionality in action. The image on the left was first generated based on a text prompt from the user for "A cute cartoon dog." Next, the user masked a small area on the dog's head along with a prompt to add a top hat, and the new image on the right was returned.

Initial Image | After Applying Mask
- | -
![alt](/assets/img/aimessages/image_mask_before.png){: .shadow } | ![alt](/assets/img/aimessages/image_mask_after.png){: .shadow }

And here's the full `generateResponseWithStabilityAiImageToImageWithMaskApi` function we used to call the API and convert a user's `image` and `prompt` into a new `image`:

```ts
// stabilityApiManager.ts

/**
 * @param {string} prompt
 * @param {string} image
 * @param {number} numSamples
 * @param {number} retries
 * @throws {StabilityError}
 * @return {Promise<{string, number}>} A response to send back to the user.
 * Calls Stability.AI"s image-to-image w/ mask REST API.
 * In the case of errors we implement exponential backoff until
 * we get a successful response, or until we exceed
 * the number of MAX_RETRIES.
 */
private async generateResponseWithStabilityAiImageToImageWithMaskApi(
  prompt: string,
  image: string,
  numSamples: number,
  retries: number
): Promise<{imageResponseArray: string[], httpType: number}> {
  const engineId = process.env.STABILITY_DIFFUSION_XL_BETA_ENGINE;

  if (!this.stabilityApiKey) {
    throw new StabilityError({
      message: "Missing Stability API key.",
      code: 401,
      cause: "Missing Stability API key.",
    });
  }
  const imageBuffer = Buffer.from(image, "base64");
  const formData = new FormData();
  formData.append("init_image", imageBuffer);
  formData.append("mask_source", "INIT_IMAGE_ALPHA");
  formData.append("text_prompts[0][text]", prompt);
  formData.append("cfg_scale", "7");
  formData.append("clip_guidance_preset", "FAST_BLUE");
  formData.append("samples", numSamples);
  formData.append("steps", NUM_TRAINING_STEPS);

  try {
    const response =
      await axios
        .post(
          this.stabilityApiHost +
          `/v1/generation/${engineId}/image-to-image/masking`,
          formData, {
            headers: {
              ...formData.getHeaders(),
              "Accept": "application/json",
              "Authorization": `Bearer ${this.stabilityApiKey}`,
            },
          });

    const responseJSON = response.data as GenerationResponse;
    const imageResponse: string[] = [];
    responseJSON.artifacts.forEach((image, index) => {
      imageResponse[index] = image.base64 ?? "";
    });

    return {imageResponseArray: imageResponse, httpType: response.status};
  } catch (error: any) {
    if (retries >= MAX_RETRIES) {
      console.log(`Maximum retries exceeded: ${retries}`);
      console.log("Error cause: ", error?.response?.data?.message);
      throw new StabilityError({
        message: "Non-200 response: maximum retries exceeded.",
        code: error?.response?.status,
        cause: error?.response?.data?.message,
      });
    }
    const delay = BASE_DELAY * 2 ** retries;
    await new Promise((resolve) => setTimeout(resolve, delay));
    console.log(
      `Error: Non-200 response (${error?.response?.status}).
      Retry number: ${retries}`
    );
    return this.generateResponseWithStabilityAiImageToImageWithMaskApi(
      prompt,
      image,
      numSamples,
      retries + 1
    );
  }
}
```

### Sketch to Image (Clipdrop)

The sketch to image API endpoint allows you to generate an image based on a `sketch` and a `prompt` describing what you expect the doodle to turn in to. The request made is an HTTPS POST request to `https://clipdrop-api.co/sketch-to-image/v1/sketch-to-image` and its body is in `form-data` format. This API was very simple to use, and the documentation was very clear and helpful.

Below you can see some examples of the functionality in action. First, a doodle of a city skyline with the prompt "A nighttime city skyline," and next to it, the resultant image. The third picture is a sketch of a "Green forest starry night," and next to it, we have the resultant, generated image.

Skyline Doodle | Resultant Image | Forest Doodle | Resultant Image
- | - | - | -
![alt](/assets/img/aimessages/skyline_before.png){: .shadow } | ![alt](/assets/img/aimessages/skyline_after.png){: .shadow } | ![alt](/assets/img/aimessages/forest_before.png){: .shadow } | ![alt](/assets/img/aimessages/forest_after.png){: .shadow }

And here's the full `generateResponseWithClipdropSketchToImageApi` function we used to call the API and convert a user's `sketch` + `prompt` into a new `image`:

```ts
// clipdropApiManager.ts

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
  // const imageBuffer = Buffer.from(image, "base64");
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
```

## Takeaways

Overall, both the Stability AI and Clipdrop APIs were extremely easy to use, and provided great performance. They were both very start-up / developer friendly, offering a generous amount of free credits to get started, and the actual cost per image was very low too. Throughout the development pf our app (and continuing to today), Stability AI was and is continually improving the stable diffusion model to generate the images, and the image generation is continuing to be faster, produce better images, and cost less money. I highly recommend both providers.  

Thanks for reading! In part 6 I plan to cover our use of Firebase in more detail. 
