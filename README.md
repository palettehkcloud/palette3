# 준비
  ![img_11.png](assets/img_11.png)
#### 1. PALETTE-UI

  * Dockerfile 설정
  ![img_4.png](assets/img_4.png)

  * 도커파일 실행 & 사내도커허브에 이미지생성 
  ![img_5.png](assets/img_5.png)

  * 사내도커 허브 또는 로컬에서 이미지 태깅 & github repository에 push
    - 172.16.0.100 ssh 접속 또는 로컬에서
    ```bash    
    docker login ghcr.io -u kjd4717 -p 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
    docker tag palette3-ui:2023.0718.001 ghcr.io/kjd4717/palette3-ui:2023.0718.001
    docker push ghcr.io/kjd4717/palette3-ui:2023.0718.001
    ```
    ![img_6.png](assets/img_6.png)
    => # GitHub 개인 Profile > Packages 에서 확인
    ![img_7.png](assets/img_7.png)

#### 2. PALETTE-API 
  * Dockerfile 설정
    ![img_8.png](assets/img_8.png)
  * 도커파일 실행 & 사내도커허브에 이미지생성
  * 사내도커 허브에서 이미지 태깅 & github repository에 push
    - 172.16.0.100 ssh 접속 또는 로컬에서    
    ```bash    
    docker login ghcr.io -u kjd4717 -p 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
    docker tag palette3-api:2023.0718.001 ghcr.io/kjd4717/palette3-api:2023.0718.001
    docker push ghcr.io/kjd4717/palette3-api:2023.0718.001
    ```

  
# GKE
  ![img_13.png](assets/img_13.png)

#### 1. 구글콘솔 클러스터 연결
  ```bash
  gcloud container clusters get-credentials palette-cluster-1 --zone asia-northeast3-b --project palette-393108
  ```
  ![img.png](assets/img1.png)

#### 2. github연동용 secret 생성 
  * github.com/palettehkcloud 에서 대한 90일 만료키 생성
    - 깃허브 > Settings > Developer Settings > Personal access tokens > Tokens(classic)  
    ![img_14.png](assets/img_14.png)
  ```bash
  echo "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" > ~/GITHUB_TOKEN.txt 
  cat ~/GITHUB_TOKEN.txt | docker login https://ghcr.io -u palettehkcloud --password-stdin
  kubectl delete secret palette3-git-regcred    
  kubectl create secret docker-registry palette3-git-regcred --docker-server=ghcr.io --docker-username=palettehkcloud --docker-password=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 --docker-email=palettehkcloud@gmail.com  
  ```  

#### 3. FileStore생성
  ![img.png](assets/img.png)
  ![img_2.png](assets/img_2.png)
  ![img_3.png](assets/img_3.png)

  * ~~GKE Filestore CSI 드라이버가 사용 설정되면 사용자는 다음 구성으로 GKE 제공 멀티 공유 StorageClass enterprise-multishare-rwx에 액세스할 수 있음.~~
  ```bash
    -사용설정
    gcloud container clusters update palette-cluster-1 --update-addons=GcpFilestoreCsiDriver=ENABLED --location asia-northeast3-b
  
    -사용중지
    gcloud container clusters update CLUSTER_NAME --update-addons=GcpFilestoreCsiDriver=DISABLED --location asia-northeast3-b
  
    --생성확인
    kubectl describe sc enterprise-multishare-rwx
  
  ```
  * ~~인스턴스 액세스를 위한 StorageClass, PersistentVolume 및 PersistentVolumeClaim 만들기 - palette-filestore-pv.yaml~~
  ```bash
    -생성확인
      kubectl get pv
  ```

#### 4. palette-configmap-api, -auth, -chat, -phone 생성
  * -chat.yaml 인경우 CHAT_ENABLED, CHAT_ROUTER-ENABLED 값을 true로 한다.
  ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: palette-configmap-api
    data:
      SERVER_PORT: "8080"
      SPRING_PROFILES_ACTIVE: production,production-chat,production-phone
      SPRING_APPLICATION_NAME: palette3-api
      PALETTE_ROOT-DIR: /orange/app/palette3
      PALETTE_WEBAPPS-DIR: /orange/app/palette3/webapps
      PALETTE_CHAT_CIPHER: "Y"
      PALETTE_CHAT_KEY: MnR6bzI4bjdocGEw
      JASYPT_KEY: paletteHello
      CHAT_ENABLED: "false"
      CHAT_ROUTER-ENABLED: "false"
  ```
  ```bash
  kubectl apply -y 
  ```


#### 3. palette-ui, palette-api, palette-auth, palette-chat, palette-phone 생성.

#### 4. 인그레스용 gcloud 명령을 이용하여, global IP를 생성한다 생성.
  ```bash
  gcloud compute addresses create palette3-ingress-ip --global
  ```
![img_9.png](assets/img_9.png)


#### 4. 인그레스 생성. 
  * SSL 작업(임시용도)
    ```bash 
    openssl genrsa -out ingress-palette3.key 2048
    openssl req -new -key ingress-palette3.key -out ingress-palette3.csr
    openssl x509 -req -days 9999 -in ingress-palette3.csr -signkey ingress-palette3.key -out ingress-palette3.crt
    ```
    ![img_10.png](assets/img_10.png)
    
    ```bash
    kubectl create secret tls ingress-palette3-serect --key ./ingress-palette3.key --cert ./ingress-palette3.crt
    ```
* SSL 작업(orange.hkpalette.com)
  ```bash
  kubectl create secret tls ingress-orange-hkpalette-com-serect --key ./ssl/orange.hkpalette.com/private.pem --cert ./ssl/orange.hkpalette.com/cert.pem
  ```  
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: ingress-palette3
      annotations:
        kubernetes.io/ingress.global-static-ip-name: "palette3-ingress-ip"
    spec:
      tls:
        - secretName: ingress-orange-hkpalette-com-serect
      rules:
        - http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: palette3-ui
                    port:
                      number: 3003
              - path: /sockjs-node
                pathType: Prefix
                backend:
                  service:
                    name: palette3-ui
                    port:
                      number: 3003
              - path: /.well-known
                pathType: Prefix
                backend:
                  service:
                    name: palette3-ui
                    port:
                      number: 3003
              - path: /api
                pathType: Prefix
                backend:
                  service:
                    name: palette3-api
                    port:
                      number: 8080
              - path: /upload/images
                pathType: Prefix
                backend:
                  service:
                    name: palette3-api
                    port:
                      number: 8080
              - path: /auth-api
                pathType: Prefix
                backend:
                  service:
                    name: palette3-auth
                    port:
                      number: 8080
              - path: /chat-api
                pathType: Prefix
                backend:
                  service:
                    name: palette3-chat
                    port:
                      number: 8080
              - path: /message
                pathType: Prefix
                backend:
                  service:
                    name: palette3-chat
                    port:
                      number: 8080
              - path: /secured/ws-stomp
                pathType: Prefix
                backend:
                  service:
                    name: palette3-chat
                    port:
                      number: 8080
              - path: /reference
                pathType: Prefix
                backend:
                  service:
                    name: palette3-chat
                    port:
                      number: 8080
              - path: /expired_session
                pathType: Prefix
                backend:
                  service:
                    name: palette3-chat
                    port:
                      number: 8080
              - path: /phone-api
                pathType: Prefix
                backend:
                  service:
                    name: palette3-phone
                    port:
                      number: 8080
  
    ```
