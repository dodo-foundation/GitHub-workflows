```yml
---
trigger: none
resources:
  - repo: self

pool:
  name: "Hosted Ubuntu 1604"

variables:
  buildConfiguration: "Release"
  buildPlatform: "AnyCPU"

stages:
  - stage: bd
    displayName: "BUILD"
    jobs:
      - job: "BUILDPROCESS"
        displayName: BUILD
        # pool:
        #   vmImage: "ubuntu-latest"
        steps:
          - task: Docker@2
            displayName: Build Docker Image
            inputs:
              command: build
              containerRegistry: containerRegistry
              repository: repo
              Dockerfile: zookeeper/docker/Dockerfile
              tags: $(Build.BuildNumber)
              arguments: "--no-cache"

          - script: |
              docker run -d --name ${{ parameters.zk }}  ${{ parameters.repositoryName }}/${{ parameters.zk }}:$(Build.BuildNumber)
              CID=$(docker ps -q -f status=running -f name=${{ parameters.zk }})
              if [ ! "${CID}" ]; then
                echo "Container doesn't exist"
                exit 1
              else
                echo "Container Running"
              fi
              unset CID
            displayName: Verify Docker Image Running State

          - script: |
              docker rm $(docker ps -aq --filter name=${{ parameters.zk }})  -f
            displayName: Delete Running State Container

          - task: Docker@2
            displayName: Push Docker Image to ACR
            inputs:
              command: push
              containerRegistry: containerRegistry
              repository: repo
              tags: $(Build.BuildNumber)

          - script: |
              Size=$(docker image inspect ${{ parameters.repositoryName }}/${{ parameters.zk }}:$(Build.BuildNumber) --format='{{.Size}}')
              DockerSize=$((($Size/1024)/1024))
              echo "${{ parameters.repositoryName }}/${{ parameters.zk }}:$(Build.BuildNumber) image size: $DockerSize Mb"
              unset Size
              docker rmi ${{ parameters.repositoryName }}/${{ parameters.zk }}:$(Build.BuildNumber)
            displayName: Remove Last Build Image

          - script: |
              tag=$(Build.BuildNumber)
              imageNameandversion=${{ parameters.zk }}:$tag
              repositoryName=${{ parameters.repositoryName }}
              sed -i 's/containerImageName/'"$repositoryName\/$imageNameandversion"'/g' zookeeper/manifests/zoo-keeper.yml
            displayName: Preparing the k8s deployment file

          - task: PublishPipelineArtifact@1
            inputs:
              targetPath: zookeeper/manifests/zoo-keeper.yml
              artifact: drop-zk
            displayName: Publish Pipeline Artifact

          - task: KubernetesManifest@0
            displayName: kubernetes-deploy
            inputs:
              kubernetesServiceConnection: .kubernetesServiceConnection
              namespace: default
              manifests: |
                zookeeper/manifests/zoo-keeper.yml

```