pipeline {
    agent any

    environment {
        // IP adresa tvojej VM
        DOCKER_HOST_IP = '192.168.50.200'
    }

    stages {
        stage('1. Prepare Environment') {
            steps {
                echo '=== Nastavujem prostredie a premenné ==='
                sh '''
                    # Vytvorenie .env súboru ak neexistuje
                    echo "DOCKER_HOST_IP=${DOCKER_HOST_IP}" > .env
                    set -a
                    . ./.env
                '''
            }
        }

        stage('2. Build Core Images') {
            steps {
                echo '=== Stavia sa Docker obrazy (Open5GS, srsLTE, srsRAN) ==='
                sh '''
                    docker build -t docker_open5gs ./base
                    docker build -t docker_srslte ./srslte
                    docker build -t docker_srsran ./srsran
                '''
            }
        }

        stage('3. Deploy 5G Core') {
            steps {
                echo '=== Spúšťam Open5GS 5G Core ==='
                sh '''
                    docker compose -f sa-deploy.yaml up -d mongodb open5gs-db amf upf smf udr udm ausf nrf nssf pcf
                    sleep 10
                '''
            }
        }

        stage('4. Add Subscriber to DB') {
            steps {
                echo '=== Automatické pridanie SIM karty do MongoDB ==='
                sh '''
                    # Vloženie subscribera priamo cez mongo príkaz (namiesto WebUI)
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
                echo '=== Spúšťam Rádio (gNB) a Mobil (UE) ==='
                sh '''
                    docker compose -f srsgnb_zmq.yaml up -d
                    sleep 5
                    docker compose -f srsue_5g_zmq.yaml up -d
                    sleep 10
                '''
            }
        }

        stage('6. Automated Testing (Ping)') {
            steps {
                echo '=== Testujem prepojenie cez 5G sieť (PING) ==='
                sh '''
                    # Overenie, či UE dostal IP a dokáže spraviť ping
                    docker exec srsue ping -c 5 10.45.0.1
                '''
            }
        }
    }

    post {
        always {
            echo '=== Upratovanie po teste ==='
            // Tu môžeme v budúcnosti pridať docker compose down, ak budeme chcieť čistiť prostredie
        }
        success {
            echo '✅ 5G Pipeline prebehla ÚSPEŠNE! Mobil je pripojený a ping prešiel.'
        }
        failure {
            echo '❌ 5G Pipeline ZLYHALA! Skontroluj logy kontajnerov.'
        }
    }
}
