pipeline {
    agent any
    environment {
        IMAGE_NAME  = "cv"                    // بس اسم الصورة
        DOCKER_CRED = 'yahyadockerhub'        // DockerHub credentials ID
        GIT_CRED    = 'yahiagithub'           // GitHub credentials ID
        BACKUP_REPO = 'https://github.com/<your-repo>/cv-backups.git'
        SOURCE_REPO = 'https://github.com/<your-repo>/cv-project.git'
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: "${GIT_CRED}",
                    url: "${SOURCE_REPO}"
            }
        }

        stage('Build & Push Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CRED}",
                                                 usernameVariable: 'DOCKER_USER',
                                                 passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker build -t $DOCKER_USER/$IMAGE_NAME:latest .
                        docker push $DOCKER_USER/$IMAGE_NAME:latest
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
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CRED}",
                                                 usernameVariable: 'DOCKER_USER',
                                                 passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        CONTAINER_NAME=cv-container
                        if [ "$(docker ps -aq -f name=$CONTAINER_NAME)" ]; then
                            docker rm -f $CONTAINER_NAME
                        fi

                        docker run -d --name $CONTAINER_NAME -p 8090:80 $DOCKER_USER/$IMAGE_NAME:latest
                    '''
                }
            }
        }

        stage('Push Backup to GitHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${GIT_CRED}",
                                                 usernameVariable: 'GIT_USER',
                                                 passwordVariable: 'GIT_PASS')]) {
                    sh '''
                        set -e
                        TIMESTAMP=$(date +%F-%H-%M)
                        BACKUP_DIR=$(ls -dt /tmp/cv-backup-* | head -1)

                        if [ -d "$BACKUP_DIR" ]; then
                            ARCHIVE_NAME="backup-$TIMESTAMP.tar.gz"
                            tar -czf /tmp/$ARCHIVE_NAME -C $BACKUP_DIR .

                            rm -rf /tmp/cv-backups
                            git clone https://${GIT_USER}:${GIT_PASS}@${BACKUP_REPO#https://} /tmp/cv-backups
                            cd /tmp/cv-backups

                            git config user.email "jenkins@example.com"
                            git config user.name "Jenkins Pipeline"

                            git checkout -B main
                            mkdir -p backups/$TIMESTAMP
                            mv /tmp/$ARCHIVE_NAME backups/$TIMESTAMP/

                            git add .
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
            echo "Pipeline finished. Backup stage executed (if backup existed)."
        }
    }
}
