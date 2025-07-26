pipeline {
    agent any

    environment {
	DEPLOY_TRIGGER_URL = "http://43.200.47.141:5000/deploy"
    }

    stages {
	stage('Trigger DockerHost Build') {
	    steps {
		script {
		    sh "curl -X POST ${DEPLOY_TRIGGER_URL}"
		}
	    }
	}
    }
}
