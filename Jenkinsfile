// Jenkinsfile (Git 리포지토리 루트에 위치)
pipeline {
    agent any

    environment {
        // --- ⚠️ 이 값들을 실제 환경에 맞게 수정해주세요 ⚠️ ---
        DOCKER_HUB_USERNAME = "pinkpink222"
        DOCKER_IMAGE_NAME = "${DOCKER_HUB_USERNAME}/dvwa"
        DVWA_HOST_IP = "43.200.47.141" // DVWA가 배포될 실제 서버의 IP 주소 (Ansible 대상 노드)
        SSH_CREDENTIALS_ID = "doc_jen_id" // Jenkins Credentials에 등록된 SSH Key ID
        GIT_CREDENTIAL_ID = "pinkpink222" // Jenkins Credentials에 등록된 Git 계정 ID
        DOCKERHUB_CREDENTIAL_ID = 'dockerhub_credentials' // Jenkins Credentials에 등록된 Docker Hub ID
        RDS_ENDPOINT = "fin-rds-4glno1.cnk02q8uu22n.ap-northeast-2.rds.amazonaws.com" // RDS 엔드포인트
        // --- ⚠️ 수정 완료 ⚠️ ---
    }

    stages {
        stage ('Checkout Source Code') {
            steps {
                // 'credentailsId' -> 'credentialsId' 오타 수정
                git branch: 'main', credentialsId: GIT_CREDENTIAL_ID, url: 'https://github.com/pinkpink222/dev_dv_pj'
            }
        }

        stage('Build Docker Image') { // 중복된 stage 블록 제거
            steps {
                script {
                    sh "docker build -t ${DOCKER_IMAGE_NAME}:${BUILD_NUMBER} ./dvwa_project"
                    // 'docer tag' -> 'docker tag' 오타 수정
                    // $DOCKER_IMAGE_NAME}:latest -> ${DOCKER_IMAGE_NAME}:latest 오타 수정
                    sh "docker tag ${DOCKER_IMAGE_NAME}:${BUILD_NUMBER} ${DOCKER_IMAGE_NAME}:latest"
                }
            }
        }

        stage('Push Docker Image to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: DOCKERHUB_CREDENTIAL_ID, passwordVariable: 'DOCKER_HUB_PASSWORD', usernameVariable: 'DOCKER_HUB_USER')]) {
                        sh "echo ${DOCKER_HUB_PASSWORD} | docker login -u ${DOCKER_HUB_USER} --password-stdin"
                        sh "docker push ${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}"
                        sh "docker push ${DOCKER_IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Deploy with Ansible') {
            steps {
                script {
                    // Ansible Playbook을 임시 파일로 생성
                    writeFile file: 'deploy_dvwa.yml', text: """
                    ---
                    - name: Deploy DVWA
                      hosts: dvwa_servers
                      become: yes
                      tasks:
                        - name: Pull latest DVWA Docker image
                          docker_image:
                            name: ${DOCKER_IMAGE_NAME}
                            tag: latest
                            pull: yes
                            source: pull

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
                              - "80:80"
                            env:
                              MYSQL_SERVER: ${RDS_ENDPOINT} # RDS 엔드포인트를 환경 변수로 받아서 사용
                              MYSQL_USER: root
                              MYSQL_PASSWORD: password
                              MYSQL_DATABASE: dvwa
                            networks:
                              - name: dvwa_network
                            state: started
                            restart_policy: always

                        - name: Ensure Docker network exists for DVWA
                          docker_network:
                            name: dvwa_network
                            state: present
                    """

                    // Ansible Inventory 파일을 임시로 생성
                    writeFile file: 'inventory.ini', text: """
                    [dvwa_servers]
                    ${DVWA_HOST_IP} ansible_user=ec2-user ansible_ssh_private_key_file=./ssh_key
                    """
                    // ⭐ 중요: DVWA_HOST_IP (43.200.47.141) 가 Ansible이 접속할 실제 DVWA 서버입니다.
                    // 'localhost'로 설정하면 Jenkins 컨테이너 내부에서 자신에게 배포하려 합니다.
                    // 실제 시나리오는 Jenkins -> DVWA_HOST_IP (원격) 배포입니다.
                    // ansible_user는 DVWA_HOST_IP 서버의 사용자 이름입니다 (Amazon Linux 2023은 ec2-user가 기본).

                    // Jenkins Credentials에서 SSH 키를 가져와 임시 파일로 저장 (침투 테스트 시 탈취 대상)
                    withCredentials([sshUserPrivateKey(credentialsId: SSH_CREDENTIALS_ID, keyFileVariable: 'SSH_KEY_FILE_PATH')]) {
                        sh "cp ${SSH_KEY_FILE_PATH} ./ssh_key"
                        sh "chmod 600 ./ssh_key"
                        // 'sh "cp' 라인이 잘려있어 다음 줄로 넘어가서 불완전한 상태였습니다.
                        // 완전한 명령어가 되도록 수정했습니다.
                        sh "ansible-playbook -i inventory.ini deploy_dvwa.yml --private-key ./ssh_key"
                    }
                }
            }
        }
    }

    post {
        always {
            // 임시 파일 정리
            sh "rm -f ./ssh_key"
            sh "rm -f ./deploy_dvwa.yml"
            sh "rm -f ./inventory.ini"
        }
    }
}
