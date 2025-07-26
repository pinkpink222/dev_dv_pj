pipeline {
    agent any

    environment {
        DOCKER_HUB_USERNAME = "pinkpink222"
        DOCKER_IMAGE_NAME = "pinkpink222/dev-dv-pj"
        DVWA_HOST_IP = "43.200.47.141"
        SSH_CREDENTIALS_ID = "doc_jen_id"
        GIT_CREDENTIAL_ID = "pinkpink222"
        DOCKERHUB_CREDENTIAL_ID = "dockerhub_credentials"
        RDS_ENDPOINT = "fin-rds-4glno1.cnk02q8uu22n.ap-northeast-2.rds.amazonaws.com"
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                git branch: 'main', credentialsId: GIT_CREDENTIAL_ID, url: 'https://github.com/pinkpink222/dev_dv_pj'
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKERHUB_CREDENTIAL_ID) {
                        sh "docker build -t ${DOCKER_IMAGE_NAME}:latest ./dvwa_project"
                        sh "docker push ${DOCKER_IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Deploy with Ansible') {
            steps {
                script {
                    // Ansible Playbook
                    writeFile file: 'deploy_dvwa.yml', text: """
                    ---
                    - name: Deploy DVWA
                      hosts: dvwa_servers
                      become: yes
                      tasks:
                        - name: Ensure Docker network exists for DVWA
                          docker_network:
                            name: dvwa_network
                            state: present

                        - name: Pull latest DVWA Docker image
                          docker_image:
                            name: ${DOCKER_IMAGE_NAME}
                            tag: latest
                            source: pull
                            pull: yes

                        - name: Stop and remove old DVWA container
                          docker_container:
                            name: dvwa_container
                            state: absent
                            force_kill: yes
                          ignore_errors: yes

                        - name: Start new DVWA container
                          docker_container:
                            name: dvwa_container
                            image: ${DOCKER_IMAGE_NAME}:latest
                            ports:
                              - "8090:80"
                            env:
                              MYSQL_SERVER: ${RDS_ENDPOINT}
                              MYSQL_USER: root
                              MYSQL_PASSWORD: password
                              MYSQL_DATABASE: dvwa
                            networks:
                              - name: dvwa_network
                            state: started
                            restart_policy: always
                    """

                    // Inventory File
                    writeFile file: 'inventory.ini', text: """
                    [dvwa_servers]
                    ${DVWA_HOST_IP} ansible_user=ec2-user ansible_ssh_private_key_file=./ssh_key
                    """

                    // SSH Key 인증 및 실행
                    withCredentials([sshUserPrivateKey(credentialsId: SSH_CREDENTIALS_ID, keyFileVariable: 'SSH_KEY_FILE_PATH')]) {
                        sh "cp ~/.ssh/ ./ssh_key"
                        sh "chmod 600 ./ssh_key"
                        sh "ansible-playbook -i inventory.ini deploy_dvwa.yml --private-key ./ssh_key"
                    }
                }
            }
        }
    }

    post {
        always {
            sh "rm -f ./ssh_key"
            sh "rm -f ./deploy_dvwa.yml"
            sh "rm -f ./inventory.ini"
        }
    }
}

