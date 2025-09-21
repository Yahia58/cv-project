pipeline {
    agent any
    environment {
         DOCKERHUB   = credentials('dockerusername')
        
         GITUSERNAME  = credentials('gitusername')
         CRED_ID     = 'yahyadockerhub' // DockerHub credentials ID
         GIT_BACKUP  = 'yahiagithub'    // GitHub credentials ID
         IMAGE_NAME  = "${DOCKERHUB}/cv"   
        BACKUP_REPO = 'https://github.com/${GITUSERNAME}/cv-backups.git'
      
    }

    triggers {
        githubPush()   // ✅ أي push في GitHub repo هيشغل الـ pipeline تلقائي
    }

    stages {
        stage('Checkout') {
            steps {
                 script {
                def repoUrl = "https://github.com/${env.GITUSERNAME}/cv-project.git"
                git branch: 'main',
                credentialsId: "${GIT_BACKUP}",
                url: repoUrl 
            }
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
            when {
                expression { return true } // شغل الباكاب دايمًا
            }
            steps {
                withCredentials([usernamePassword(credentialsId: "${GIT_BACKUP}",
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
                            git clone https://${GIT_USER}:${GIT_PASS}@github.com/${GIT_USER}/cv-backups.git /tmp/cv-backups
                            cd /tmp/cv-backups

                            git config user.email "yahia@example.com"
                            git config user.name "Yahia Jenkins"

                            if ! git rev-parse --verify main >/dev/null 2>&1; then
                                git checkout -b main
                            else
                                git checkout main
                            fi

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
