---
layout: post
title: "How to verify AWS SNS message with Clojure"
categories: 
tags: Clojure AWS SNS
excerpt: AWS Simple Notifications Services (SNS) delivers messages to multiple recipients  including HTTP and HTTPS subscribers. Endpoint should verify message origin before it is processed. Let's see how message validation can be implemented in Clojure. 
---

## What is the problem and how it can be solved

Endpoint code receives the message and before it is processed, our code has to decide if message could be trusted. We want to process only legit events and filter out any spam or malicious requests. Fortunately AWS signs messages and clients can reliably verify messages origin.

andMessage Signature Verification is [document by AWS](https://docs.aws.amazon.com/sns/latest/dg/SendMessageToHttp.verify.signature.html) and it is quite a complicated process. Work below is to be done for every request:

* Read the message content as key/value pairs. Determine the message type and concatenate these keys and values using rules unique for every type. Resulting string will be used later to generate message signature.
* Download and optionally cache certificate using url from  `SigningCertURL` field.
* Validate the authenticity of the certificate. Validation includes checks for certificate  schema and endpoint.
* Extract the public key from the certificate.
* Generate hash for message content using certificate public key.
* Grab message `Signature` field and decode it from Base64 format.
* Compare the derived hash value to the asserted hash value. If the values are identical, then the receiver is assured that the message has not been modified while in transit and the message must have originated from Amazon SNS.

## How to verify Message in Clojure

Let's start with bad news. As of 2019 there is no native Clojure library **yet** to verify SNS Signature. Fortunately there is a good news too. AWS provides Java SDK that covers work with SNS including message validation. We van use that SDK from our Clojure project.

For this sample we are going to use [Leiningen](https://leiningen.org/) and [ring](https://github.com/ring-clojure/ring). At first we need to add AWS SDK dependency to `project.clj` file:

```clojure
:dependencies [[com.amazonaws/aws-java-sdk-sns "1.11.641"]]
```

In our handler we can use `SnsMessageManager` class that has `parseMessage()` method to read and validate SNS message from io stream.

```clojure
(:import (com.amazonaws.services.sns.message SnsMessageManager))
```

Create an instance of SnsMessageManager that can be reused for multiple calls. `manager` is bounded to AWS Region.

Region can be provided through constructor. Another option is to use constructor without parameters. In that case SnsMessageManager gets region (if it is available) from runtime context.

```clojure
(def manager (new SnsMessageManager "us-east-1"))
```

`SnsMessageManager.parseMessage()` loads SnsMessage from io stream. Method .parseMessage() combines two actions, it parses message content and validates message signature. It does not return validation status as a result, instead  it throws `com.amazonaws.SdkClientException` error if validation failed.

In code below we load only several fields from base object `SnsMessage`. Rest of the fields can be loaded if we use actual message type like [SnsNotification](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/sns/message/SnsNotification.html) or [SnsSubscriptionConfirmation](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/sns/message/SnsSubscriptionConfirmation.html)

```clojure
(defn parse-message
  "Loads message from request body stream. Throws if message is not valid."
  [body-stream]
  (let [message (.parseMessage manager body-stream)]
    {:topic-arn (.getTopicArn message)
     :message-id (.getMessageId message)
     :timestamp (.getTimestamp message)}))
```

`parse-message` expects request body as input parameter. This HTTP handler takes request body and calls function to process it.

```clojure
(defn sns-handler
  "Returns 200 and Parsed message fields for valid SNS Message."
  [request]
  {:status 200
   :headers {"Content-Type" "text/html"}
   :body (parse-message (:body request))})
```

Important. `parse-message` reads body stream and after it is used body cannot be read second time.

Now we have all pieces together to receive, validate and parse SNS messages in HTTP(S) handler. Working code and tests available on [github](https://github.com/dharnitski/sns-verify-clj).

Happy Coding!
