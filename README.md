# Dining Concierge Chatbot

A serverless, microservice-based chatbot that recommends restaurants based on a short conversation with the user (location, cuisine, dining time, party size, and phone number). It was originally built as a project for the Cloud Computing and Big Data course at Columbia University, using Amazon Lex, API Gateway, Lambda, SQS, DynamoDB, Elasticsearch, and SNS.

This README documents the original architecture and setup steps, and also calls out what has changed on the AWS side since the project was first built, so it can be rebuilt on the current AWS services if you want to run it today.

## Demo (from the original build)

![demo](https://user-images.githubusercontent.com/20079387/77853640-0da19780-71b3-11ea-8bc7-f7a2539ccef5.gif)

After collecting the user's preferences, the chatbot follows up with a text message containing three restaurant suggestions:

![sms](https://user-images.githubusercontent.com/20079387/77853920-fa8fc700-71b4-11ea-85ce-d40a794cdeca.jpg)

## Architecture

![architecture](https://user-images.githubusercontent.com/20079387/77852560-49d1f980-71ad-11ea-8117-c46eebf6713f.png)

The system is split into two decoupled paths so the chat stays fast while the actual restaurant lookup happens in the background:

**Conversation path (synchronous):** browser -> API Gateway -> LF0 -> Amazon Lex -> LF1 -> SQS. The user gets an immediate acknowledgement, not the restaurant list itself.

**Fulfillment path (asynchronous):** SQS -> LF2 (on a schedule) -> Elasticsearch/OpenSearch + DynamoDB -> SNS -> the user's phone. A minute or so later, the user gets a text with actual suggestions.

Components:

| Component | File | Role |
|---|---|---|
| Front end | `front-end/` | Static chat widget (HTML/CSS/JS) hosted on S3, calls the API through a generated JS SDK |
| API | `swagger.yaml` | OpenAPI spec for a single `POST /chatbot` endpoint, imported into API Gateway |
| LF0 | `LF0.py` | API Gateway's Lambda integration. Forwards the user's text to Lex and returns Lex's reply |
| Lex bot | not in this repo, built in the AWS console | Holds the conversation state machine and three intents: `GreetingIntent`, `ThankYouIntent`, `DiningSuggestionsIntent` |
| LF1 | `LF1.py` | Lex code hook. Once all slots are filled, pushes the collected preferences to an SQS queue and closes the conversation |
| scraper.py | `scraper.py` | One-off script that pulls restaurant data from the Yelp Fusion API and stores it in DynamoDB |
| LF2 | `LF2.py` | Queue worker, run on a schedule. Pulls a request off SQS, looks up a matching restaurant by cuisine in Elasticsearch/OpenSearch, fetches the full record from DynamoDB, and texts the user via SNS |

## Repository layout

```
aws_restaurant_chatbot-master/
  front-end/        Static chat UI and the generated API Gateway JS SDK
  LF0.py             Lambda: API Gateway <-> Lex bridge
  LF1.py             Lambda: Lex code hook for DiningSuggestionsIntent
  LF2.py             Lambda: SQS queue worker, fulfillment and SMS
  scraper.py          Yelp Fusion scraper and DynamoDB loader
  swagger.yaml        OpenAPI definition for the chatbot API
```

## Building it on AWS

These steps describe the original design. Where the AWS service has since changed name or version, both the original and current names are given, see "What has changed since this was built" below for details.

### 1. Front end

1. Open `front-end/chatbot.html`, `chatbot.css`, and `chatbot.js` and adjust branding as you like.
2. Create an S3 bucket, enable static website hosting on it, and upload the contents of `front-end/`.
3. The chat UI calls the API through the generated `apigClient.js` SDK described in `front-end/README.md`. You will regenerate this file after the API is deployed (step 2.4 below).

### 2. API Gateway

1. Create a REST API in API Gateway.
2. Import `swagger.yaml` to create the `POST /chatbot` resource and method.
3. Enable CORS on the method.
4. Deploy the API, then generate a JavaScript SDK for it and drop the generated files into `front-end/` in place of the placeholders, or wire up your own fetch/axios call using the deployed endpoint URL and, if you added one, an API key.
5. Point the Lambda integration at LF0 (see step 3). Do not use Lambda proxy integration, LF0 and `swagger.yaml` both assume the older, non-proxy request and response shape.

### 3. LF0: the API-to-Lex bridge

1. Create a Lambda function from `LF0.py`, Python runtime.
2. Give it an execution role with Lex runtime access (see the note at the top of the file) and CloudWatch Logs access.
3. Set `AMAZON_LEX_BOT` and `LEX_BOT_ALIAS` to match the bot you create in the next step.
4. Attach it as the integration behind `POST /chatbot`.

### 4. Amazon Lex bot

1. Create a bot with three intents: `GreetingIntent`, `ThankYouIntent`, and `DiningSuggestionsIntent`.
2. For `DiningSuggestionsIntent`, define slots for Location, Cuisine, Dining Time, Number of People, and Phone Number, and let Lex prompt the user for each one it doesn't already have.
3. Attach `LF1.py` as the Lambda code hook for intent fulfillment.
4. Publish an alias for the bot and note the bot name and alias, they go into LF0's environment variables above.

### 5. LF1: the code hook

1. Create a Lambda function from `LF1.py`.
2. Create an SQS queue (for example `RestaurantRequest`) and set `QUEUE_URL` in the file to its URL.
3. Give the function's execution role permission to send messages to that queue.
4. Attach the function as the code hook for `DiningSuggestionsIntent` in the Lex console.

### 6. Restaurant data: scraper and DynamoDB

1. Create a DynamoDB table (the code refers to it as `YelpRestaurants`) with `id` as the partition key.
2. Get a Yelp Fusion API key and put it somewhere other than the source file, an environment variable is enough. Do not commit a real key.
3. Run `scraper.py` locally or as a one-off job. It pages through the Yelp API for a list of cuisines and writes each restaurant into DynamoDB, tagging each item with `insertedAtTimestamp` and merging `cusine_types` if the restaurant already exists.

### 7. Search index

1. Create a search domain and an index called `restaurants`.
2. For each restaurant in DynamoDB, index just the restaurant `id` and its cuisine into that index. This keeps the search layer small since DynamoDB is still the source of truth for full restaurant details.

### 8. LF2: the queue worker and SNS

1. Create a Lambda function from `LF2.py`.
2. Set `TABLENAME` to your DynamoDB table name, and `ELASTIC_SEARCH_URL`/`es_host` to your search domain's endpoint.
3. Give the function's execution role permission to read from SQS, query the search domain, read from DynamoDB, and publish to SNS.
4. Configure the function as an SQS trigger (or, as originally designed, run it on a fixed schedule so it polls the queue itself) so it's invoked as requests arrive.
5. Verify or register the destination phone numbers with SNS if your account is still in the SNS SMS sandbox.

## Configuration placeholders

Before anything will run, replace these values (none of them should be committed with real data):

| File | Placeholder | What it is |
|---|---|---|
| `LF1.py` | `QUEUE_URL` | SQS queue URL |
| `LF2.py` | `ELASTIC_SEARCH_URL`, `es_host` | Search domain endpoint |
| `LF2.py` | default phone number in `send_sns_message` | Only a fallback, pass a real number in the SQS message body |
| `scraper.py` | `MY_API_KEY` | Yelp Fusion API key |

`LF2.py` also needs the `aws_requests_auth` and `requests` packages bundled into its deployment package, since neither ships with the default Lambda Python runtime.

## What has changed since this was built

This project reflects AWS as it looked around 2019 to 2020. Several of the services it depends on have since been renamed, replaced, or retired. If you're rebuilding this today, here is what's different:

- **Amazon Lex V1 is retired.** AWS stopped letting you create new V1 bot resources on March 31, 2025, and fully discontinued V1 on September 15, 2025. `LF0.py` uses the V1 `lex-runtime` client and `post_text`. A current build needs a Lex V2 bot and the `lexv2-runtime` client's `recognize_text` call instead, along with the corresponding V2 console workflow for defining intents and slots. AWS's own V1-to-V2 migration tooling and guide can help translate an existing V1 bot definition.
- **Amazon Elasticsearch Service is now Amazon OpenSearch Service.** AWS renamed the managed search service in September 2021 after the Elasticsearch licensing dispute, and new domains are created as OpenSearch domains. The request/response shape used in `LF2.py` still works against OpenSearch, but you'll create an OpenSearch domain rather than an "Elasticsearch Service" domain, and the console and IAM policy names have changed accordingly.
- **CloudWatch Events is now part of Amazon EventBridge.** The original design polls the queue by triggering LF2 from a CloudWatch Events schedule rule. EventBridge is the current name for that same rules engine, and its scheduled-rule feature (or the newer EventBridge Scheduler) is the direct replacement. Simpler still, current designs typically skip polling entirely and configure the SQS queue itself as LF2's event source trigger, which is the approach called out as an alternative in the LF2 setup step above.
- **The Lambda Python 3.6 runtime is long past end of support.** `LF1.py`'s header comment references Python 3.6. Current Lambda Python runtimes are considerably newer; there is no reason to target 3.6 today, and AWS will not let you create a new function on it.
- **The generated API Gateway JavaScript SDK approach is dated.** The front end's SigV4 signing setup (`apigClient.js` plus the `lib/` folder of third-party crypto helpers) reflects how API Gateway's SDK generator worked at the time. It still functions, but a current build would more commonly call the API with `fetch` directly, optionally behind an API key or a Cognito authorizer, rather than shipping a hand-rolled SigV4 signing stack in the browser.
- **Treat the Yelp API key and phone numbers as secrets.** `scraper.py` originally hardcoded the Yelp key as a string literal. Load it from an environment variable or a secrets manager instead, and never commit a real key.

## Known limitations (carried over from the original design)

- Suggestions are random restaurants matching the requested cuisine, not a ranked recommendation. Building an actual recommender was explicitly out of scope for the original assignment.
- Filtering only considers cuisine. Location and neighborhood collected during the conversation are not used to filter results.
- Some fields will be missing on some restaurants since Yelp doesn't return a uniform schema for every business, which is part of why DynamoDB (schemaless per item) was chosen over a relational store for this data.
