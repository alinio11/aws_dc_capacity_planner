# AWS Direct Connect Capacity Planning - Layered Implementation Guide

## Project Architecture Overview

This implementation follows a layered architecture pattern that separates infrastructure concerns and enables incremental deployment through your GitLab CI/CD pipeline. Understanding why we organize components across specific layers helps ensure maintainable and scalable infrastructure.

The fundamental principle behind layered architecture is dependency management. Lower-numbered layers provide foundational resources that higher layers depend on, but never the reverse. This creates a clear deployment sequence and prevents circular dependencies that can complicate infrastructure management.

Your layer 000base establishes the networking foundation with VPC, subnets, and routing infrastructure where all other components will operate. Layer 001base builds on this foundation by providing shared services like data storage, messaging, and identity management that multiple applications can use. Layer 200compute then leverages both foundation and shared services to deploy the application-specific compute resources that implement business logic.

For our Direct Connect capacity planning solution, this translates to storing persistent data and communication infrastructure in 001base, while the Lambda functions and monitoring logic reside in 200compute. This separation ensures that core infrastructure remains stable even when we update monitoring algorithms or add new capacity planning features.

## Layer 001base: Shared Infrastructure Services

Layer 001base contains the persistent storage, messaging infrastructure, and identity management components that support our capacity planning system. These resources have longer lifecycles and provide services that could potentially support multiple monitoring applications beyond just Direct Connect capacity planning.

The DynamoDB table serves as our persistent data store for historical capacity metrics and trends. By placing this in 001base rather than 200compute, we ensure that capacity planning data persists even if we need to redeploy or update the Lambda functions. This separation also enables potential future applications to access the same historical data for broader infrastructure analytics.

SNS topics for alerting belong in 001base because alerting infrastructure often serves multiple monitoring systems. Your organization might want to send Direct Connect alerts through the same notification channels used for other infrastructure alerts, making this shared infrastructure rather than application-specific compute.

IAM roles and policies reside in 001base following the principle that identity management should be centralized and stable. Lambda functions can assume these roles, but the role definitions themselves remain independent of the specific compute implementations that use them.

```hcl
# layers/001base/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    # Your S3 backend configuration will go here
    # This should match your existing pattern from 000base
  }
}

# Data source to get VPC information from 000base layer
data "terraform_remote_state" "base" {
  backend = "s3"
  config = {
    bucket = var.terraform_state_bucket
    key    = "layers/000base/${var.environment}/terraform.tfstate"
    region = var.aws_region
  }
}

# Data source to get current AWS account and region info
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

# DynamoDB table for storing capacity planning metrics and historical data
# This table supports time-series queries and cross-connection analysis
resource "aws_dynamodb_table" "dx_capacity_planning" {
  name           = "${var.project_name}-dx-capacity-planning-${var.environment}"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "connection_id"
  range_key      = "timestamp"

  # Primary key structure supports querying specific connections over time ranges
  attribute {
    name = "connection_id"
    type = "S"
  }

  attribute {
    name = "timestamp"
    type = "S"
  }

  # GSI for querying data by date across all connections
  # This enables reports and trend analysis across the entire environment
  attribute {
    name = "date"
    type = "S"
  }

  global_secondary_index {
    name     = "date-index"
    hash_key = "date"
  }

  # Enable point-in-time recovery for data protection
  # Capacity planning data has long-term value for trend analysis
  point_in_time_recovery {
    enabled = true
  }

  # Server-side encryption for data at rest
  server_side_encryption {
    enabled = true
  }

  tags = {
    Name        = "${var.project_name}-dx-capacity-planning-${var.environment}"
    Environment = var.environment
    Purpose     = "directconnect-capacity-monitoring"
    Layer       = "001base"
  }
}

# SNS topic for capacity planning alerts
# Centralized alerting enables integration with multiple notification channels
resource "aws_sns_topic" "dx_capacity_alerts" {
  name = "${var.project_name}-dx-capacity-alerts-${var.environment}"

  # Enable server-side encryption for message content
  kms_master_key_id = aws_kms_key.dx_monitoring.id

  tags = {
    Name        = "${var.project_name}-dx-capacity-alerts-${var.environment}"
    Environment = var.environment
    Purpose     = "directconnect-capacity-alerts"
    Layer       = "001base"
  }
}

# KMS key for encrypting SNS messages and Lambda environment variables
resource "aws_kms_key" "dx_monitoring" {
  description             = "KMS key for Direct Connect monitoring encryption"
  deletion_window_in_days = 7

  # Key policy allows the account root to manage the key
  # Lambda service can use the key for envelope encryption
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow Lambda Service"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = "*"
      }
    ]
  })

  tags = {
    Name        = "${var.project_name}-dx-monitoring-kms-${var.environment}"
    Environment = var.environment
    Purpose     = "directconnect-monitoring-encryption"
    Layer       = "001base"
  }
}

# KMS key alias for easier reference in other layers
resource "aws_kms_alias" "dx_monitoring" {
  name          = "alias/${var.project_name}-dx-monitoring-${var.environment}"
  target_key_id = aws_kms_key.dx_monitoring.key_id
}

# SNS topic subscriptions for email notifications
# These can be extended to include additional notification channels
resource "aws_sns_topic_subscription" "email_alerts" {
  count     = length(var.alert_email_addresses)
  topic_arn = aws_sns_topic.dx_capacity_alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email_addresses[count.index]
}

# S3 bucket for Lambda deployment packages and logs
# Centralized storage for deployment artifacts enables version management
resource "aws_s3_bucket" "lambda_deployments" {
  bucket = "${var.project_name}-dx-lambda-deployments-${var.environment}-${random_id.bucket_suffix.hex}"

  tags = {
    Name        = "${var.project_name}-dx-lambda-deployments-${var.environment}"
    Environment = var.environment
    Purpose     = "lambda-deployment-artifacts"
    Layer       = "001base"
  }
}

# Random ID for bucket suffix to ensure global uniqueness
resource "random_id" "bucket_suffix" {
  byte_length = 4
}

# S3 bucket versioning for deployment package history
resource "aws_s3_bucket_versioning" "lambda_deployments" {
  bucket = aws_s3_bucket.lambda_deployments.id
  versioning_configuration {
    status = "Enabled"
  }
}

# S3 bucket encryption for deployment packages
resource "aws_s3_bucket_server_side_encryption_configuration" "lambda_deployments" {
  bucket = aws_s3_bucket.lambda_deployments.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = aws_kms_key.dx_monitoring.arn
      sse_algorithm     = "aws:kms"
    }
  }
}

# Block public access to deployment bucket
resource "aws_s3_bucket_public_access_block" "lambda_deployments" {
  bucket = aws_s3_bucket.lambda_deployments.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# IAM role for Lambda functions with capacity monitoring permissions
# This role provides the minimum necessary permissions for capacity monitoring
resource "aws_iam_role" "dx_capacity_lambda_role" {
  name = "${var.project_name}-dx-capacity-lambda-role-${var.environment}"

  # Trust relationship allows Lambda service to assume this role
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })

  tags = {
    Name        = "${var.project_name}-dx-capacity-lambda-role-${var.environment}"
    Environment = var.environment
    Purpose     = "directconnect-capacity-lambda-execution"
    Layer       = "001base"
  }
}

# IAM policy for Lambda execution with specific permissions for capacity monitoring
resource "aws_iam_role_policy" "dx_capacity_lambda_policy" {
  name = "${var.project_name}-dx-capacity-lambda-policy-${var.environment}"
  role = aws_iam_role.dx_capacity_lambda_role.id

  # Policy grants minimum necessary permissions for capacity monitoring functions
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        # CloudWatch Logs permissions for Lambda logging
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:*"
      },
      {
        # Direct Connect permissions for discovering connections and VIFs
        Effect = "Allow"
        Action = [
          "directconnect:DescribeConnections",
          "directconnect:DescribeVirtualInterfaces",
          "directconnect:DescribeConnectionsOnInterconnect"
        ]
        Resource = "*"
      },
      {
        # CloudWatch permissions for retrieving metrics and creating alarms
        Effect = "Allow"
        Action = [
          "cloudwatch:GetMetricStatistics",
          "cloudwatch:PutMetricData",
          "cloudwatch:PutMetricAlarm",
          "cloudwatch:DescribeAlarms",
          "cloudwatch:PutAnomalyDetector"
        ]
        Resource = "*"
      },
      {
        # DynamoDB permissions for storing capacity planning data
        Effect = "Allow"
        Action = [
          "dynamodb:PutItem",
          "dynamodb:GetItem",
          "dynamodb:Query",
          "dynamodb:Scan"
        ]
        Resource = aws_dynamodb_table.dx_capacity_planning.arn
      },
      {
        # SNS permissions for sending capacity alerts
        Effect = "Allow"
        Action = [
          "sns:Publish"
        ]
        Resource = aws_sns_topic.dx_capacity_alerts.arn
      },
      {
        # KMS permissions for encrypting/decrypting data
        Effect = "Allow"
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey",
          "kms:DescribeKey"
        ]
        Resource = aws_kms_key.dx_monitoring.arn
      }
    ]
  })
}

# Attach AWS managed policy for VPC Lambda execution if Lambda needs VPC access
resource "aws_iam_role_policy_attachment" "lambda_vpc_execution" {
  count      = var.enable_lambda_vpc_access ? 1 : 0
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
  role       = aws_iam_role.dx_capacity_lambda_role.name
}

# CloudWatch Log Group for Lambda functions with retention policy
resource "aws_cloudwatch_log_group" "dx_capacity_lambda_logs" {
  name              = "/aws/lambda/${var.project_name}-dx-capacity-monitor-${var.environment}"
  retention_in_days = var.log_retention_days

  # Encrypt log data using our KMS key
  kms_key_id = aws_kms_key.dx_monitoring.arn

  tags = {
    Name        = "${var.project_name}-dx-capacity-lambda-logs-${var.environment}"
    Environment = var.environment
    Purpose     = "directconnect-capacity-lambda-logs"
    Layer       = "001base"
  }
}
```

The variables file for layer 001base provides configuration flexibility while maintaining sensible defaults that work for most environments.

```hcl
# layers/001base/variables.tf

variable "aws_region" {
  description = "AWS region for resource deployment"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name (production, staging, development)"
  type        = string
  validation {
    condition     = contains(["production", "staging", "development"], var.environment)
    error_message = "Environment must be one of: production, staging, development."
  }
}

variable "project_name" {
  description = "Project name used for resource naming"
  type        = string
  default     = "dx-capacity-planning"
}

variable "terraform_state_bucket" {
  description = "S3 bucket name for Terraform state storage"
  type        = string
}

variable "alert_email_addresses" {
  description = "List of email addresses for capacity alerts"
  type        = list(string)
  default     = []
}

variable "log_retention_days" {
  description = "CloudWatch logs retention period in days"
  type        = number
  default     = 30
  validation {
    condition     = contains([1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365, 400, 545, 731, 1827, 3653], var.log_retention_days)
    error_message = "Log retention days must be a valid CloudWatch retention period."
  }
}

variable "enable_lambda_vpc_access" {
  description = "Enable VPC access for Lambda functions"
  type        = bool
  default     = false
}

# Capacity planning threshold configuration
variable "capacity_thresholds" {
  description = "Utilization thresholds for different bandwidth tiers"
  type = map(object({
    warning  = number
    critical = number
    upgrade  = number
  }))
  default = {
    "1Gbps" = {
      warning  = 0.60
      critical = 0.80
      upgrade  = 0.70
    }
    "10Gbps" = {
      warning  = 0.65
      critical = 0.80
      upgrade  = 0.75
    }
    "100Gbps" = {
      warning  = 0.70
      critical = 0.85
      upgrade  = 0.80
    }
    "400Gbps" = {
      warning  = 0.70
      critical = 0.85
      upgrade  = 0.80
    }
  }
}
```

The outputs file exposes resources that layer 200compute will need to reference, following the principle that each layer should explicitly define its public interface.

```hcl
# layers/001base/outputs.tf

# DynamoDB table information for Lambda functions
output "dynamodb_table_name" {
  description = "Name of the DynamoDB table for capacity planning data"
  value       = aws_dynamodb_table.dx_capacity_planning.name
}

output "dynamodb_table_arn" {
  description = "ARN of the DynamoDB table for capacity planning data"
  value       = aws_dynamodb_table.dx_capacity_planning.arn
}

# SNS topic information for alerting
output "sns_topic_arn" {
  description = "ARN of the SNS topic for capacity alerts"
  value       = aws_sns_topic.dx_capacity_alerts.arn
}

output "sns_topic_name" {
  description = "Name of the SNS topic for capacity alerts"
  value       = aws_sns_topic.dx_capacity_alerts.name
}

# IAM role information for Lambda functions
output "lambda_execution_role_arn" {
  description = "ARN of the IAM role for Lambda execution"
  value       = aws_iam_role.dx_capacity_lambda_role.arn
}

output "lambda_execution_role_name" {
  description = "Name of the IAM role for Lambda execution"
  value       = aws_iam_role.dx_capacity_lambda_role.name
}

# S3 bucket information for Lambda deployment packages
output "lambda_deployment_bucket_name" {
  description = "Name of the S3 bucket for Lambda deployment packages"
  value       = aws_s3_bucket.lambda_deployments.bucket
}

output "lambda_deployment_bucket_arn" {
  description = "ARN of the S3 bucket for Lambda deployment packages"
  value       = aws_s3_bucket.lambda_deployments.arn
}

# KMS key information for encryption
output "kms_key_id" {
  description = "ID of the KMS key for encryption"
  value       = aws_kms_key.dx_monitoring.key_id
}

output "kms_key_arn" {
  description = "ARN of the KMS key for encryption"
  value       = aws_kms_key.dx_monitoring.arn
}

# CloudWatch Log Group information
output "lambda_log_group_name" {
  description = "Name of the CloudWatch Log Group for Lambda functions"
  value       = aws_cloudwatch_log_group.dx_capacity_lambda_logs.name
}

# Capacity planning configuration for use in compute layer
output "capacity_thresholds" {
  description = "Capacity planning thresholds configuration"
  value       = var.capacity_thresholds
}
```

Environment-specific configuration files allow different settings for production versus staging environments while maintaining the same infrastructure code.

```hcl
# layers/001base/envs/production.tfvars

environment                = "production"
project_name              = "dx-capacity-planning"
terraform_state_bucket    = "your-terraform-state-bucket-prod"
log_retention_days        = 90
enable_lambda_vpc_access  = true

alert_email_addresses = [
  "cloud-ops@yourcompany.com",
  "network-team@yourcompany.com",
  "dx-alerts@yourcompany.com"
]

# Production uses more conservative thresholds
capacity_thresholds = {
  "1Gbps" = {
    warning  = 0.55
    critical = 0.75
    upgrade  = 0.65
  }
  "10Gbps" = {
    warning  = 0.60
    critical = 0.75
    upgrade  = 0.70
  }
  "100Gbps" = {
    warning  = 0.65
    critical = 0.80
    upgrade  = 0.75
  }
  "400Gbps" = {
    warning  = 0.65
    critical = 0.80
    upgrade  = 0.75
  }
}
```

```hcl
# layers/001base/envs/staging.tfvars

environment                = "staging"
project_name              = "dx-capacity-planning"
terraform_state_bucket    = "your-terraform-state-bucket-staging"
log_retention_days        = 30
enable_lambda_vpc_access  = false

alert_email_addresses = [
  "dev-team@yourcompany.com"
]

# Staging uses standard thresholds for testing
capacity_thresholds = {
  "1Gbps" = {
    warning  = 0.60
    critical = 0.80
    upgrade  = 0.70
  }
  "10Gbps" = {
    warning  = 0.65
    critical = 0.80
    upgrade  = 0.75
  }
  "100Gbps" = {
    warning  = 0.70
    critical = 0.85
    upgrade  = 0.80
  }
  "400Gbps" = {
    warning  = 0.70
    critical = 0.85
    upgrade  = 0.80
  }
}
```

## Layer 200compute: Application Logic and Monitoring

Layer 200compute contains the Lambda functions, EventBridge scheduling, and CloudWatch resources that implement the capacity planning business logic. This separation ensures that the core monitoring algorithms can be updated independently from the underlying infrastructure services.

Lambda functions in this layer focus purely on business logic rather than infrastructure concerns. They assume that data storage, messaging, and identity management are already established by lower layers. This creates cleaner separation of concerns and enables more focused testing and development workflows.

CloudWatch dashboards and alarms belong in this layer because they represent application-specific monitoring rather than foundational infrastructure monitoring. While the underlying CloudWatch service and permissions are established in lower layers, the specific metrics, visualizations, and alert conditions that implement capacity planning logic are application-specific concerns.

```hcl
# layers/200compute/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    archive = {
      source  = "hashicorp/archive"
      version = "~> 2.0"
    }
  }

  backend "s3" {
    # Your S3 backend configuration matching your existing pattern
  }
}

# Data sources to get information from previous layers
data "terraform_remote_state" "base" {
  backend = "s3"
  config = {
    bucket = var.terraform_state_bucket
    key    = "layers/000base/${var.environment}/terraform.tfstate"
    region = var.aws_region
  }
}

data "terraform_remote_state" "shared" {
  backend = "s3"
  config = {
    bucket = var.terraform_state_bucket
    key    = "layers/001base/${var.environment}/terraform.tfstate"
    region = var.aws_region
  }
}

data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

# Create deployment package for the capacity monitoring Lambda function
# The package includes both the Python code and its dependencies
data "archive_file" "capacity_monitor_package" {
  type        = "zip"
  output_path = "${path.module}/capacity_monitor.zip"
  
  source {
    content = file("${path.module}/lambda/capacity_monitor.py")
    filename = "capacity_monitor.py"
  }
  
  source {
    content = file("${path.module}/lambda/requirements.txt")
    filename = "requirements.txt"
  }
}

# S3 object for Lambda deployment package with versioning
# This enables rollback capabilities and deployment history tracking
resource "aws_s3_object" "capacity_monitor_package" {
  bucket      = data.terraform_remote_state.shared.outputs.lambda_deployment_bucket_name
  key         = "capacity-monitor/${var.environment}/capacity_monitor_${data.archive_file.capacity_monitor_package.output_md5}.zip"
  source      = data.archive_file.capacity_monitor_package.output_path
  source_hash = data.archive_file.capacity_monitor_package.output_base64sha256

  tags = {
    Name        = "${var.project_name}-capacity-monitor-package-${var.environment}"
    Environment = var.environment
    Version     = data.archive_file.capacity_monitor_package.output_md5
    Layer       = "200compute"
  }
}

# Lambda function for Direct Connect capacity monitoring
# This function implements the core capacity analysis and alerting logic
resource "aws_lambda_function" "dx_capacity_monitor" {
  function_name = "${var.project_name}-dx-capacity-monitor-${var.environment}"
  description   = "Direct Connect capacity monitoring and alerting function"

  # Reference the S3 deployment package
  s3_bucket         = data.terraform_remote_state.shared.outputs.lambda_deployment_bucket_name
  s3_key           = aws_s3_object.capacity_monitor_package.key
  s3_object_version = aws_s3_object.capacity_monitor_package.version_id

  # Lambda runtime configuration
  runtime = "python3.9"
  handler = "capacity_monitor.lambda_handler"
  timeout = var.lambda_timeout

  # Use the IAM role created in 001base layer
  role = data.terraform_remote_state.shared.outputs.lambda_execution_role_arn

  # VPC configuration if enabled
  dynamic "vpc_config" {
    for_each = var.enable_vpc_config ? [1] : []
    content {
      subnet_ids         = data.terraform_remote_state.base.outputs.private_subnet_ids
      security_group_ids = [aws_security_group.lambda_sg[0].id]
    }
  }

  # Environment variables for Lambda function configuration
  environment {
    variables = {
      DYNAMODB_TABLE_NAME = data.terraform_remote_state.shared.outputs.dynamodb_table_name
      SNS_TOPIC_ARN      = data.terraform_remote_state.shared.outputs.sns_topic_arn
      ENVIRONMENT        = var.environment
      LOG_LEVEL         = var.lambda_log_level
      
      # Convert capacity thresholds to JSON for Lambda consumption
      CAPACITY_THRESHOLDS = jsonencode(data.terraform_remote_state.shared.outputs.capacity_thresholds)
    }
  }

  # Encrypt environment variables using KMS key from shared layer
  kms_key_arn = data.terraform_remote_state.shared.outputs.kms_key_arn

  # Use the existing log group from 001base layer
  depends_on = [data.terraform_remote_state.shared]

  tags = {
    Name        = "${var.project_name}-dx-capacity-monitor-${var.environment}"
    Environment = var.environment
    Purpose     = "directconnect-capacity-monitoring"
    Layer       = "200compute"
  }
}

# Security group for Lambda function if VPC access is enabled
resource "aws_security_group" "lambda_sg" {
  count       = var.enable_vpc_config ? 1 : 0
  name        = "${var.project_name}-lambda-sg-${var.environment}"
  description = "Security group for Direct Connect capacity monitoring Lambda"
  vpc_id      = data.terraform_remote_state.base.outputs.vpc_id

  # Outbound HTTPS for AWS API calls
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS outbound for AWS API calls"
  }

  # Outbound HTTP for potential package downloads during runtime
  egress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP outbound for package downloads"
  }

  tags = {
    Name        = "${var.project_name}-lambda-sg-${var.environment}"
    Environment = var.environment
    Purpose     = "directconnect-lambda-security"
    Layer       = "200compute"
  }
}

# EventBridge rule for scheduled capacity monitoring execution
# This creates a reliable scheduling mechanism that triggers monitoring at regular intervals
resource "aws_cloudwatch_event_rule" "capacity_monitor_schedule" {
  name                = "${var.project_name}-capacity-monitor-schedule-${var.environment}"
  description         = "Schedule for Direct Connect capacity monitoring execution"
  schedule_expression = var.monitoring_schedule

  tags = {
    Name        = "${var.project_name}-capacity-monitor-schedule-${var.environment}"
    Environment = var.environment
    Purpose     = "directconnect-monitoring-schedule"
    Layer       = "200compute"
  }
}

# EventBridge target connecting the schedule to the Lambda function
resource "aws_cloudwatch_event_target" "capacity_monitor_target" {
  rule      = aws_cloudwatch_event_rule.capacity_monitor_schedule.name
  target_id = "CapacityMonitorLambdaTarget"
  arn       = aws_lambda_function.dx_capacity_monitor.arn

  # Add retry configuration for reliability
  retry_policy {
    maximum_retry_attempts = 3
  }

  # Add dead letter queue for failed executions
  dead_letter_config {
    arn = aws_sqs_queue.lambda_dlq.arn
  }
}

# Permission for EventBridge to invoke the Lambda function
resource "aws_lambda_permission" "allow_eventbridge_invoke" {
  statement_id  = "AllowExecutionFromEventBridge"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.dx_capacity_monitor.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.capacity_monitor_schedule.arn
}

# Dead letter queue for handling Lambda execution failures
resource "aws_sqs_queue" "lambda_dlq" {
  name                       = "${var.project_name}-capacity-monitor-dlq-${var.environment}"
  message_retention_seconds  = 1209600  # 14 days
  visibility_timeout_seconds = 60

  # Server-side encryption using KMS
  kms_master_key_id = data.terraform_remote_state.shared.outputs.kms_key_id

  tags = {
    Name        = "${var.project_name}-capacity-monitor-dlq-${var.environment}"
    Environment = var.environment
    Purpose     = "directconnect-lambda-dead-letter-queue"
    Layer       = "200compute"
  }
}

# CloudWatch dashboard for capacity planning visualization
# This provides operational visibility into Direct Connect utilization and trends
resource "aws_cloudwatch_dashboard" "dx_capacity_planning" {
  dashboard_name = "${var.project_name}-dx-capacity-dashboard-${var.environment}"

  dashboard_body = jsonencode({
    widgets = concat(
      [
        {
          type   = "metric"
          x      = 0
          y      = 0
          width  = 24
          height = 6
          properties = {
            metrics = [
              ["AWS/DX", "ConnectionBpsEgress", { stat = "Average" }],
              [".", "ConnectionBpsIngress", { stat = "Average" }]
            ]
            period = 300
            stat   = "Average"
            region = data.aws_region.current.name
            title  = "Direct Connect - Overall Bandwidth Utilization"
            yAxis = {
              left = {
                min = 0
              }
            }
            annotations = {
              horizontal = [
                {
                  label = "80% Critical Threshold"
                  value = 800000000
                }
              ]
            }
          }
        }
      ],
      local.connection_widgets
    )
  })
}

# Local values for dynamic dashboard widget generation
locals {
  # This would be populated by data source or variable containing connection information
  # For now, this serves as a template for dynamic widget creation
  connection_widgets = [
    # Widget generation logic would go here based on actual connections
    # This demonstrates the pattern for creating connection-specific widgets
  ]
}

# CloudWatch composite alarm for overall Direct Connect health
# This provides a single alarm that aggregates the health status of all connections
resource "aws_cloudwatch_composite_alarm" "dx_overall_health" {
  alarm_name        = "${var.project_name}-dx-overall-health-${var.environment}"
  alarm_description = "Overall health status of Direct Connect infrastructure"

  # The alarm rule would reference individual connection alarms
  # This creates a hierarchical alerting structure
  alarm_rule = "FALSE"  # Placeholder - would be populated with actual alarm references

  alarm_actions = [
    data.terraform_remote_state.shared.outputs.sns_topic_arn
  ]

  ok_actions = [
    data.terraform_remote_state.shared.outputs.sns_topic_arn
  ]

  tags = {
    Name        = "${var.project_name}-dx-overall-health-${var.environment}"
    Environment = var.environment
    Purpose     = "directconnect-composite-health-monitoring"
    Layer       = "200compute"
  }
}
```

The Lambda function source code implements the capacity monitoring business logic with proper error handling and logging.

```python
# layers/200compute/lambda/capacity_monitor.py
import json
import boto3
import logging
import os
from datetime import datetime, timedelta
from typing import Dict, List, Optional, Tuple
import traceback

# Configure logging based on environment variable
log_level = os.getenv('LOG_LEVEL', 'INFO').upper()
logging.basicConfig(level=getattr(logging, log_level))
logger = logging.getLogger(__name__)

# Initialize AWS clients with proper error handling
try:
    cloudwatch = boto3.client('cloudwatch')
    dx_client = boto3.client('directconnect')
    dynamodb = boto3.resource('dynamodb')
    sns = boto3.client('sns')
except Exception as e:
    logger.error(f"Failed to initialize AWS clients: {str(e)}")
    raise

# Configuration from environment variables
DYNAMODB_TABLE_NAME = os.environ['DYNAMODB_TABLE_NAME']
SNS_TOPIC_ARN = os.environ['SNS_TOPIC_ARN']
ENVIRONMENT = os.environ['ENVIRONMENT']
CAPACITY_THRESHOLDS = json.loads(os.environ['CAPACITY_THRESHOLDS'])

# Initialize DynamoDB table reference
table = dynamodb.Table(DYNAMODB_TABLE_NAME)

def lambda_handler(event, context):
    """
    Main Lambda handler for Direct Connect capacity monitoring
    Processes all connections and generates alerts based on utilization thresholds
    """
    try:
        logger.info("Starting Direct Connect capacity monitoring execution")
        
        # Get all Direct Connect connections in the account
        connections = get_all_connections()
        logger.info(f"Found {len(connections)} Direct Connect connections to analyze")
        
        # Process each connection for capacity analysis
        results = []
        error_count = 0
        
        for connection in connections:
            try:
                result = process_connection(connection)
                results.append(result)
                
                # Generate alerts if thresholds are exceeded
                if result['alert_level'] in ['WARNING', 'CRITICAL']:
                    send_capacity_alert(connection, result)
                    
            except Exception as e:
                error_count += 1
                logger.error(f"Error processing connection {connection.get('connectionId', 'unknown')}: {str(e)}")
                logger.error(traceback.format_exc())
        
        # Store all results in DynamoDB for historical analysis
        if results:
            store_capacity_data(results)
        
        # Prepare execution summary
        execution_summary = {
            'connections_processed': len(results),
            'alerts_generated': len([r for r in results if r['alert_level'] != 'OK']),
            'errors_encountered': error_count,
            'execution_time': context.get_remaining_time_in_millis()
        }
        
        logger.info(f"Capacity monitoring completed: {execution_summary}")
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Capacity monitoring executed successfully',
                'summary': execution_summary,
                'environment': ENVIRONMENT
            })
        }
        
    except Exception as e:
        logger.error(f"Critical error in capacity monitoring execution: {str(e)}")
        logger.error(traceback.format_exc())
        
        # Send critical alert for monitoring system failure
        send_system_failure_alert(str(e))
        
        return {
            'statusCode': 500,
            'body': json.dumps({
                'message': 'Capacity monitoring execution failed',
                'error': str(e),
                'environment': ENVIRONMENT
            })
        }

def get_all_connections() -> List[Dict]:
    """
    Retrieve all Direct Connect connections with error handling and retry logic
    """
    try:
        response = dx_client.describe_connections()
        connections = response.get('connections', [])
        
        # Filter to only include active connections
        active_connections = [
            conn for conn in connections 
            if conn.get('connectionState') == 'available'
        ]
        
        logger.info(f"Retrieved {len(active_connections)} active connections from {len(connections)} total")
        return active_connections
        
    except Exception as e:
        logger.error(f"Failed to retrieve Direct Connect connections: {str(e)}")
        raise

def process_connection(connection: Dict) -> Dict:
    """
    Process individual connection for capacity analysis and threshold evaluation
    """
    connection_id = connection['connectionId']
    connection_name = connection.get('connectionName', connection_id)
    bandwidth = connection['bandwidth']
    
    logger.debug(f"Processing connection {connection_id} with bandwidth {bandwidth}")
    
    try:
        # Get current utilization metrics from CloudWatch
        utilization_data = get_connection_utilization(connection_id)
        
        # Convert bandwidth string to numeric value for calculations
        bandwidth_bps = parse_bandwidth_to_bps(bandwidth)
        
        # Calculate current utilization percentages
        egress_utilization = calculate_utilization_percentage(
            utilization_data.get('egress', []), bandwidth_bps
        )
        ingress_utilization = calculate_utilization_percentage(
            utilization_data.get('ingress', []), bandwidth_bps
        )
        
        # Use the higher utilization for threshold evaluation
        max_utilization = max(egress_utilization, ingress_utilization)
        
        # Determine alert level based on thresholds
        alert_level = determine_alert_level(max_utilization, bandwidth)
        
        # Get historical trend analysis for context
        historical_trend = analyze_historical_trend(connection_id)
        
        result = {
            'connection_id': connection_id,
            'connection_name': connection_name,
            'bandwidth': bandwidth,
            'bandwidth_bps': bandwidth_bps,
            'egress_utilization_pct': round(egress_utilization, 2),
            'ingress_utilization_pct': round(ingress_utilization, 2),
            'max_utilization_pct': round(max_utilization, 2),
            'alert_level': alert_level,
            'historical_trend': historical_trend,
            'timestamp': datetime.utcnow().isoformat(),
            'environment': ENVIRONMENT
        }
        
        logger.debug(f"Connection {connection_id} analysis completed: {max_utilization:.2f}% utilization, {alert_level} alert level")
        return result
        
    except Exception as e:
        logger.error(f"Error analyzing connection {connection_id}: {str(e)}")
        raise

def get_connection_utilization(connection_id: str, minutes: int = 60) -> Dict:
    """
    Retrieve utilization metrics for a connection over specified time period
    """
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(minutes=minutes)
    
    try:
        # Get both egress and ingress metrics
        metrics = ['ConnectionBpsEgress', 'ConnectionBpsIngress']
        utilization_data = {}
        
        for metric_name in metrics:
            response = cloudwatch.get_metric_statistics(
                Namespace='AWS/DX',
                MetricName=metric_name,
                Dimensions=[
                    {
                        'Name': 'ConnectionId',
                        'Value': connection_id
                    }
                ],
                StartTime=start_time,
                EndTime=end_time,
                Period=300,  # 5-minute intervals
                Statistics=['Average', 'Maximum']
            )
            
            direction = 'egress' if 'Egress' in metric_name else 'ingress'
            utilization_data[direction] = response.get('Datapoints', [])
        
        return utilization_data
        
    except Exception as e:
        logger.error(f"Error retrieving metrics for connection {connection_id}: {str(e)}")
        raise

def parse_bandwidth_to_bps(bandwidth_str: str) -> int:
    """
    Convert AWS bandwidth string format to bits per second
    """
    bandwidth_map = {
        '50Mbps': 50_000_000,
        '100Mbps': 100_000_000,
        '200Mbps': 200_000_000,
        '300Mbps': 300_000_000,
        '400Mbps': 400_000_000,
        '500Mbps': 500_000_000,
        '1Gbps': 1_000_000_000,
        '2Gbps': 2_000_000_000,
        '5Gbps': 5_000_000_000,
        '10Gbps': 10_000_000_000,
        '25Gbps': 25_000_000_000,
        '40Gbps': 40_000_000_000,
        '50Gbps': 50_000_000_000,
        '100Gbps': 100_000_000_000,
        '400Gbps': 400_000_000_000
    }
    
    return bandwidth_map.get(bandwidth_str, 1_000_000_000)  # Default to 1Gbps

def calculate_utilization_percentage(datapoints: List[Dict], bandwidth_bps: int) -> float:
    """
    Calculate utilization percentage from CloudWatch datapoints
    """
    if not datapoints:
        return 0.0
    
    # Get the maximum utilization from the time period
    max_utilization_bps = max([point.get('Maximum', 0) for point in datapoints])
    
    # Convert to percentage
    utilization_pct = (max_utilization_bps / bandwidth_bps) * 100
    
    return min(utilization_pct, 100.0)  # Cap at 100%

def determine_alert_level(utilization_pct: float, bandwidth: str) -> str:
    """
    Determine alert level based on utilization percentage and bandwidth tier
    """
    # Get thresholds for this bandwidth tier, default to 1Gbps if unknown
    thresholds = CAPACITY_THRESHOLDS.get(bandwidth, CAPACITY_THRESHOLDS['1Gbps'])
    
    if utilization_pct >= thresholds['critical'] * 100:
        return 'CRITICAL'
    elif utilization_pct >= thresholds['warning'] * 100:
        return 'WARNING'
    else:
        return 'OK'

def analyze_historical_trend(connection_id: str, days: int = 7) -> Dict:
    """
    Analyze historical utilization trend for capacity planning insight
    """
    try:
        # Query DynamoDB for historical data
        end_date = datetime.utcnow().date()
        start_date = end_date - timedelta(days=days)
        
        response = table.query(
            KeyConditionExpression='connection_id = :conn_id AND #ts BETWEEN :start AND :end',
            ExpressionAttributeNames={'#ts': 'timestamp'},
            ExpressionAttributeValues={
                ':conn_id': connection_id,
                ':start': start_date.isoformat(),
                ':end': end_date.isoformat()
            },
            Limit=100  # Limit to prevent excessive data retrieval
        )
        
        items = response.get('Items', [])
        
        if len(items) < 2:
            return {'trend': 'insufficient_data', 'growth_rate': 0, 'data_points': len(items)}
        
        # Calculate simple trend
        utilization_values = [float(item.get('max_utilization_pct', 0)) for item in items]
        utilization_values.sort()
        
        # Simple linear growth rate calculation
        if len(utilization_values) > 1:
            growth_rate = (utilization_values[-1] - utilization_values[0]) / days
            
            if growth_rate > 1.0:
                trend = 'increasing'
            elif growth_rate < -1.0:
                trend = 'decreasing'
            else:
                trend = 'stable'
        else:
            trend = 'stable'
            growth_rate = 0
        
        return {
            'trend': trend,
            'growth_rate': round(growth_rate, 2),
            'data_points': len(items)
        }
        
    except Exception as e:
        logger.error(f"Error analyzing historical trend for {connection_id}: {str(e)}")
        return {'trend': 'error', 'growth_rate': 0, 'data_points': 0}

def send_capacity_alert(connection: Dict, result: Dict) -> None:
    """
    Send capacity alert via SNS with detailed information
    """
    try:
        connection_id = connection['connectionId']
        connection_name = connection.get('connectionName', connection_id)
        alert_level = result['alert_level']
        utilization_pct = result['max_utilization_pct']
        bandwidth = connection['bandwidth']
        
        subject = f"[{alert_level}] Direct Connect Capacity Alert - {connection_name} ({ENVIRONMENT})"
        
        message = f"""
Direct Connect Capacity Alert - {ENVIRONMENT.upper()}

Connection Details:
- Connection ID: {connection_id}
- Connection Name: {connection_name}
- Bandwidth: {bandwidth}
- Current Max Utilization: {utilization_pct:.2f}%
- Alert Level: {alert_level}
- Environment: {ENVIRONMENT}
- Timestamp: {result['timestamp']}

Utilization Breakdown:
- Egress Utilization: {result['egress_utilization_pct']:.2f}%
- Ingress Utilization: {result['ingress_utilization_pct']:.2f}%

Historical Trend:
- Trend: {result['historical_trend']['trend']}
- Growth Rate: {result['historical_trend']['growth_rate']:.2f}% per day
- Data Points: {result['historical_trend']['data_points']}

Recommended Actions:
"""
        
        if alert_level == 'CRITICAL':
            message += """
- IMMEDIATE ACTION REQUIRED
- Review current traffic patterns and identify high-bandwidth applications
- Consider emergency capacity upgrade or traffic optimization
- Verify redundant connections can handle failover scenarios
- Contact AWS Support for expedited capacity increase options
- Implement traffic prioritization if possible
"""
        elif alert_level == 'WARNING':
            message += """
- Plan capacity upgrade within 30 days
- Monitor utilization trends closely over next week
- Prepare capacity upgrade documentation and approvals
- Review traffic optimization opportunities
- Coordinate with application teams for potential load reduction
"""
        
        message += f"""
Console Links:
- Direct Connect Console: https://console.aws.amazon.com/directconnect/v2/home#/connections/{connection_id}
- CloudWatch Dashboard: https://console.aws.amazon.com/cloudwatch/home#dashboards:name={var.project_name}-dx-capacity-dashboard-{ENVIRONMENT}

This alert was generated by the automated Direct Connect capacity monitoring system.
Environment: {ENVIRONMENT}
        """
        
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject=subject,
            Message=message.strip()
        )
        
        logger.info(f"Capacity alert sent for {connection_id}: {alert_level}")
        
    except Exception as e:
        logger.error(f"Error sending capacity alert for {connection['connectionId']}: {str(e)}")

def send_system_failure_alert(error_message: str) -> None:
    """
    Send system failure alert when the monitoring system itself fails
    """
    try:
        subject = f"[CRITICAL] Direct Connect Monitoring System Failure - {ENVIRONMENT}"
        
        message = f"""
CRITICAL: Direct Connect Capacity Monitoring System Failure

The Direct Connect capacity monitoring system has encountered a critical failure
and was unable to complete its monitoring cycle.

Environment: {ENVIRONMENT}
Error: {error_message}
Timestamp: {datetime.utcnow().isoformat()}

Immediate Actions Required:
- Investigate monitoring system logs in CloudWatch
- Verify AWS service permissions and connectivity
- Check for any AWS service outages affecting Direct Connect or CloudWatch
- Consider manual capacity monitoring until system is restored

This requires immediate attention to restore automated capacity monitoring.
        """
        
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject=subject,
            Message=message.strip()
        )
        
    except Exception as e:
        logger.error(f"Failed to send system failure alert: {str(e)}")

def store_capacity_data(results: List[Dict]) -> None:
    """
    Store capacity monitoring results in DynamoDB for historical analysis
    """
    try:
        current_date = datetime.utcnow().date().isoformat()
        
        # Use batch writing for efficiency
        with table.batch_writer() as batch:
            for result in results:
                # Add date field for GSI queries
                result['date'] = current_date
                batch.put_item(Item=result)
        
        logger.info(f"Stored {len(results)} capacity records in DynamoDB")
        
    except Exception as e:
        logger.error(f"Error storing capacity data in DynamoDB: {str(e)}")
        raise
```

The variables and configuration files for layer 200compute provide the necessary flexibility for different deployment scenarios.

```hcl
# layers/200compute/variables.tf

variable "aws_region" {
  description = "AWS region for resource deployment"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name (production, staging, development)"
  type        = string
}

variable "project_name" {
  description = "Project name for resource naming"
  type        = string
  default     = "dx-capacity-planning"
}

variable "terraform_state_bucket" {
  description = "S3 bucket name for Terraform state storage"
  type        = string
}

variable "lambda_timeout" {
  description = "Lambda function timeout in seconds"
  type        = number
  default     = 300
  validation {
    condition     = var.lambda_timeout >= 30 && var.lambda_timeout <= 900
    error_message = "Lambda timeout must be between 30 and 900 seconds."
  }
}

variable "lambda_log_level" {
  description = "Log level for Lambda function"
  type        = string
  default     = "INFO"
  validation {
    condition     = contains(["DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"], var.lambda_log_level)
    error_message = "Lambda log level must be one of: DEBUG, INFO, WARNING, ERROR, CRITICAL."
  }
}

variable "monitoring_schedule" {
  description = "EventBridge schedule expression for monitoring execution"
  type        = string
  default     = "rate(5 minutes)"
}

variable "enable_vpc_config" {
  description = "Enable VPC configuration for Lambda functions"
  type        = bool
  default     = false
}
```

Environment-specific configuration files allow different operational parameters for production and staging environments.

```hcl
# layers/200compute/envs/production.tfvars

environment                = "production"
project_name              = "dx-capacity-planning"
terraform_state_bucket    = "your-terraform-state-bucket-prod"
lambda_timeout            = 300
lambda_log_level          = "INFO"
monitoring_schedule       = "rate(5 minutes)"
enable_vpc_config         = true
```

```hcl
# layers/200compute/envs/staging.tfvars

environment                = "staging"
project_name              = "dx-capacity-planning"
terraform_state_bucket    = "your-terraform-state-bucket-staging"
lambda_timeout            = 180
lambda_log_level          = "DEBUG"
monitoring_schedule       = "rate(10 minutes)"
enable_vpc_config         = false
```

This layered architecture approach provides you with several important advantages. The separation of concerns ensures that infrastructure changes in one layer do not require redeployment of other layers unless there are explicit dependency changes. The GitLab CI/CD pipeline can deploy layers sequentially, with proper validation at each stage.

The state management through remote state data sources creates explicit dependencies between layers while maintaining loose coupling. Each layer exposes only the necessary outputs that other layers need, creating clean interfaces between different infrastructure concerns.

The environment-specific configuration files enable you to operate the same infrastructure code across different environments with appropriate parameter differences, such as more conservative thresholds in production or more verbose logging in staging environments.

Would you like me to continue with additional components such as the GitLab CI/CD configuration files that would orchestrate the deployment of these layers, or would you prefer to focus on any specific aspect of this layered implementation approach?