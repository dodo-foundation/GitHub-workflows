```yml
---
trigger: none
resources:
  - repo: self

pool: 
  name: miga
  demands:  
  - Agent.ComputerName -equals access-server

variables:
  tag: $(Build.BuildNumber)
  repositoryName: acriodemoslockdev.azurecr.io
  imageName: api-one

stages:
  - stage: bd
    displayName: "BUILD"
    jobs:
      - job: "BUILDPROCESS"
        displayName: BUILD
        steps:
          - task: Docker@2
            displayName: Build Docker Image
            inputs:
              command: build
              containerRegistry: docker-service
              repository: $(imageName)
              Dockerfile: Dockerfile
              tags: $(tag)
        - script: |
            docker run -d --name $(imageName)  $(repositoryName)/$(imageName):$(Build.BuildNumber)
          displayName: Verify Docker Image Running State
        - script: |
            docker rm $(docker ps -aq --filter name=$(imageName))  -f
          displayName: Delete Running State Container
        - task: Docker@2
          displayName: Push Docker Image to ACR
          inputs:
            command: push
            containerRegistry: docker-service
            repository: $(imageName)
            Dockerfile: source/BackendService/Dockerfile
            tags: $(tag)

        - script: |
            docker rmi $(repositoryName)/$(imageName):$(Build.BuildNumber)
          displayName: Remove Last Build Image


```
