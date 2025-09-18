pipeline {
    agent any

    environment {
        DOCKERHUB   = 'yahya080'
        IMAGE_NAME  = "${DOCKERHUB}/cv"
        CRED_ID     = 'yahyadockerhub'   // DockerHub credentials ID
        GIT_BACKUP  = 'yahyagithub'      // GitHub credentials ID (لو هتعمل Backup)
        BACKUP_REPO = 'https://github.com/yourusername/cv-backups.git'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Yahia58/cv-project.git',
                    credentialsId: "${GIT_BACKUP}"
            }
        }

        stage('Build & Push Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${CRED_ID}",
                                                 usernameVariable: 'DOCKER_USER',
                                                 passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
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
                    BACKUP_DIR=/tmp/cv-backup-$(date +%F-%H-%M)
                    if [ "$(docker ps -q -f name=$CONTAINER_NAME)" ]; then
                        mkdir -p $BACKUP_DIR
                        docker export $(docker ps -q -f name=$CONTAINER_NAME) > $BACKUP_DIR/container-backup.tar
                        docker stop $CONTAINER_NAME
                        docker rm $CONTAINER_NAME
                    fi
                '''
            }
        }

        stage('Deploy New Version') {
            steps {
                sh '''
                    docker run -d --name cv-container -p 8080:80 $IMAGE_NAME:latest
                '''
            }
        }

        // Stage احتياطي لو حابب تعمل push للـ backup repo (اختياري)
        stage('Push Backup to GitHub') {
            when {
                expression { return false } // خليها true لو عايز تشغل الباكاب
            }
            steps {
                withCredentials([usernamePassword(credentialsId: "${GIT_BACKUP}",
                                                 usernameVariable: 'GIT_USER',
                                                 passwordVariable: 'GIT_PASS')]) {
                    sh '''
                        cd /tmp
                        BACKUP_DIR=cv-backup-$(date +%F-%H-%M)
                        git clone https://${GIT_USER}:${GIT_PASS}@${BACKUP_REPO} repo
                        cp -r $BACKUP_DIR/* repo/
                        cd repo
                        git add .
                        git commit -m "Backup on $(date)"
                        git push
                    '''
                }
            }
        }
    }
}
