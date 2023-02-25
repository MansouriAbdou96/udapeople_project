def addAWSCredentials() {
        sh 'aws configure set aws_access_key_id "${AWS_ACCESS_KEY_ID}"'
        sh 'aws configure set aws_secret_access_key "${AWS_SECRET_ACCESS_KEY}"'
        sh 'aws configure set aws_session_token "${AWS_SESSION_TOKEN}"'
        sh 'aws configure set region us-east-1'
}
def destroyEnvironment(){

        sh '''
             aws cloudformation delete-stack --stack-name udapeople-backend-${BUILD_ID}
             aws s3 rm "s3://udapeople-${BUILD_ID}" --recursive
             aws cloudformation delete-stack --stack-name udapeople-frontend-${BUILD_ID}
        '''
}
def migrationEnv(){
    sh '''
        export TYPEORM_CONNECTION=${TYPEORM_CONNECTION}
        export TYPEORM_MIGRATIONS_DIR=${TYPEORM_MIGRATIONS_DIR}
        export TYPEORM_ENTITIES=${TYPEORM_ENTITIES}
        export TYPEORM_MIGRATIONS=${TYPEORM_MIGRATIONS}
        export TYPEORM_HOST=${TYPEORM_HOST}
        export TYPEORM_PORT=${TYPEORM_PORT}
        export TYPEORM_USERNAME=${TYPEORM_USERNAME}
        export TYPEORM_PASSWORD=${TYPEORM_PASSWORD}
        export TYPEORM_DATABASE=${TYPEORM_DATABASE}
    '''
}
def revertMigration(){

    migrationEnv()

    sh '''
        #!/bin/bash
        SUCCESS=$(curl --insecure https://kvdb.io/JtRRna2uqYBZYR8JUF26ZT/migration_${BUILD_ID})
        if [ $SUCCESS == 1 ]; then
            cd backend
            npm install
            npm run migrations:revert
        fi
    '''
}


pipeline{

        agent {
                label 'docker-myAgent'
        }

    stages {

        stage('deploy-infra') {
           steps{  
                addAWSCredentials()
            
                sh'''
                    # create a backend infrastructure 
                    
                    aws cloudformation deploy \
                    --template-file files/backend.yml \
                    --stack-name "udapeople-backend-${BUILD_ID}" \
                    --parameter-overrides ID="${BUILD_ID}"  \
                    --region=us-east-1 \
                    --tags project=udapeople
                '''    


                sh'''
                    # create a frontend infrastructure
                    aws cloudformation deploy \
                        --template-file files/frontend.yml \
                        --tags project=udapeople \
                        --stack-name "udapeople-frontend-${BUILD_ID}" \
                        --parameter-overrides ID="${BUILD_ID}" \
                        --region=us-east-1
                '''

                sh'''
                    # Adding Backend Ip to ansible inventory
                    
                    cd ansible
                    echo "[web]" >> inventory.txt
                    aws ec2 describe-instances --filters "Name=tag:Name,Values=backend-${BUILD_ID}" \
                        --query "Reservations[*].Instances[*].PublicIpAddress" \
                        --region=us-east-1 \
                        --output text >> inventory.txt
                '''

                sh 'cat ansible/inventory.txt'

                archiveArtifacts artifacts: 'ansible/inventory.txt'
              
           }
           
           post {
                   failure {
                        destroyEnvironment()
                   }
           }
            
        }

       stage('configure-infra') {

            steps {

            withCredentials([sshUserPrivateKey(credentialsId: 'ec2_key_pair', keyFileVariable: 'PRIVATE_KEY_FILE')]){

                    dir('ansible'){
                        migrationEnv()
                        sh'''
                                ansible-playbook -i inventory.txt configure-server.yml --private-key=$PRIVATE_KEY_FILE -u ubuntu
                        '''
                    }    
            }                
            }
            
            post {
                   failure {
                        destroyEnvironment()
                   }
           }

       }

       stage('run-migration'){
            steps{
                dir('backend'){
                    sh 'npm install && npm run build'
                    migrationEnv()
                    sh 'npm run migrations > migrations_dump.txt'
                    sh 'cat migrations_dump.txt'

                    sh'''
                    # Check the output for the success message
                    if grep -q "has been executed successfully." migrations_dump.txt
                    then
                        # Send a "1" to indicate success to the key-value store
                        curl --insecure https://kvdb.io/JtRRna2uqYBZYR8JUF26ZT/migration_${BUILD_ID} -d '1'
                    fi

                    '''
                }

                
            }
            post {
                failure {
                    destroyEnvironment()
                    revertMigration()
                }
            }
       }

       stage('deploy-frontend'){
            steps {
                addAWSCredentials()

                sh '''
                    export BACKEND_IP=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=backend-${BUILD_ID}" \
                        --query "Reservations[*].Instances[*].PublicIpAddress" \
                        --output text)
                    export API_URL="http://${BACKEND_IP}:3030"
                    echo "API_URL = ${API_URL}"
                    echo API_URL="http://${BACKEND_IP}:3030" >> frontend/.env
                    cat frontend/.env
                '''

                dir('frontend'){
                    sh '''
                        npm install
                        npm run build
                        tar -czvf artifact-"${BUILD_ID}".tar.gz dist
                        aws s3 cp dist s3://udapeople-${BUILD_ID} --recursive
                    '''
                }
                
            }
            post {
                failure {
                    destroyEnvironment()
                    revertMigration()
                }
            }
       }

       stage('deploy-backend'){
            steps{

                dir('backend'){
                    sh 'npm i'
                    sh 'npm run build'
                }  

                withCredentials([sshUserPrivateKey(credentialsId: 'ec2_key_pair', keyFileVariable: 'PRIVATE_KEY_FILE')]){
                    sh '''
                        # Zip the directory
                        tar -C backend -czvf artifact.tar.gz .
                    '''

                    dir('ansible'){
                        migrationEnv()

                        sh '''
                            echo "Contents  of the inventory.txt file is -------"
                            cat inventory.txt
                            ansible-playbook -i inventory.txt deploy-backend.yml --private-key=$PRIVATE_KEY_FILE -u ubuntu
                        '''
                    }
                } 
            }
            post {
                failure {
                    destroyEnvironment()
                    revertMigration()
                }
            }
       }

        stage('smoke-test'){
            steps {
                addAWSCredentials()

                sh ''' 
                    # BACKEND SMOKE TEST

                    export BACKEND_IP=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=backend-${BUILD_ID}" \
                        --query "Reservations[*].Instances[*].PublicIpAddress" \
                        --output text)
                    export API_URL="http://${BACKEND_IP}:3030"
                    echo "${API_URL}"

                    # Set maximum number of retries
                    MAX_RETRIES=10
                    RETRY_COUNT=0

                    # Wait for API to be ready
                    while true; do
                        if curl "${API_URL}/api/status" | grep "ok"; then
                            break
                        fi

                        RETRY_COUNT=$((RETRY_COUNT + 1))

                        if [ ${RETRY_COUNT} -eq ${MAX_RETRIES} ]; then
                            echo "Backend API is not ready after ${MAX_RETRIES} retries."
                            exit 1
                        fi

                        # Wait for 5 seconds before retrying
                        sleep 5
                    done
                '''

                sh ''' 
                    # FRONTEND SMOKE TEST

                    URL="http://udapeople-${BUILD_ID}.s3-website-us-east-1.amazonaws.com/#/employees"            
                    echo ${URL} 
                    if curl -s ${URL} | grep "Welcome"; then
                        exit 0
                    else
                        exit 1
                    fi
                '''
            }
            post {
                failure {
                    destroyEnvironment()
                    revertMigration()
                }
            }
        }

        stage('cloudfront-update'){
            steps {
                addAWSCredentials()

                dir('ansible'){
                    sh''' 
                        # Fetch the Old build ID
                        aws cloudformation \
                        list-exports --query "Exports[?Name=="WorkflowID"].Value" \
                            --no-paginate --output text > OldBuildID.txt
                        echo JENKINS BUILD ID "${BUILD_ID}"
                        cat OldBuildID.txt
                    '''
                }

                dir('files'){
                    sh ''' 
                        # Update the cloudFront
                        aws cloudformation deploy \
                            --template-file cloudfront.yml \
                            --stack-name InitialStack\
                            --parameter-overrides WorkflowID="${BUILD_ID}"
                    '''
                }
                
                archiveArtifacts artifacts: 'ansible/OldBuildID.txt'
            }
            post {
                failure {
                    // destroyEnvironment()
                    revertMigration()
                }
            }
        }

        stage('cleanup'){
            steps {
                addAWSCredentials()

                dir('ansible'){
                    sh ''' 
                        export OldBuildID=$(cat OldBuildID.txt)
                        echo OldBuildID: "${OldBuildID}"

                        if [[ -z "${OldBuildID}" ]]; then
                            OldBuildID="${BUILD_ID}"
                        fi
                    
                        if [[ "${BUILD_ID}" != "${OldBuildID}" ]]
                        then
                            aws s3 rm "s3://udapeople-${OldBuildID}" --recursive
                            aws cloudformation delete-stack --stack-name "udapeople-frontend-${OldBuildID}"
                            aws cloudformation delete-stack --stack-name "udapeople-backend-${OldBuildID}"
                        fi
                    '''
                }
            }
        }

        stage('config-prometheus-node-exporter'){
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2_key_pair', keyFileVariable: 'PRIVATE_KEY_FILE')]){
                    
                    dir('ansible'){
                    sh ''' 
                        cat inventory.txt
                        ansible-playbook -i inventory.txt node-exporter.yml --private-key=$PRIVATE_KEY_FILE -u ubuntu
                    '''
                    }
                }
                
            }
        }

    }

}
