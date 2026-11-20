pipeline {
    agent { label 'kubernetes' }

    options {
        disableConcurrentBuilds()
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    stages {
        stage('Build and push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                        set +x
                        export DOCKER_CONFIG="$(mktemp -d)"
                        trap 'rm -rf "$DOCKER_CONFIG"' EXIT

                        printf '%s' "$DOCKERHUB_TOKEN" |
                          docker login --username "$DOCKERHUB_USER" --password-stdin

                        IMAGE="docker.io/$DOCKERHUB_USER/beginner-website:$(git rev-parse --short=12 HEAD)-$BUILD_NUMBER"
                        docker build -t "$IMAGE" .
                        docker push "$IMAGE"
                        printf '%s\\n' "$IMAGE" > image.txt
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    IMAGE="$(cat image.txt)"
                    kubectl set image -f deployment.yml website="$IMAGE" \
                      --local -o yaml > rendered-deployment.yml
                    kubectl apply -f rendered-deployment.yml -f service.yml
                    kubectl rollout status deployment/website --timeout=180s
                    kubectl get deployment website
                    kubectl get service website
                '''
            }
        }
    }
}
