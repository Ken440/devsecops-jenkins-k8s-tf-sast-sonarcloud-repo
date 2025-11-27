pipeline {
    agent any
    tools { 
        maven 'Maven_3_8_4'  
    }

    stages {

        stage('CompileandRunSonarAnalysis') {
            steps {	
                sh 'mvn clean verify sonar:sonar -Dsonar.projectKey=ha23 -Dsonar.organization=ha23 -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=44659a0d9fda26c54ed97a253386720850b20453'
            }
        }

        stage('RunSCAAnalysisUsingSnyk') {
            steps {		
                withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
                    sh 'mvn snyk:test -fn'
                }
            }
        }

        stage('Build') { 
            steps { 
                withDockerRegistry([credentialsId: "dockerlogin", url: ""]) {
                    script {
                        app = docker.build("hck")
                    }
                }
            }
        }

        stage('Push') {
            steps {
                script {
                    docker.withRegistry('https://627073649810.dkr.ecr.us-west-2.amazonaws.com', 'ecr:us-west-2:aws-credentials') {
                        app.push("latest")
                    }
                }
            }
        }

        stage('Kubernetes Deployment of ASG Bugg Web Application') {
            steps {
                withKubeConfig([credentialsId: 'kubelogin']) {
                    sh('kubectl delete all --all -n devsecops')
                    sh('kubectl apply -f deployment.yaml --namespace=devsecops')
                }
            }
        }
stage ('wait_for_testing'){
	   steps {
		   sh 'pwd; sleep 180; echo "Application Has been deployed on K8S"'
	   	}
	   }
	   
	stage('RunDASTUsingZAP') {
    steps {
        withKubeConfig([credentialsId: 'kubelogin']) {

            script {
                // Get LoadBalancer hostname
                def targetHost = sh(
                    script: 'kubectl get svc/asgbuggy -n devsecops -o json | jq -r ".status.loadBalancer.ingress[0].hostname"',
                    returnStdout: true
                ).trim()

                echo "ZAP target URL = http://${targetHost}"

                // Run OWASP ZAP using Docker (recommended)
                sh """
                    docker run --rm \
                    -v ${WORKSPACE}:/zap/wrk \
                    owasp/zap2docker-stable zap.sh \
                    -cmd -quickurl http://${targetHost} \
                    -quickprogress \
                    -quickout zap_report.html
                """
            }

            archiveArtifacts artifacts: 'zap_report.html'
        }
    }
}

    }  // END stages

}  // END pipeline
