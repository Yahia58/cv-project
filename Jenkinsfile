pipeline {
    agent any
    environment {
        DOCKERHUB = 'yahya080'
        IMAGE_NAME = "${DOCKERHUB}/cv"
        CRED_ID = 'dockerhub-creds'
        GIT_BACKUP = 'github-creds'
        BACKUP_REPO = 'https://github.com/yourusername/cv-backups.git'
    }

    stages {
        stage('Backup Old Container') {
            steps {
                sh '''
                    CONTAINER_NAME=cv-container
                    BACKUP_DIR=/tmp/cv-backup-$(date +%F-%H-%M)

                    if [ "$(docker ps -q -f name=$CONTAINER_NAME)" ]; then
                      mkdir -p $BACKUP_DIR
                      docker cp $CONTAINER_NAME:/usr/share/nginx/html $BACKUP_DIR/

                      cd $BACKUP_DIR
                      git init
                      git config user.name "jenkins"
                      git config user.email "jenkins@example.com"
                      git checkout -b backup-$(date +%F-%H-%M)
                      git add .
                      git commit -m "Backup from $CONTAINER_NAME at $(date)"
                      git remote add origin $BACKUP_REPO

                      git push -u origin backup-$(date +%F-%H-%M)
                    fi
                '''
            }
        }

        stage('Deploy New Version') {
            steps {
                sh '''
                    docker pull $IMAGE_NAME:latest
                    docker rm -f cv-container || true
                    docker run -d -it -p 555:80 --name cv-container $IMAGE_NAME:latest
                '''
            }
        }
    }
}
