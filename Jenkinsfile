pipeline {
    agent any
    environment {
        DOCKERHUB   = 'yahya080'
        IMAGE_NAME  = "${DOCKERHUB}/cv"
        CRED_ID     = 'yahyadockerhub' // DockerHub credentials ID
        GIT_BACKUP  = 'yahiagithub'    // GitHub credentials ID
        BACKUP_REPO = 'https://github.com/Yahia58/cv-backups.git'
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
                    # Remove old container if exists
                    if [ "$(docker ps -aq -f name=$CONTAINER_NAME)" ]; then
                        docker rm -f $CONTAINER_NAME
                    fi

                    docker run -d --name $CONTAINER_NAME -p 8090:80 $IMAGE_NAME:latest
                '''
            }
        }

        stage('Push Backup to GitHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${GIT_BACKUP}", usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                    sh '''
                        TIMESTAMP=$(date +%F-%H-%M)
                        BACKUP_DIR=$(ls -dt /tmp/cv-backup-* 2>/dev/null | head -1)

                        if [ -d "$BACKUP_DIR" ]; then
                            cd /tmp
                            if [ ! -d cv-backups ]; then
                                git clone https://$GIT_USER:$GIT_PASS@github.com/Yahia58/cv-backups.git
                            fi
                            cd cv-backups
                            cp -r $BACKUP_DIR/* .
                            git add .
                            git commit -m "Backup on $TIMESTAMP" || true
                            git push origin main
                        else
                            echo "No backup found to push."
                        fi
                    '''
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished. Backup stage executed (if backup existed)."
        }
    }
}
