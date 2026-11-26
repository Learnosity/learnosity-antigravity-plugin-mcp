---
name: install-and-set-up-the-sdk
description: Guide a developer to install and configure the Learnosity SDK in their chosen server-side language (PHP, Python, Node.js, Ruby, Java, or .NET/C#) so they can generate signed requests and make Data API calls.
---

# Install and Set Up the Learnosity SDK

**Goal:** Get the Learnosity SDK installed and configured in your chosen
server-side language so you can generate signed requests and make Data API
calls.

## How to use the SDKs

1. **Pick the right SDK.** Official SDKs are available for six languages:
   **PHP**, **Python**, **Node.js**, **Ruby**, **Java**, and **.NET/C#**. Choose
   the one that matches the server-side language the project already uses.
2. **Read that SDK's README for install and setup steps.** Each SDK's README
   contains the authoritative, up-to-date install instructions and a
   setup/verify example. Call the `read_document` tool with `source: "github"`
   and the `article_id` for the matching language from the table below, then
   follow the install and quickstart steps it describes.
3. **Sign a request on your server.** Following the README, call the SDK's
   `Init(...)` helper with the service name, a security object, your
   `consumer_secret`, and a request object, then call `generate()`. This returns
   a signed init object (a JSON blob).
4. **Load an API in the browser.** Pass the signed init object to the matching
   Learnosity JavaScript API.
5. **Optionally make Data API calls.** Most SDKs provide helper functions to
   make server-side Data API requests simpler.

> **Note:** The Node.js SDK includes a Data API routing layer (`DataApi`) in
> addition to request signing. If you prefer, you can still sign the request
> with the SDK and send it using any server-side HTTP library.

## SDK README lookup

Detect the project's server-side language, then call `read_document` with
`source: "github"` and the matching `article_id`:

| Language  | `article_id`          | README title                                  |
| --------- | --------------------- | --------------------------------------------- |
| PHP       | `2408634164751404355` | `Learnosity/learnosity-sdk-php/README.md`     |
| Python    | `6247850392671961384` | `Learnosity/learnosity-sdk-python/README.md`  |
| Node.js   | `4551592665427693190` | `Learnosity/learnosity-sdk-nodejs/README.md`  |
| Ruby      | `2800851377229186036` | `Learnosity/learnosity-sdk-ruby/README.md`    |
| Java      | `6202924830261288759` | `Learnosity/learnosity-sdk-java/README.md`    |
| .NET / C# | `2197070393494315675` | `Learnosity/learnosity-sdk-asp.net/readme.md` |

> **Note:** The Java SDK is not yet published to Maven Central. For now it must
> be installed from source (git clone or the release ZIP, then built with
> Maven); its README covers this.

## Credentials and security

Learnosity's README examples use **placeholder** credentials:

- `consumer_key`: `YOUR_CONSUMER_KEY`
- `consumer_secret`: `YOUR_CONSUMER_SECRET`

When Learnosity provisions your own account, replace these with your own key and
secret. Your `consumer_secret` is a private credential: keep it in a secured
credential store and **never** commit it to version control or expose it to the
browser.

## Next steps

Once the SDK is installed and generating signed requests, you're ready to build
a full integration. See the **Build a Complete Assessment Platform** skill
(`golden-path-author-items-reports`) for the recommended Author API, Items API,
and Reports API flow.
