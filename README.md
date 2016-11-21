<p align="center" >
  <img src="https://raw.githubusercontent.com/choefele/swift-lambda-app/master/swift%2Blambda.png" alt="Swift + Amazon Lambda" title="Swift + Amazon Lambda">
</p>

# Swift Lambda App
Template to build an Amazon Lambda app in Swift

[![Build Status](https://travis-ci.org/choefele/swift-lambda-app.svg?branch=master)](https://travis-ci.org/choefele/swift-lambda-app)

## Overview
This repo contains code and scripts to quickly get you started with writing Swift apps for AWS Lambda, Amazon's serverless computing platform. It contains:
- A sample Lambda app that implements a custom skill for Amazon's Alexa Voice Service using [AlexaSkillsKit](https://github.com/choefele/AlexaSkillsKit)
- A setup to develop and test the app in your development environment
- Scripts to build the app for the Lambda target environment
- Integration tests to proof you app is working before deploying to Lambda
- Instructions on deploying the app to Lambda

swift-lamda-app has been inspired by [SwiftOnLambda](https://github.com/algal/SwiftOnLambda), which provided the initial working code to execute Swift programs on Lambda.

## Development
Tools: [Xcode](https://developer.apple.com/download/), [ngrok](https://ngrok.com) (optional)

The sample app in this repo uses a standard Swift Package Manager directory layout and [package file](https://github.com/choefele/swift-lambda-app/blob/master/Package.swift) thus `swift build`, `swift test` and `swift package generate-xcodeproj` work as expected. Check out the [SPM's documentation](https://github.com/apple/swift-package-manager/blob/master/Documentation/Usage.md) for more info.

There are three targets:
- **AlexaSkill**: this is a library with the code that implements the custom Alexa skill. It's a separate library so it can be used by the other two targets. Also, libraries have `ENABLE_TESTABILITY` enabled by default which allows you to use `@testable import` in your unit tests.
- **Lambda**: The command line executable for deployment to Lambda. This program uses `stdin` and `stdout` for processing data.
- **Server** (macOS only): To simplify implementing a custom Alexa Skill, the Server target provides an HTTP interface to the AlexaSkill library. This HTTP server can be exposed publicly via ngrok and configured in the Alexa console, which enables you to develop and debug an Alexa skill with code running on your development server. This target is macOS only because it wasn't possible to cleanly separate target dependencies and I didn't want to link libraries intended for server development to the Lambda executable used for deployment.

For development, I recommend a [TDD](https://en.wikipedia.org/wiki/Test-driven_development) approach against the library target because this results in the quickest turnaround for code changes. Uploading to Lambda to quickly verify changes isn't really an option because of slow updating times. Exposing your functionality via HTTPS as described below, however, might enable you to test and debug your functionality in a slightly different way.

To run a local HTTPS server:
- Make sure the sample builds by running `swift build`
- Generate an Xcode project with `swift package generate-xcodeproj`
- Open the generated Xcode project, select the Server scheme and run the product (CMD-R). This will start a server at port 8090
- Install ngrok via `brew cask install ngrok`. This tool allows you to expose a local HTTP server to the internet
- Run `ngrok http 8090` and copy the HTTPS URL generated by ngrok (it looks similar to https://258ba658.ngrok.io)
- To create an Alexa skill
 - Go to the [Alexa console](https://developer.amazon.com/edw/home.html#/skills/list) and create a new skill
 - Skill type: Custom Interaction Model
 - Intent: `{ "intents": [{"intent": "TestIntent"}]}`
 - Sample utterances: "TestIntent test swift"
 - Service endpoint type: _HTTPS (use the URL from ngrok)_
 
Now you can test the skill in the Alexa Console using the utterance "test swift". This will call your local HTTP server allowing you to modify and debug your code with the Alexa service.

## Integration
Tools: [Docker](https://www.docker.com/products/docker), [Travis](https://travis-ci.org/choefele/swift-lambda-app)

Before uploading to Lambda, it's worthwhile to build for the target OS and run tests that simulate the execution environment. This repo provides [`build-lambda-package.sh`](https://github.com/choefele/swift-lambda-app/blob/master/build-lambda-package.sh) to do the former and [`run-integration-tests.sh`](https://github.com/choefele/swift-lambda-app/blob/master/run-integration-tests.sh) to do the latter.

`build-lambda-package.sh` builds and tests the Lambda target inside a Swift Docker container based on Ubuntu because there's currently no Swift compiler for Amazon Linux (based on RHEL). Executables built on different Linux distributions are compatible with each other if you provide all dependencies necessary to run the program. For this reason, the script captures all shared libraries required to run the executable using `ldd`.

To prove that the resulting package works, `run-integration-tests.sh` runs the resulting Swift code inside a Docker container that comes close to Lambda’s execution environment (unfortunately, [Amazon only provides Docker images](https://hub.docker.com/_/amazonlinux/) for version 2016.09 of Amazon Linux whereas [Lambda uses 2015.09](http://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html)). The integration with Lambda is done via a small [Node.js script](https://github.com/choefele/swift-lambda-app/blob/master/Shim/index.js) that uses the `child_process` module to run the Swift executable. The script follows Amazon's recommendations to [run arbitrary executables in AWS Lambda](https://aws.amazon.com/blogs/compute/running-executables-in-aws-lambda/).

After [configuring Travis](https://github.com/choefele/swift-lambda-app/blob/master/.travis.yml), you can run the same integration scripts also for every commit.

## Deployment

To deploy your code to Lambda:

- Run `build-lambda-package.sh` to produce a zip file at .build/lambda/lambda.zip with all required files to upload to Lambda
- Create a new Lambda function in the [AWS Console](https://console.aws.amazon.com/lambda/home) in the US East (N. Virginia) region
 - Use an Alexa Skills Kit trigger
 - Runtime: NodeJS 4.3
 - Code entry type: ZIP file (upload the lambda.zip file from the previous step)
 - Handler: index.handler
 - Role: Create from template or use existing role

After creating the Lambda function, you can now create an Alexa skill:
- Go to the [Alexa console](https://developer.amazon.com/edw/home.html#/skills/list) and create a new skill
- Skill type: Custom Interaction Model
- Intent: `{ "intents": [{"intent": "TestIntent"}]}`
- Sample utterances: "TestIntent test swift"
- Service endpoint type: AWS Lambda ARN (use the ARN from the AWS Console)
 
Now you can test the skill in the Alexa Console using the utterance "test swift". More details on configuring Alexa skills can be found on [Amazon's developer portal](https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/overviews/steps-to-build-a-custom-skill).
