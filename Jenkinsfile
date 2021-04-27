pipeline {
    agent none
    stages {
        stage('Compile and Test') {
            agent any
            steps {
                sh "mvn --batch-mode package" 
            }
        }

        stage('Publish Tests Results') {
            agent any
            steps {
               echo 'Archive Unit Test Results'
               step([$class: 'JUnitResultArchiver', testResults: 'target/surefire-reports/TEST-*.xml'])
            }
        }
        
       stage('Create and Publish Docker Image'){
            agent any
            steps{
                withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'github', usernameVariable: 'GITHUB_USERNAME', passwordVariable: 'GITHUB_TOKEN']]) {
                    script {
                        env.SHORT_COMMIT= env.GIT_COMMIT[0..7]
                        env.DOCKER_REPOSITORY="docker.pkg.github.com/$GITHUB_USERNAME/pet-clinic/petclinic".toLowerCase()
                        env.TAG_NAME="$DOCKER_REPOSITORY:$SHORT_COMMIT"
                    }
                    sh "docker build -t $TAG_NAME -f Dockerfile.deploy ."
                    sh "echo $GITHUB_TOKEN | docker login https://docker.pkg.github.com -u $GITHUB_USERNAME --password-stdin"
                    sh "docker push $TAG_NAME"
                }
            }
        }

        stage('Deploy Development') {
            agent any
            steps {
                sh "chmod +x deploy.sh"
                sh "./deploy.sh dev $TAG_NAME"
            }
        }
        
        stage('Decide Deploy to Test'){
            when {
                branch 'master'
            }
            agent none
            steps {
                input message: 'Deploy to Test?'
            }            
        }
        
        stage('Deploy Test'){
            when {
                branch 'master'
            }
            agent any
            steps {
                sh "chmod +x deploy.sh"
                sh "./deploy.sh test $TAG_NAME"
            }
        }
        
        stage("End to End Tests") {
            when {
                branch 'master'
            }
            agent any
            steps {
                sh "chmod +x ui-tests.sh"
                sh "./ui-tests.sh"
            }
        }
        
        stage('Decide Deploy to Prod'){
            when {
                branch 'master'
            }
            agent none
            steps {
                input message: 'Deploy to Prod?'
            }            
        }
        
        stage('Deploy Prod'){
            when {
                branch 'master'
            }
            agent any
            steps {
                sh "chmod +x deploy.sh"
                sh "./deploy.sh prod $TAG_NAME"
            }
        }
        
        stage('Verify Deployment'){
            when {
                branch 'master'
            }
            agent none
            steps {
                script {
                    env.PRODUCTION_OK = input message: 'Does the deployment is correct?', ok: 'Continue', 
                    parameters: [booleanParam(defaultValue: true, name: 'The application is working as expected')]

                    env.PRODUCTION_LATEST="${env.DOCKER_REPOSITORY}:production-latest"
                    env.PRODUCTION_PREVIOUSLY="${env.DOCKER_REPOSITORY}:production-previously"
                }
            } 
        }
        
        
        stage('Rollback'){
            when {
                expression { env.PRODUCTION_OK == 'false'}
            }
            agent any
            steps {
                sh "chmod +x deploy.sh"
                sh "./deploy.sh prod $PRODUCTION_LATEST"
            }
        }
        
        stage('Tag Production Package'){
            when {
                expression { env.PRODUCTION_OK == 'true'}
            }
            agent any
            steps{
                sh 'docker tag $PRODUCTION_LATEST $PRODUCTION_PREVIOUSLY'
                sh 'docker tag $TAG_NAME $PRODUCTION_LATEST'
                sh 'docker push $PRODUCTION_PREVIOUSLY'
                sh 'docker push $PRODUCTION_LATEST'
            }
        }






        

        
    }
}
