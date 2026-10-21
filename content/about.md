+++
title = 'About'
+++

This is the blog of [tomato](https://github.com/tomatool/tomato), a tool for
testing services as black boxes: it starts a service's real dependencies in
containers, stands in for the services it calls, and runs Gherkin scenarios
against it over HTTP, gRPC, Kafka, RabbitMQ, WebSockets, S3 and more.

The posts are about the practice around that: how to write a suite worth
trusting, how to run it in CI, and how to tell what it does and doesn't test.

- Documentation: [tomatool.github.io/tomato](https://tomatool.github.io/tomato/)
- Source: [github.com/tomatool/tomato](https://github.com/tomatool/tomato)
- Install: `brew install tomatool/tap/tomato`, or
  `curl -fsSL https://raw.githubusercontent.com/tomatool/tomato/main/install.sh | sh`
