# Tasky Deployment on AWS EKS

Tasky is a todo list application written with Go. 

This project deploys Tasky in a lab environment prepared on AWS EKS. It uses a MongoDB back-end which is deployed on AWS EC2.

The kube-manifests folder contains the yaml files used to deploy Tasky on EKS. The deployment runs a single pod and a service of type LoadBalancer exposes Tasky to the Internet.

## Docker
A Dockerfile has been provided to run this application.  The default port exposed is 8080.

## Environment Variables
The following environment variables are needed.
|Variable|Purpose|example|
|---|---|---|
|`MONGODB_URI`|Address to mongo server|`mongodb://servername:27017` or `mongodb://username:password@hostname:port` or `mongodb+srv://` schema|
|`SECRET_KEY`|Secret key for JWT tokens|`secret123`|

Alternatively, you can create a `.env` file and load it up with the environment variables.

## Running with Go

Clone the repository tasky from https://github.com/jeffthorne/tasky into a directory of your choice. Run the command `go mod tidy` to download the necessary packages.

You'll need to add a .env file and add a MongoDB connection string with the name `MONGODB_URI` to access your collection for task and user storage.
You'll also need to add `SECRET_KEY` to the .env file for JWT Authentication.

Run the command `go run main.go` and the project should run on `locahost:8080`

# License

This project is licensed under the terms of the MIT license.

Original project: https://github.com/dogukanozdemir/golang-todo-mongodb
