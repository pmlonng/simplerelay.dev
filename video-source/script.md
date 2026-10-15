# Simple Relay: Reliable HTTP delivery

## hook
When a webhook arrives during a deploy, your application may not be ready. [#deploy] The sender still needs a quick, dependable response. [#response]

## setup
Run Simple Relay as one container. [#container] There is no broker to install and no separate database to provision. [#no-broker] Embedded SQLite is ready to store requests. [#sqlite]

## ack
Relay can verify the signature, when configured. [#verify] It saves the original payload before sending a fast acknowledgement. [#persist] The sender does not wait for your application to recover. [#ack]

## retry
Background workers deliver the request. [#worker] If your application is temporarily unavailable, Relay retries with a bounded backoff. [#retry] When it recovers, delivery continues. [#delivered]

## recovery
If all attempts fail, the request moves to the {DLQ|D L Q}, where failed deliveries stay visible. [#dlq] Inspect the failure. [#inspect] Replay it when you are ready. [#replay]

## close
Start with one command and be up in about two minutes. [#quickstart] Simple Relay makes HTTP delivery more reliable, without a broker to run. [#summary] Read the Quickstart to try it.
