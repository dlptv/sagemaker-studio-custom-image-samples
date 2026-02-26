# PyTorch Computer Vision Custom Image

A single custom image with PyTorch and computer vision libraries that works across SageMaker Studio V2 and Unified Studio.

## What's included

The base image ([SageMaker Distribution](https://github.com/aws/sagemaker-distribution) 3.5.1) already includes PyTorch, OpenCV, NumPy, and Pandas. This example adds:

- **albumentations** - Image augmentation library
- **timm** - Pre-trained vision models (ViT, EfficientNet, ConvNeXt, etc.)
- **ultralytics** - YOLOv8 for object detection and segmentation

## Key design: single image, multiple platforms

A single Docker image works across SageMaker Studio platforms by:
1. Not specifying `ENTRYPOINT` in the Dockerfile
2. Configuring entrypoints via `AppImageConfig` at attach time

| Platform | Entrypoint | AppImageConfig suffix |
|----------|-----------|----------------------|
| Studio V2 JupyterLab | `jupyter-lab` | `-jl` |
| Unified Studio JupyterLab | `entrypoint-sagemaker-ui-jupyter-server` | `-unified-jl` |
| Code Editor (both) | `entrypoint-code-editor` | `-ce` |

All three entrypoints ship with the sagemaker-distribution base image.

## Prerequisites

- AWS CLI configured with appropriate permissions
- Docker installed and running locally
- An ECR repository (or permission to create one)

## Setup

Run this block first. Put the `aws sts` call last — it is slow and can cause shells to drop pasted lines that follow it:

```bash
export AWS_REGION=us-west-2          # change to your region
export ECR_REPO=sagemaker-custom-images/pytorch-cv
export IMAGE_NAME=pytorch-cv
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "Account: $AWS_ACCOUNT_ID  Region: $AWS_REGION  Repo: $ECR_REPO"
```

Verify the output shows the correct values before proceeding.

## Build and push

```bash
# Create ECR repository (skip if it already exists)
aws ecr create-repository --repository-name $ECR_REPO --region $AWS_REGION || true

# Build (--platform linux/amd64 is required when building on Apple Silicon)
docker build --platform linux/amd64 -t $IMAGE_NAME .

# Login to ECR
aws ecr get-login-password --region $AWS_REGION | \
    docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

# Tag and push
docker tag $IMAGE_NAME $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/${ECR_REPO}:latest
docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/${ECR_REPO}:latest
```

## Register with SageMaker

```bash
export ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/YourSageMakerRole"  # change to your execution role

# Create SageMaker Image
aws sagemaker create-image \
    --image-name $IMAGE_NAME \
    --role-arn $ROLE_ARN \
    --region $AWS_REGION

# Create Image Version
aws sagemaker create-image-version \
    --image-name $IMAGE_NAME \
    --base-image $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/${ECR_REPO}:latest \
    --region $AWS_REGION

# AppImageConfig for Studio V2 JupyterLab
aws sagemaker create-app-image-config \
    --app-image-config-name ${IMAGE_NAME}-jl \
    --jupyter-lab-app-image-config '{
        "ContainerConfig": {
            "ContainerEntrypoint": ["jupyter-lab"],
            "ContainerArguments": ["--ServerApp.base_url=/jupyterlab/default", "--ServerApp.ip=0.0.0.0", "--ServerApp.token=", "--no-browser"]
        }
    }' \
    --region $AWS_REGION

# AppImageConfig for Code Editor (works on both Studio V2 and Unified Studio)
aws sagemaker create-app-image-config \
    --app-image-config-name ${IMAGE_NAME}-ce \
    --code-editor-app-image-config '{
        "ContainerConfig": {
            "ContainerEntrypoint": ["entrypoint-code-editor"]
        }
    }' \
    --region $AWS_REGION

# AppImageConfig for Unified Studio JupyterLab
aws sagemaker create-app-image-config \
    --app-image-config-name ${IMAGE_NAME}-unified-jl \
    --jupyter-lab-app-image-config '{
        "ContainerConfig": {
            "ContainerEntrypoint": ["entrypoint-sagemaker-ui-jupyter-server"]
        }
    }' \
    --region $AWS_REGION
```

## Attach to a domain

### Studio V2

```bash
export STUDIO_DOMAIN_ID=d-xxxxxxxxxxxx   # change to your Studio domain ID

aws sagemaker update-domain \
    --domain-id $STUDIO_DOMAIN_ID \
    --region $AWS_REGION \
    --default-user-settings '{
        "JupyterLabAppSettings": {
            "CustomImages": [{
                "ImageName": "'"$IMAGE_NAME"'",
                "AppImageConfigName": "'"${IMAGE_NAME}-jl"'"
            }]
        },
        "CodeEditorAppSettings": {
            "CustomImages": [{
                "ImageName": "'"$IMAGE_NAME"'",
                "AppImageConfigName": "'"${IMAGE_NAME}-ce"'"
            }]
        }
    }'
```

### Unified Studio

The SageMaker domain ID is provisioned by the DataZone Tooling environment. Find it in the environment's provisioned resources (`sageMakerDomainId`).

```bash
export UNIFIED_DOMAIN_ID=d-xxxxxxxxxxxx    # change to your Unified Studio domain ID

aws sagemaker update-domain \
    --domain-id $UNIFIED_DOMAIN_ID \
    --region $AWS_REGION \
    --default-user-settings '{
        "JupyterLabAppSettings": {
            "CustomImages": [{
                "ImageName": "'"$IMAGE_NAME"'",
                "AppImageConfigName": "'"${IMAGE_NAME}-unified-jl"'"
            }]
        },
        "CodeEditorAppSettings": {
            "CustomImages": [{
                "ImageName": "'"$IMAGE_NAME"'",
                "AppImageConfigName": "'"${IMAGE_NAME}-ce"'"
            }]
        }
    }'
```

## Validate

After launching a space with your custom image:

```bash
python validate.py
```

## Related resources

- [Custom image specs for Studio V2](https://docs.aws.amazon.com/sagemaker/latest/dg/studio-updated-byoi-specs.html)
- [Dockerfile specs for Unified Studio](https://docs.aws.amazon.com/sagemaker-unified-studio/latest/userguide/byoi-specifications.html)
- [SageMaker Distribution on GitHub](https://github.com/aws/sagemaker-distribution)
