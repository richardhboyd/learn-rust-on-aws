# Lesson 02

## Readings

- AWS CloudFormation

- AWS Lambda

- Amazon API Gateway

- Serverless Application Model

- AWS IAM

## Homework

### Project 1 - Just the Basics

Create a SAM application that consists of an API Gateway REST API that proxies all requests to a single Lambda function.
This function should return the following json response `{"hello": "world"}`

### Project 2 - Fine-grained API Paths & Adding IAM Permissions

Create a SAM application that maps specific API Gateway REST API paths to specific Lambda functions.

- HTTP `GET` requests to the path `/functions` should return a list of all Lambda function names in that account and region.
- HTTP `GET` requests to the path `/tables` should return a list of all DynamoDB table names in that account and region.
- HTTP `GET` requests to the path `/functions/some-function-name` should return a description of a Lambda function in that account/region with a matching name.
If the there is no function with that name, the response should indicate as such by returning an HTTP 404 status code.
- HTTP `GET` requests to the path `/tables/some-table-name` should return a description of a DynamoDB table in that account/region with a matching name.
If the there is no table with that name, the response should indicate as such by returning an HTTP 404 status code.

### Project 3 - A Ridiculous Stretch Goal

Repeat Project 2 without using any Lambda functions to process the request, use only the VTL mappings and API Gateway service integrations to perform the lookup.