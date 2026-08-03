pipeline {
    agent any

    environment {
        DOCKER_HOST_IP = '192.168.50.200'
    }

    stages {
        stage('1. Prepare Environment') {
            steps {
                echo '=== Environment preparing ==='
                sh '''
                    # Ak existuje šablóna .env.example, skopírujeme ju, inak použijeme existujúci .env
                    if [ -f .env.example ]; then
                        cp .env.example .env
                    fi
                    
                    # Doplň / prepíš DOCKER_HOST_IP
                    echo "DOCKER_HOST_IP=${DOCKER_HOST_IP}" >> .env
                    
                    set -a
                    . ./.env
                '''
            }
        }

        stage('2. Build Core Images') {
            steps {
                echo '=== Preparing docker images ==='
                sh '''
                    set -a
                    . ./.env
                    docker-compose -f sa-deploy.yaml -f srsgnb_zmq.yaml -f srsue_5g_zmq.yaml pull --ignore-pull-failures || true
                    
                    docker-compose -f sa-deploy.yaml -f srsgnb_zmq.yaml -f srsue_5g_zmq.yaml build
                '''
            }
        }

        stage('3. Deploy 5G Core') {
            steps {
                echo '=== Deploy Open5GS 5G Core ==='
                sh '''
                    set -a
                    . ./.env
                    docker-compose -f sa-deploy.yaml down --remove-orphans || true
                    
                    docker-compose -f sa-deploy.yaml up -d --no-build
                    sleep 10
                '''
            }
        }
        stage('4. Add Subscriber to DB') {
            steps {
                echo '=== Add subscriber to MongoDB ==='
                sh '''
                    docker exec mongodb mongosh open5gs --eval '
                    db.subscribers.updateOne(
                      { imsi: "001011234567895" },
                      {
                        $set: {
                          imsi: "001011234567895",
                          subscribed_rau_tau_timer: 12,
                          network_access_mode: 0,
                          subscriber_status: 0,
                          security: {
                            k: "8BAF473F2F8FD09487CCCBD7097C6862",
                            amf: "8000",
                            op: null,
                            opc: "11111111111111111111111111111111",
                            sqn: NumberLong(32),
                            schema: "OPC"
                          },
                          slice: [{
                            sst: 1,
                            sd: "000001",
                            default_indicator: true,
                            session: [{
                              name: "internet",
                              type: 3,
                              pcc_rule: [],
                              ambr: { uplink: { value: 1, unit: 3 }, downlink: { value: 1, unit: 3 } },
                              qos: { index: 9, arp: { priority_level: 8, pre_emption_capability: 1, pre_emption_vulnerability: 1 } }
                            }]
                          }],
                          ambr: { uplink: { value: 1, unit: 3 }, downlink: { value: 1, unit: 3 } }
                        }
                      },
                      { upsert: true }
                    )'
                '''
            }
        }

        stage('5. Deploy RAN & UE') {
            steps {
                echo '=== Deploy gNB and UE ==='
                sh '''
                    set -a
                    . ./.env
                    docker-compose -f srsgnb_zmq.yaml up -d
                    sleep 5
                    docker-compose -f srsue_5g_zmq.yaml up -d
                    sleep 10
                '''
            }
        }

        stage('6. Automated Testing (Ping)') {
            steps {
                echo '=== 5G network (PING) ==='
                sh '''
                    docker exec srsue ping -c 5 10.45.0.1
                '''
            }
        }
    }

    post {
        always {
            echo '=== Cleanup ==='
            sh '''
                set -a
                . ./.env
                docker-compose -f srsue_5g_zmq.yaml down -v || true
                docker-compose -f srsgnb_zmq.yaml down -v || true
                docker-compose -f sa-deploy.yaml down -v || true
            '''
        }
        success {
            echo '✅ 5G Pipeline successful.'
        }
        failure {
            echo '❌ 5G Pipeline ERROR!'
        }
    }
}
