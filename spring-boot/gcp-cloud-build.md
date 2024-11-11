# GCP Cloud Build로 배포 자동화

Github 레포지토리의 main 브랜치에 push가 발생할 경우 GCP Cloud Build에서 빌드 및 배포까지 실행하는 파이프라인을 만들어보았습니다.



#### Cloud Build&#x20;

1. **GCP Cloud Build > 트리거 > 트리거 만들기**
   1. 이벤트: 브랜치로 푸시
   2. 소스: 1세대
   3. 브랜치: `^main$`
   4. 유형: Cloud Build 구성 파일 (YAML 또는 JSON)
   5. 위치: 저장소
2. **cloudbuild.yaml 배포 과정**
   1. main 브랜치 pull
   2. GCS > {bucket\_name}/credential 폴더에서 credentials.json 파일 복사
   3. `gradle clean build -Pprofile=prod -x test` 빌드 실행
   4. 빌드 파일 실행
      1. GCS > {bucket\_name}/history 폴더에 빌드 파일을 저장하여 버전 관리
   5. 배포 스크립트 (deploy.sh) 실행&#x20;
      1. 그냥 백그라운드로 실행하면 로그 출력 때문에 빌드가 끝나지 않음!
         1. `> $HOME_DIR/nohup.out 2>&1 &` 을 추가하여 로그를 파일에서 관리하고 스크립트는 종료



#### YAML 파일

<pre class="language-yaml" data-title="cloudbuild.yaml" data-overflow="wrap" data-full-width="false"><code class="lang-yaml">steps:
  # git pull
  - name: 'gcr.io/cloud-builders/git'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        git stash
        git remote update
        git checkout main
        git pull origin main

  # credential 파일 저장
  - name: 'gcr.io/cloud-builders/gsutil'
    args:
      - 'cp'
      - 'gs://{bucket_name}/credential/*.json'
      - '/workspace/server/mkt-solution/src/main/resources/'

  # credential 파일에 권한 부여
  - name: 'ubuntu'
<strong>    entrypoint: 'bash'
</strong>    args:
      - '-c'
      - |
        echo "INFO: Listing JSON files in the directory:"
        ls -l /workspace/server/mkt-solution/src/main/resources/
        chmod -R 777 /workspace/server/mkt-solution/src/main/resources/*.json

  # 빌드 실행
  - name: 'gradle:8.8-jdk17'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        cd /workspace/server/mkt-solution
        if gradle clean build -Pprofile=prod -x test; then
          echo "Build succeeded."
        else
          echo "Build failed."
          curl -X POST "{slack_webhook}" -H "Content-Type: application/json" -d "{\"text\": \"$_BUILD_FAIL\"}"
          exit 1
        fi

  # 빌드 파일 GCS에 업로드
  - name: 'gcr.io/cloud-builders/gsutil'
    args:
      - 'cp'
      - '/workspace/server/mkt-solution/build/libs/mkt-solution-0.0.1-SNAPSHOT.jar'
      - 'gs://{bucket_name}/history/$BUILD_ID-mkt-solution-0.0.1-SNAPSHOT.jar'

  # 배포 스크립트 GCS에 업로드
  - name: 'gcr.io/cloud-builders/gsutil'
    args:
      - 'cp'
      - '/workspace/server/mkt-solution/deploy.sh'
      - 'gs://{bucket_name}/'

  # 배포 스크립트 실행 및 완료 슬랙 알림
  - name: 'gcr.io/cloud-builders/gcloud'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        echo "INFO: Deploying application"
        curl -X POST "{slack_webhook}" -H "Content-Type: application/json" -d "{\"text\": \"$_START\"}"
        gcloud compute ssh {instance_id} --zone asia-northeast3-a \
        --command="sudo gsutil cp gs://{bucket_name}/deploy.sh /home/{user}/ &#x26;&#x26; sudo chmod +x /home/{user}/deploy.sh &#x26;&#x26; sudo /home/{user}/deploy.sh $BUILD_ID"
        if [[ $? -eq 0 ]]; then
          curl -X POST "{slack_webhook}" -H "Content-Type: application/json" -d "{\"text\": \"$_SUCCESS\"}"
        else
          curl -X POST "{slack_webhook}" -H "Content-Type: application/json" -d "{\"text\": \"$_FAIL\"}"
        fi

# 슬랙 메시지
substitutions:
  _BUILD_FAIL: "❌ Ads Hub build failed. Please check the system and try again.  - &#x3C;{github_link}|Github>, &#x3C;{codebuild_link}|CodeBuild>"
  _START: "🚀 Ads Hub deployment started. Please monitor. - &#x3C;{github_link}|Github>, &#x3C;{codebuile_link}|CodeBuild>"
  _SUCCESS: "✅ Ads Hub deployment completed. Thank you for your patience. - &#x3C;{github_link}|Github>, &#x3C;{codebuild_link}|CodeBuild>"
  _FAIL: "❌ Ads Hub deployment failed. Please check the system and try again.  - &#x3C;{github_link}|Github>, &#x3C;{codebuild_link}|CodeBuild>"

# Cloud Build 설정
options:
  logging: CLOUD_LOGGING_ONLY
  machineType: 'E2_HIGHCPU_8'
  dynamicSubstitutions: true

</code></pre>

* `name`: 실행할 컨테이너 이미지
* `entrypoint`: 컨테이너가 시작될 때 실행할 명령어
* `args`: entrypoint에 전달할 인자들 지정
  * 여러 명령어는 `-c` 플래그로 지정
  * `|` 파이프를 이용하여 여러 줄의 명령어 실행
* `substitutions`: 빌드 과정에서 사용할 변수 정의
* `dynamicSubstitutions`: true로 지정하면 동적으로 변수 값 설정 가능



#### 배포 스크립트

{% code title="deploy.sh" overflow="wrap" %}
```bash
#!/bin/bash

BUILD_ID=$1
HOME_DIR=/home/{user}

echo "INFO: Current user: $(whoami)"
echo "INFO: Home directory: $HOME_DIR"

export JWT_SECRET='***'

# 실행중인 서버 종료
echo 'INFO: Killing existing process'
pkill -f mkt-solution-0.0.1-SNAPSHOT.jar

# 종료 확인
sleep 5
if pgrep -f mkt-solution-0.0.1-SNAPSHOT.jar; then
    echo 'ERROR: Failed to kill existing process'
fi

# 이전 로그 파일 백업
echo 'INFO: Create new nohup.out file. Check history folder for old nohup.out file'
if [ -f $HOME_DIR/nohup.out ]; then
    if ! { cp $HOME_DIR/nohup.out $HOME_DIR/logs/$(date +'%Y%m%d').log && rm $HOME_DIR/nohup.out && touch $HOME_DIR/nohup.out; }; then
        echo 'ERROR: Failed to create new nohup.out file'
    fi
else
    echo 'INFO: No nohup.out file to archive, creating new one.'
    touch $HOME_DIR/nohup.out
fi

# GCS에서 빌드 파일 다운로드
echo "INFO: Downloading new JAR file from GCS"
if ! gsutil cp gs://{bucket_name}/history/$BUILD_ID-mkt-solution-0.0.1-SNAPSHOT.jar $HOME_DIR/mkt-solution-0.0.1-SNAPSHOT.jar; then
    echo 'ERROR: Failed to download new JAR'
fi

# 빌드 파일 실행
echo 'INFO: Starting new jar file'
nohup java -jar -Dspring.profiles.active=prod -Duser.timezone=Asia/Seoul $HOME_DIR/mkt-solution-0.0.1-SNAPSHOT.jar > $HOME_DIR/nohup.out 2>&1 &
NEW_PID=$!

sleep 5
if ps -p $NEW_PID > /dev/null; then
   echo "INFO: Java process started successfully with PID $NEW_PID"
   exit 0
else
   echo "ERROR: Failed to start Java process"
   exit 1
fi
```
{% endcode %}



#### 리전 선택

[본 문서](https://cloud.google.com/build/docs/locations?hl=ko#restricted\_regions\_for\_some\_projects)에 따라 제한이 적은 `asia-east` 리전을 선택했습니다.

Cloud Build > 트리거에서 수동으로 빌드를 트리거할 수 있고, 대시보드에서 로그 및 상태를 확인할 수 있습니다.![](<../.gitbook/assets/스크린샷 2024-07-10 오후 6.11.19.png>)



#### main 브랜치 protection

Github에서는 main 브랜치에 바로 push하는 것을 막고, develop 브랜치에 먼저 커밋을 모은 뒤 pull request를 통해서만 main에 push할 수 있도록 설정했습니다.&#x20;

Settings > Branches > Add classic branch protection rule 에서 branch rule을 생성할 수 있습니다.

<figure><img src="../.gitbook/assets/스크린샷 2024-11-11 오후 6.45.40.png" alt=""><figcaption></figcaption></figure>



이렇게 yaml 파일 작성만으로 빌드와 배포를 쉽게 자동화할 수 있습니다. 👍
