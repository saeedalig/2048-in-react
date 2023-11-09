## Deploying a React.js 2048 Game using a CI/CD pipeline with Jenkins and deploying it on a Kubernetes cluster. The goal is to automate the build, test, and deployment processes to ensure a streamlined and reliable deployment of the application.

## Workflow of Project
In this project I am utilizing **Git, Jenkins, SonarQube, Maven, Trivy, OWASP, Docker and Kubernetes **. 

The project involves deploying a React.js 2048 game using a comprehensive CI/CD pipeline with several essential tools. It begins with version control using Git, where the source code is stored and tracked. Jenkins, the CI/CD orchestrator, monitors the Git repository for changes and triggers a series of automated steps. The React.js application is built, dependencies are installed, and production-ready bundles are generated. SonarQube is integrated to analyze code quality and ensure adherence to coding standards. Docker is employed to containerize the application. The image built with the help of Docker is then scanned by Trivy to check vulnerabilities. The project also integrates OWASP for security checks, ensuring the game's resilience against potential threats. Docker images are pushed to a container registry, making them available for deployment. Kubernetes is then utilized for continuous deployment, orchestrating and scaling containers. 

The pipeline provides a robust and automated development and deployment process for the React.js 2048 game.
