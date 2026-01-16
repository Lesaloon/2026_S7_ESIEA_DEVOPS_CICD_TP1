# CICD

this project is a CI CD school project.

the goal is to put on a ftp server a zip with the manifest files for a k8s deployment.

## CICD Pipeline

we check the docker compose
we build the docker image and push it to ghcr.io
we use kompose to generate the k8s manifest files
we zip the manifest files
we put the zip on an ftp server

## Usage

just commit !
