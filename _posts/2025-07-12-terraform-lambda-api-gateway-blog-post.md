Building Production-Ready HTTP APIs with Terraform, AWS Lambda and API Gateway

  This guide demonstrates how to build production-ready serverless HTTP APIs using Terraform configuration that combines AWS Lambda with API Gateway, featuring proper logging, monitoring, and security controls for scalable deployment.

  Table of Contents

  - #the-challenge-beyond-basic-serverless
  - #architecture-overview
  - #core-components
  - #key-features-and-implementation
  - #route-configuration
  - #intelligent-route-processing
  - #production-ready-logging
  - #cors-configuration
  - #custom-domain-integration
  - #security-and-best-practices
  - #iam-permissions
  - #encryption
  - #regional-endpoints
  - #real-world-usage-example
  - #monitoring-and-observability
  - #benefits-of-this-approach
  - #considerations-and-limitations
  - #conclusion

  The Challenge: Beyond Basic Serverless

  While AWS makes it relatively straightforward to create a simple Lambda function and API Gateway, production deployments require additional considerations:

  - Comprehensive logging and monitoring
  - Custom domain support with SSL certificates
  - CORS configuration for web applications
  - Proper IAM policies and security controls
  - Routing for multiple endpoints
  - Infrastructure as Code for reproducibility

  Architecture Overview

  The configuration creates a complete serverless HTTP API infrastructure using these key components:

  Internet → Route 53 → API Gateway V2 → Lambda Functions → CloudWatch Logs

  Core Components

  API Gateway V2 (HTTP API): The module uses the newer HTTP API (not REST API) which offers better performance, lower latency, and reduced costs—up to 70% cheaper than REST
  API.

  Lambda Integration: Uses AWS_PROXY integration for request/response handling between API Gateway and Lambda functions.

  CloudWatch Logging: Access logging with multiple format options (JSON, CLF, or custom).

  Custom Domain Support: Integration with Route 53 and ACM certificates for domain management.

  Key Features and Implementation

  Route Configuration

  The standout feature is the flexible route configuration:

  route_config = {
    "GET /users" = {
      lambda_function_arn = "arn:aws:lambda:..."
      description = "Get users endpoint"
      payload_format_version = "2.0"
      timeout_milliseconds = 29000
    }
    "POST /users" = {
      lambda_function_arn = "arn:aws:lambda:..."
      description = "Create user endpoint"
    }
    "PUT /users/{id}" = {
      lambda_function_arn = "arn:aws:lambda:..."
      description = "Update user endpoint"
    }
  }

  This approach provides several benefits:
  - Simple Syntax: HTTP method and path combined in a single key
  - Per-Route Configuration: Each route can have specific settings
  - Dynamic Scaling: Easy to add or remove endpoints

  Intelligent Route Processing

  This includes logic to parse route keys:

  locals {
    route_key_split = {
      for route_key, _ in var.route_config :
      route_key => {
        method = split(" ", route_key)[0]  # Extract HTTP method
        path   = split(" ", route_key)[1]  # Extract path
      }
    }
  }

  This parsing enables the configuration to automatically:
  - Extract HTTP methods (GET, POST, PUT, DELETE)
  - Parse API paths including path parameters
  - Create appropriate API Gateway routes and integrations

  Production-Ready Logging

  The module offers three logging formats:

  JSON Format (recommended for production):
  {
    "requestId": "$context.requestId",
    "extendedRequestId": "$context.extendedRequestId",
    "ip": "$context.identity.sourceIp",
    "caller": "$context.identity.caller",
    "user": "$context.identity.user",
    "requestTime": "$context.requestTime",
    "httpMethod": "$context.httpMethod",
    "resourcePath": "$context.resourcePath",
    "status": "$context.status",
    "protocol": "$context.protocol",
    "responseLength": "$context.responseLength"
  }

  Common Log Format (CLF) for web server compatibility:
  $context.identity.sourceIp - - [$context.requestTime] "$context.httpMethod $context.resourcePath $context.protocol" $context.status $context.responseLength

  CORS Configuration

  Built-in CORS support for web applications:

  cors_configuration = {
    allow_credentials = true
    allow_headers = ["content-type", "x-amz-date", "authorization"]
    allow_methods = ["GET", "POST", "PUT", "DELETE", "OPTIONS"]
    allow_origins = ["https://example.com", "https://app.example.com"]
    max_age = 86400
  }

  Custom Domain Integration

  This configuration integrates with Route 53 and ACM:

  module "api_gateway" {
    source = "./lambda-http-api-gateway"
    
    name = "production-api"
    
    # Custom domain configuration
    create_route53_entry = true
    domain_name = "api.example.com"
    hosted_zone_id = "Z123456789"

    # SSL certificate (managed by ACM)
    certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/..."
    
    route_config = {
      "GET /api/users" = {
        lambda_function_arn = aws_lambda_function.get_users.arn
      }
    }
  }

  Security and Best Practices

  IAM Permissions

  Creating least-privilege IAM policies:

  resource "aws_lambda_permission" "api_gateway_invoke" {
    for_each = var.route_config

    statement_id  = "AllowExecutionFromAPIGateway"
    action        = "lambda:InvokeFunction"
    function_name = each.value.lambda_function_arn
    principal     = "apigateway.amazonaws.com"
    source_arn    = "${aws_apigatewayv2_api.this.execution_arn}/*/*"
  }

  Encryption

  CloudWatch logs are encrypted with customer-managed KMS keys:

  resource "aws_cloudwatch_log_group" "api_gateway_access_logs" {
    name              = var.access_log_cloudwatch_log_group_name
    retention_in_days = var.access_log_retention_in_days
    kms_key_id        = var.access_log_kms_key_id
  }

  Regional Endpoints

  Uses TLS 1.2 and regional endpoints for better performance and security:

  resource "aws_apigatewayv2_domain_name" "this" {
    domain_name = var.domain_name
    
    domain_name_configuration {
      certificate_arn = data.aws_acm_certificate.this[0].arn
      endpoint_type   = "REGIONAL"
      security_policy = "TLS_1_2"
    }
  }

  Real-World Usage Example

  Here's how it might be implemented in a production environment:

  module "user_api" {
    source = "./lambda-http-api-gateway"
    
    name        = "user-management-api"
    description = "User management API for mobile app"
    
    route_config = {
      "GET /users" = {
        lambda_function_arn = aws_lambda_function.list_users.arn
        description = "List all users"
        payload_format_version = "2.0"
      }
      "GET /users/{id}" = {
        lambda_function_arn = aws_lambda_function.get_user.arn
        description = "Get user by ID"
        payload_format_version = "2.0"
      }
      "POST /users" = {
        lambda_function_arn = aws_lambda_function.create_user.arn
        description = "Create new user"
        payload_format_version = "2.0"
      }
      "PUT /users/{id}" = {
        lambda_function_arn = aws_lambda_function.update_user.arn
        description = "Update user"
        payload_format_version = "2.0"
      }
      "DELETE /users/{id}" = {
        lambda_function_arn = aws_lambda_function.delete_user.arn
        description = "Delete user"
        payload_format_version = "2.0"
      }
    }

    # Production domain configuration
    create_route53_entry = true
    domain_name = "api.myapp.com"
    hosted_zone_id = data.aws_route53_zone.main.zone_id

    # CORS for web app
    cors_configuration = {
      allow_credentials = true
      allow_headers = ["content-type", "authorization"]
      allow_methods = ["GET", "POST", "PUT", "DELETE", "OPTIONS"]
      allow_origins = ["https://myapp.com"]
      max_age = 86400
    }

    # Enhanced logging
    access_log_cloudwatch_log_group_name = "/aws/apigateway/user-api"
    access_log_format_type = "JSON"
    access_log_retention_in_days = 30

    # Log forwarding to central logging
    access_log_cloudwatch_log_subscription_filter_name = "user-api-logs"
    access_log_cloudwatch_log_subscription_filter_log_group_destination = "/aws/lambda/log-processor"

    tags = {
      Environment = "production"
      Team        = "backend"
      Project     = "user-management"
    }
  }

  Monitoring and Observability

  The configuration exposes outputs for monitoring:

  output "api_endpoint" {
    value = module.user_api.api_endpoint
  }

  output "cloudwatch_log_group" {
    value = module.user_api.access_log_cloudwatch_log_group_name
  }

  These outputs can be used to create CloudWatch dashboards, alarms, and integrate with monitoring tools like Datadog or New Relic.

  Benefits of This Approach

  Reusability
  This can be used across multiple projects and environments with minimal configuration changes.

  Scalability
  Adding new endpoints is as simple as adding entries to the route_config map.

  Maintainability
  All infrastructure is defined in code, making it easy to version, review, and rollback.

  Security
  Built-in security best practices including proper IAM policies, encryption, and TLS configuration.

  Cost Optimization
  Uses HTTP API instead of REST API for cost savings.

  Considerations and Limitations

  Cold Starts

  Lambda functions may experience cold starts. Consider using provisioned concurrency for critical endpoints.

  Payload Size Limits

  API Gateway has a 10MB payload limit. For larger payloads, using presigned URLs for direct S3 uploads is a better choice.

  Timeout Limits

  API Gateway has a 30-second timeout limit. For longer-running operations, asynchronous processing patterns is advised.

  Conclusion

  We have successfully demonstrated how to build production-ready serverless APIs that go beyond basic tutorials. By combining AWS Lambda with API Gateway V2, proper logging,
  custom domains, and security controls, we can create maintainable APIs that are ready for production workloads.

  This approach makes it easy to standardize API deployment across organizations while maintaining the flexibility to customize for specific use cases. This pattern provides a
  solid foundation for serverless development when building simple REST APIs or complex microservices architectures.

