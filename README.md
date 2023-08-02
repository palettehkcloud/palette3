# 준비
  ![img_11.png](assets/img_11.png)

#### - GITHUB 접근용 키 생성
  * github.com/palettehkcloud 에서 대한 90일 만료키 생성
    - 깃허브 > Settings > Developer Settings > Personal access tokens > Tokens(classic)
      ![img_14.png](assets/img_14.png)
    - 2023.08.02일 90일만료로 생성된 키값 : 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

#### - PALETTE-UI
  * Dockerfile 설정
  ![img_4.png](assets/img_4.png)

  * 도커파일 실행 & 사내도커허브에 이미지생성 
  ![img_5.png](assets/img_5.png)

  * 사내도커 허브 또는 로컬에서 이미지 태깅 & github repository에 push
    - 172.16.0.100 ssh 접속 또는 로컬에서
    ```bash    
    docker login ghcr.io -u palettehkcloud -p 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
    docker tag palette3-ui:<<버전>> ghcr.io/palettehkcloud/palette3-ui:<<버전>>
    docker push ghcr.io/palettehkcloud/palette3-ui:<<버전>>
    ```
    ![img_6.png](assets/img_6.png)
    => # GitHub 개인 Profile > Packages 에서 확인
    ![img_7.png](assets/img_7.png)

#### - PALETTE-API 
  * Dockerfile 설정
    ![img_8.png](assets/img_8.png)
  * 도커파일 실행 & 사내도커허브에 이미지생성
  * 사내도커 허브에서 이미지 태깅 & github repository에 push
    - 172.16.0.100 ssh 접속 또는 로컬에서    
    ```bash    
    docker login ghcr.io -u palettehkcloud -p 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
    docker tag palette3-api:2023.0718.001 ghcr.io/palettehkcloud/palette3-api:2023.0718.001
    docker push ghcr.io/kjd4717/palette3-api:2023.0718.001
    ```

#### - PALETTE-API

  
# GKE
  ![img_13.png](assets/img_13.png)

#### - 구글콘솔 클러스터 연결
```bash
gcloud container clusters get-credentials palette-cluster-1 --zone asia-northeast3-b --project palette-393108
```
  ![img.png](assets/img1.png)

#### - github연동용 secret 생성

```bash
echo "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" > ~/GITHUB_TOKEN.txt 
cat ~/GITHUB_TOKEN.txt | docker login https://ghcr.io -u palettehkcloud --password-stdin
kubectl delete secret palette3-git-regcred    
kubectl create secret docker-registry palette3-git-regcred --docker-server=ghcr.io --docker-username=palettehkcloud --docker-password=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 --docker-email=palettehkcloud@gmail.com  
```

#### - FileStore생성
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

#### - ConfigMap생성 (palette-configmap-api, -auth, -chat, -phone)
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
  - 구글 CLOUD SHELL에서 
```bash
kubectl apply -y https://raw.githubusercontent.com/palettehkcloud/palette3/main/gke/yaml/palette-configmap-api.yaml
kubectl apply -y https://raw.githubusercontent.com/palettehkcloud/palette3/main/gke/yaml/palette-configmap-auth.yaml
kubectl apply -y https://raw.githubusercontent.com/palettehkcloud/palette3/main/gke/yaml/palette-configmap-chat.yaml
kubectl apply -y https://raw.githubusercontent.com/palettehkcloud/palette3/main/gke/yaml/palette-configmap-phone.yaml
```

#### - pod 생성 (palette-ui, palette-api, palette-auth, palette-chat, palette-phone, ttalk-biz, ttalk-chat)
  - 구글 CLOUD SHELL에서
```bash
kubectl apply -y https://raw.githubusercontent.com/palettehkcloud/palette3/main/gke/yaml/palette-api.yaml
kubectl apply -y https://raw.githubusercontent.com/palettehkcloud/palette3/main/gke/yaml/palette-auth.yaml
kubectl apply -y https://raw.githubusercontent.com/palettehkcloud/palette3/main/gke/yaml/palette-chat.yaml
kubectl apply -y https://raw.githubusercontent.com/palettehkcloud/palette3/main/gke/yaml/palette-phone.yaml
kubectl apply -y https://raw.githubusercontent.com/palettehkcloud/palette3/main/gke/yaml/ttalk-biz.yaml
kubectl apply -y https://raw.githubusercontent.com/palettehkcloud/palette3/main/gke/yaml/ttalk-chat.yaml
```

#### - 인그레스에서 사용할 Global IP생성 
```bash
gcloud compute addresses create palette3-ingress-ip --global
```
![img_9.png](assets/img_9.png)


#### 4. 인그레스 생성.
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
    networking.gke.io/v1beta1.FrontendConfig: "http-to-https"
spec:
  tls:
    - secretName: ingress-orange-hkpalette-com-serect
  rules:
    - host: orange.hkpalette.com
      http:
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
          - path: /message_bot
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
    - host: chat.hkpalette.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: 
              service:
                name: ttalk-chat
                port:
                  number: 51000
          - path: /apis
            pathType: Prefix
            backend: 
              service:
                name: ttalk-chat
                port:
                  number: 51000
          - path: /user
            pathType: Prefix
            backend: 
              service:
                name: ttalk-chat
                port:
                  number: 51000
    - host: biz.hkpalette.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: ttalk-biz
                port:
                  number: 51001
          - path: /apis
            pathType: Prefix
            backend:
              service:
                name: ttalk-biz
                port:
                  number: 51001                  
---
apiVersion: networking.gke.io/v1beta1
kind: FrontendConfig
metadata:
  name: http-to-https
spec:
  redirectToHttps:
    enabled: true
    responseCodeName: PERMANENT_REDIRECT
```
```bash
kubectl apply -y https://raw.githubusercontent.com/palettehkcloud/palette3/main/gke/yaml/ingress-palette3.yaml
```

#### - GCP VM 인스턴스 접근방법
- Xshell (https://psawesome.tistory.com/62)
```bat
ssh-keygen -t rsa -b 4096 -f palettehkcloud -C palettehkcloud@gmail.com
```
결과값 palettehkcloud.pub의 내용을
구글클라우드 > ComputerEngine >설정>메타데이터>SSH키(탭) 에 등록한다.
( https://console.cloud.google.com/compute/metadata?project=palette-demo-391808&tab=sshkeys )

- FileZilla (https://copycoding.tistory.com/408)
```bat
C:\puttygen.exe
```
  
