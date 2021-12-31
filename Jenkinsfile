pipeline
{
    agent any
    environment
    {
        VERSION="${env.BUILD_ID}"
    }
    stages
    {
        stage("Sonar Quality Check")
        {
            agent
            {
                docker
                {
                    image 'openjdk:11'
                }
            }
            steps
            {
               script
               { 
                   withSonarQubeEnv(credentialsId: 'sonar-pwd') 
                   {
                     sh 'chmod +x gradlew'
                     sh './gradlew sonarqube'
                   }
                   
                   timeout(time: 1, unit: 'HOURS') {
                      def qg = waitForQualityGate()
                      if (qg.status != 'OK') {
                           error "Pipeline aborted due to quality gate failure: ${qg.status}"
                      }
                    }
               }
            }
        
        }
        stage("Docker build and Push")
        {
            steps
            {
                script
                {
                    withCredentials([string(credentialsId: 'nexus-pass', variable: 'nexuspwd')])
                    {
                        sh '''
                            docker build -t 34.125.101.133:8083/springapp:${VERSION} .
                            docker login -u admin -p $nexuspwd 34.125.101.133:8083
                            docker push 34.125.101.133:8083/springapp:${VERSION}
                            docker rmi 34.125.101.133:8083/springapp:${VERSION}
                        '''
                    }                   
                }
                
            }
        }
        stage("Datree: Helm Manifest validation")
        {
            steps
            {
                script
                {
                    dir('kubernetes/')
                     {
                        withEnv(['DATREE_TOKEN=cyLUhPV4NEEpv2MMdcEdgD']) {
                              sh 'helm datree test myapp/'
                        }
                     }
                 }
            }
        }
        stage("Helm push")
        {
            steps
            {
                script
                {
                    withCredentials([string(credentialsId: 'nexus-pass', variable: 'nexuspwd')])
                    {
                        dir('kubernetes/')
                        {
                        sh '''
                            helmversion=$( helm show chart myapp | grep version | cut -d: -f 2 | tr -d ' ')
                            tar -czvf  myapp-${helmversion}.tgz myapp/
                            curl -u admin:$nexuspwd http://34.125.101.133:8081/repository/helm-hosted/ --upload-file myapp-${helmversion}.tgz -v
                        '''
                        }
                        
                    }
                }
            }
        }
    }    
}
