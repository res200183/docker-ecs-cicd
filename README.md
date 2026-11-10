AWS DevOps Project: Docker → ECS Fargate + CI/CD

Overview

This project demonstrates the design, implementation, and testing of an automated CI/CD pipeline for a containerized Python Flask application deployed to Amazon ECS Fargate.

The solution uses GitHub, AWS CodePipeline, AWS CodeBuild, Amazon ECR, Docker, Amazon ECS Fargate, and an Application Load Balancer.

The main objective was to gain practical experience with containerization, automated testing, AWS infrastructure, IAM permissions, CI/CD artifacts, deployment automation, monitoring, and troubleshooting.

The project was designed around a cost-conscious AWS architecture.

Architecture

Developer
    |
    | git push origin main
    v
GitHub Repository
    |
    | AWS CodeConnections
    v
AWS CodePipeline
    |
    | SourceArtifact
    v
AWS CodeBuild
    |
    | Install Python dependencies
    | Run pytest
    | Build Docker image
    | Generate unique image tag
    | Push image to ECR
    | Generate imagedefinitions.json
    v
Amazon ECR
    |
    | BuildArtifact
    v
AWS CodePipeline - Deploy
    |
    v
Amazon ECS Fargate
    |
    | ECS Service
    | Task Definition
    | Docker Container
    v
Application Load Balancer
    |
    v
Flask Application
    |
    +--- GET /
    |
    +--- GET /health

Technology Stack

Technology| Purpose
GitHub| Source control
Git| Version management
Python| Application development
Flask| Web application
pytest| Automated testing
Docker| Containerization
Gunicorn| Application server
AWS CodeConnections| GitHub integration
AWS CodePipeline| CI/CD orchestration
AWS CodeBuild| Build and test automation
Amazon ECR| Container image registry
Amazon ECS| Container orchestration
AWS Fargate| Serverless container compute
Application Load Balancer| HTTP traffic routing
Amazon VPC| Networking
AWS IAM| Access control
Amazon CloudWatch| Logs and monitoring
AWS CLI| Infrastructure verification
Bash| Build automation

Project Structure

docker-ecs-cicd/
├── app.py
├── Dockerfile
├── buildspec.yml
├── requirements.txt
├── tests/
│   └── test_app.py
├── .dockerignore
├── .gitignore
└── README.md

Project Implementation

The original implementation plan contains 40 tasks organized into five stages.

Stage 1 — Planning and Cost Control

The project was designed with the following objectives:

- Use a single AWS Region.
- Monitor AWS spending.
- Apply consistent resource naming.
- Avoid unnecessary NAT Gateway and VPC endpoint costs.
- Use a cost-conscious public subnet architecture.
- Prepare Git, Python, Docker, and AWS CLI.

AWS Region used: "us-east-1" (N. Virginia).

The architecture was planned to minimize unnecessary infrastructure costs during development and testing.

Stage 2 — Flask Application and Docker

A Python Flask application was developed with two endpoints.

Application endpoint

GET /

Response after the final CI/CD test:

{
  "message": "CI/CD automatic deployment SUCCESS!",
  "status": "running"
}

Health check endpoint

GET /health

Response:

{
  "status": "healthy"
}

The application uses port "8080" and runs with Gunicorn in the container.

Automated tests were implemented with pytest.

Verified local test results:

tests/test_app.py::test_home PASSED
tests/test_app.py::test_health PASSED

2 passed in 0.19s

Stage 3 — AWS Build Infrastructure

An Amazon ECR repository was used to store application Docker images.

Repository:

docker-release-app

AWS CodeBuild was configured to:

1. Install Python dependencies.
2. Execute pytest.
3. Authenticate to Amazon ECR.
4. Build the Docker image.
5. Generate a unique image tag.
6. Push the image to ECR.
7. Generate the ECS deployment artifact.

Docker image versioning

The build process uses the source revision and CodeBuild build number:

COMMIT_SHA="${CODEBUILD_RESOLVED_SOURCE_VERSION:?Source revision is missing}"
IMAGE_TAG="${COMMIT_SHA}-${CODEBUILD_BUILD_NUMBER:?Build number is missing}"
IMAGE_URI="${ECR_REPOSITORY_URI}:${IMAGE_TAG}"

This approach avoids relying on the mutable "latest" tag and helps trace deployments to specific builds.

ECS deployment artifact

CodeBuild generates "imagedefinitions.json".

Example:

[
  {
    "name": "docker-release-container",
    "imageUri": "<account-id>.dkr.ecr.us-east-1.amazonaws.com/docker-release-app:<image-tag>"
  }
]

The container name must match the container definition in the ECS task definition.

CodePipeline passes this artifact from the Build stage to the ECS Deploy stage.

Stage 4 — Networking and Amazon ECS

The deployment uses Amazon ECS Fargate.

AWS resources include:

ECS Cluster

docker-release-cluster

ECS Service

docker-release-task-service

Task Definition

docker-release-task

Container

docker-release-container

The ECS service was verified with:

Running tasks: 1
Desired tasks: 1
Deployment status: COMPLETED

A deployed task definition revision was also inspected using AWS CLI.

The Application Load Balancer provides access to the application.

A successful HTTP request returned:

HTTP/1.1 200 OK
Content-Type: application/json
Server: gunicorn

The Flask application also provides a dedicated health check endpoint.

Stage 5 — Automated CI/CD Deployment

The final CI/CD pipeline was created with three main stages.

Source

GitHub repository:

res200183/docker-ecs-cicd

Branch:

main

Build

CodeBuild project:

docker-release-build

Deploy

Amazon ECS standard deployment to:

docker-release-cluster
docker-release-task-service

Pipeline:

docker-release-pipeline

The pipeline was tested by changing the Flask application's response, committing the change, and pushing it to GitHub.

git add app.py
git commit -m "Test automatic ECS deployment"
git push origin main

The pipeline executed the new source revision and completed the deployment.

Verified deployment commit:

e3c74d52

The updated application response was subsequently confirmed.

Troubleshooting

The implementation involved resolving real AWS configuration problems.

Issue 1 — IAM PassRole Access Denied

CodePipeline failed to start CodeBuild because the CodePipeline service role did not have permission to pass the selected CodeBuild service role.

Error:

Not authorized to perform iam:PassRole

Resolution:

Updated the IAM permissions required for the CodePipeline-to-CodeBuild integration.

Issue 2 — ECS Container Name Mismatch

The ECS deployment initially failed with:

The AWS ECS container web does not exist

Root cause:

The container name in "imagedefinitions.json" did not match the ECS task definition.

Resolution:

Updated "buildspec.yml" to generate the correct container name:

docker-release-container

After committing and pushing the correction, the pipeline successfully deployed the application.

Issue 3 — Deployment Verification

The deployed ECS service and task definition were inspected using AWS CLI.

Verification included:

- ECS service running count.
- ECS desired count.
- Deployment rollout status.
- Task definition revision.
- Container name.
- ECR image URI.
- Application HTTP response.

Verified Results

The following capabilities were successfully demonstrated:

- GitHub source integration.
- Automated pipeline triggering after a push.
- CodePipeline Source, Build, and Deploy stages.
- Automated Python testing.
- Docker image builds.
- Unique image tagging.
- Amazon ECR image publishing.
- CodePipeline build artifact generation.
- ECS task definition revision updates.
- Automated ECS Fargate deployment.
- Successful ALB HTTP response.
- Successful application update verification.
- Practical troubleshooting of IAM and ECS deployment errors.

Additional Validation and Cleanup

The original project plan also includes:

- AWS Budget verification.
- ECR immutable tag and lifecycle policy verification.
- Least-privilege IAM review.
- CloudWatch log retention verification.
- Security group restriction verification.
- ECS deployment circuit breaker and rollback testing.
- ECS Service Auto Scaling testing.
- Failure simulation for tests and health checks.
- Removal of chargeable resources after testing.
- Final AWS Billing review.

These tasks should be verified separately before marking the entire 40-task plan as completed.

Future Improvements

Potential extensions include:

- Blue/Green deployments with AWS CodeDeploy.
- CloudWatch alarms and SNS notifications.
- Infrastructure as Code using AWS CloudFormation.
- Automated integration tests.
- Deployment approval stages.
- Enhanced security scanning.
- Automated infrastructure cleanup.

Key Skills Demonstrated

This project provided hands-on experience in:

- AWS CI/CD pipeline implementation.
- Docker containerization.
- Amazon ECS Fargate deployments.
- IAM service role troubleshooting.
- Build automation with "buildspec.yml".
- Artifact management between pipeline stages.
- Docker image versioning.
- Application health checking.
- AWS CLI troubleshooting.
- End-to-end deployment validation.

Conclusion

This project demonstrates a working automated deployment pipeline for a containerized Flask application on AWS.

A code change pushed to GitHub can trigger testing, Docker image creation, image publishing, and ECS Fargate deployment without manually rebuilding or redeploying the application.

The project represents practical hands-on experience with core AWS DevOps technologies and CI/CD workflows.
