# Picus

[![status-badge](https://github-ci.fonona.net/api/badges/windsource/picus/status.svg)](https://github-ci.fonona.net/windsource/picus)

Picus connects to a [Woodpecker CI](https://woodpecker-ci.org) server and
creates/starts an agent in the cloud when there are pending jobs. The
agent will be shutdown/stopped owhen there are no more build jobs for a specific time in
order to reduce cloud costs.
Picus polls the Woodpecker API and creates/starts an agent in the cloud when it is
required. Currently [Hetzner cloud](https://www.hetzner.com/cloud) and
[AWS](https://aws.amazon.com) are supported.

## Usage

Picus provides a container image which can be used with `docker-compose` to
start the service:

```yml
# docker-compose.yml
version: '3'

services:
  picus:
    # Better replace latest with specific version in next line
    image: ghcr.io/windsource/picus:latest
    restart: always
    environment:
      - PICUS_WOODPECKER_SERVER=https://woodpecker.moengage.com
      - PICUS_WOODPECKER_TOKEN=<WOODPECKER-PERSONAL-ACCESS-TOKEN>
      - PICUS_AGENT_WOODPECKER_SERVER=woodpecker.moeinternal.com:9000
      - PICUS_AGENT_WOODPECKER_AGENT_SECRET=<...>
      - PICUS_AGENT_WOODPECKER_BACKEND=docker
      - PICUS_PROVIDER_TYPE=aws
      - AWS_ACCESS_KEY_ID=AKIAXXXXXXXXXXX
      - AWS_REGION=<AWS-REGION>
      - AWS_SECRET_ACCESS_KEY=XXXXXXXXXXXXXXXXXX
      - PICUS_AWS_INSTANCE_ID=i-12345678abcde
```

The following environment variables can or have to be used independent of
the specific provider:

Name | Description | Default
---- | ----------- | -------
`PICUS_WOODPECKER_SERVER` | URL to the Woodpecker host like `https://woodpecker.example.com` | -
`PICUS_WOODPECKER_TOKEN` | Personal token to Woodpecker | -
`PICUS_POLL_INTERVAL` | Interval in which Picus will poll the Woodpecker API `/api/queue/info`.  For format see [go_parse_duration](https://docs.rs/go-parse-duration/latest/go_parse_duration/). | 10s
`PICUS_MAX_IDLE_TIME` | Duration to wait after the last running job before shutting down or stopping an agent. For format see [go_parse_duration](https://docs.rs/go-parse-duration/latest/go_parse_duration/). | 30m
`PICUS_PROVIDER_TYPE` | Type of cloud provider to use. Valid values are `hcloud` and `aws` | -

### AWS

When using AWS as cloud provider Picus requires an existing EC2 instance
which already has a Woodpecker agent installed. Picus will then start and
stop the instance when required. AWS does not charge for stopped EC2 
instances but only for the EBS volumes that are attached to it 
(see [Amazon EC2 On-Demand Pricing](https://aws.amazon.com/ec2/pricing/on-demand)).

To setup such an instance the cloudformation template
[`aws_agent_example.yml`](aws_agent_example.yml) could be used.

Setup this AWS policy to provide Picus access to the required resources:

```yaml
Version: 2012-10-17
Statement:
  - Effect: Allow
    Action:
      - ec2:StartInstances
      - ec2:StopInstances
    Resource: arn:aws:ec2:*:*:instance/*
  - Effect: Allow
    Action: ec2:DescribeInstances
    Resource: "*"
```

These environment variables exists for AWS:

Name | Description | Default
---- | ----------- | -------
`AWS_ACCESS_KEY_ID` | AWS access key | -
`AWS_SECRET_ACCESS_KEY` | The secret key associated with the access key | -
`AWS_REGION` | The AWS region in which the EC2 instance has been deployed | -
`PICUS_AWS_INSTANCE_ID` | The EC2 instance ID (for example, `i-1234567890abcdef0`| -


## Development

Picus is written in Rust. The project provides a VSCode devcontainer which
contains all required tools.

### Build

```shell
cargo build
```

Build for aarch64 target on x86 environment:

```shell
RUSTFLAGS="-C linker=aarch64-linux-gnu-gcc" PKG_CONFIG_SYSROOT_DIR=/ cargo build --target aarch64-unknown-linux-gnu
```

### Test

```console
cargo test
```

In order to run the tests using Hetzner cloud as well, provide the required
environment variables and run

```console
cargo test hcloud -- --ignored
```

To run all AWS tests with log level set to `trace` for Picus:

```console
 RUST_LOG="picus=trace" cargo test aws -- --ignored --nocapture
```
