pipeline {
    agent any

    environment {
	DEPLOY_TRIGGER_URL = "http://43.200.47.141:8090/"
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
