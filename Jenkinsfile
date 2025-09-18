pipeline {
    agent any
    environment {
        DOCKERHUB   = 'yahya080'
        IMAGE_NAME  = "${DOCKERHUB}/cv"
        CRED_ID     = 'yahyadockerhub' // DockerHub credentials ID
        GIT_BACKUP  = 'yahiagithub'    // GitHub credentials ID
        BACKUP_REPO = 'https://github.com/Yahia58/cv-backups.git'
    }

    triggers {
        githubPush()   // ✅ ده اللي يخلي Jenkins يشتغل تلقائي عند أي push
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: "${GIT_BACKUP}",
                    url: 'https://github.com/Yahia58/cv-project.git'
            }
        }

        stage('Build & Push Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${CRED_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker build -t $IMAGE_NAME:latest .
                        docker push $IMAGE_NAME:latest
                    '''
                }
            }
        }

        stage('Backup Old Container') {
            steps {
                sh '''
                    CONTAINER_NAME=cv-container
                    TIMESTAMP=$(date +%F-%H-%M)
                    BACKUP_DIR=/tmp/cv-backup-$TIMESTAMP

                    if [ "$(docker ps -q -f name=$CONTAINER_NAME)" ]; then
                        mkdir -p $BACKUP_DIR
                        docker cp $CONTAINER_NAME:/usr/share/nginx/html/index.html $BACKUP_DIR/index.html || true
                        echo "Backup created at $BACKUP_DIR"
                    else
                        echo "No old container found, skipping backup."
                    fi
                '''
            }
        }

        stage('Deploy New Version') {
            steps {
                sh '''
                    CONTAINER_NAME=cv-container
                    if [ "$(docker ps -aq -f name=$CONTAINER_NAME)" ]; then
                        docker rm -f $CONTAINER_NAME
                    fi

                    docker run -d --name $CONTAINER_NAME -p 8090:80 $IMAGE_NAME:latest
                '''
            }
        }

        stage('Push Backup to GitHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${GIT_BACKUP}",
                                                 usernameVariable: 'GIT_USER',
                                                 passwordVariable: 'GIT_PASS')]) {
                    sh '''
                        set -e
                        TIMESTAMP=$(date +%F-%H-%M)
                        BACKUP_DIR=$(ls -dt /tmp/cv-backup-* | head -1)

                        if [ -d "$BACKUP_DIR" ]; then
                            rm -rf /tmp/cv-backups
                            git clone https://${GIT_USER}:${GIT_PASS}@github.com/Yahia58/cv-backups.git /tmp/cv-backups
                            cd /tmp/cv-backups

                            git config user.email "yahia@example.com"
                            git config user.name "Yahia Jenkins"

                            if ! git rev-parse --verify main >/dev/null 2>&1; then
                                git checkout -b main
                            else
                                git checkout main
                            fi

                            ZIP_FILE="backup-$TIMESTAMP.zip"
                            zip -r $ZIP_FILE $BACKUP_DIR
                            git add $ZIP_FILE
                            git commit -m "Backup on $TIMESTAMP" || echo "No changes to commit"
                            git push origin main
                        else
                            echo "No backup found!"
                        fi
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished."
        }
    }
}
