# outstagram-application

Outstagram 프로젝트의 Spring 애플리케이션 코드와 GitHub Actions를 위한 Workflow 파일입니다.

이 중 Spring 애플리케이션 코드는 GitHub Repository 코드를 사용하였습니다.
- https://github.com/orgs/dev-online-k8s/repositories

## GitHub Actions 
### Workflow
워크플로우를 단계별로 설정하여 GitHub Actions를 동작하게 합니다.

- 트리거 설정 (on)
    - 트리거: 이 워크플로우는 push 이벤트가 발생할 때 실행됩니다.
    - 경로: feed-server/ 디렉토리 내의 파일들이 변경될 때만 실행됩니다.
    - 태그: **f_**로 시작하는 태그가 푸시될 때도 워크플로우가 실행됩니다.
      ```
      name: Feed CI
      
      on:
        push:
          paths:
          - 'feed-server/**'
          tags:
          - 'f_*'
      ```

- 작업: build-and-push
    - 작업: 이 작업은 ubuntu-latest 가상 머신에서 실행됩니다.
    - 권한: ID 토큰에 대한 쓰기 권한과 레포지토리 파일에 대한 읽기 권한이 설정되어 있습니다.
      ```
      jobs:
        build-and-push:
          runs-on: ubuntu-latest
          permissions:
            id-token: write
            contents: read
      ```

- 1단계: 레포지토리 체크아웃
    - 체크아웃: 워크플로우가 레포지토리 코드를 실행할 수 있도록 레포지토리의 파일을 확인하고 다운로드합니다.
      ```
          steps:
          - name: Checkout Repository
            uses: actions/checkout@v4
      ```
 
- 2단계: Docker 설정 / MySQL 컨테이너 빌드 및 실행
    - Docker 설정: Docker Buildx를 설정하여 다중 플랫폼 빌드 및 Docker 이미지를 빌드할 수 있게 합니다.
    - MySQL 컨테이너 빌드 및 실행: feed-server 디렉토리에서 MySQL Docker 이미지를 빌드한 후, MySQL 컨테이너를 백그라운드에서 실행합니다.
      ```
          - name: Set up Docker
            uses: docker/setup-buildx-action@v2
      
          - name: Build and run MySQL container
            working-directory: ./feed-server
            run: |
              docker build -t custom-mysql .
              docker run -d --name test-mysql -p 3306:3306 custom-mysql
              docker ps    
      ```
 
- 3단계: application.yaml 파일 생성
    - application.yaml 파일 생성: feed-server 디렉토리로 이동하여 application.yaml 파일을 생성하고, GitHub Secrets에서 가져온 AWS_FEED_PROPERTIES 값을 해당 파일에 작성합니다.
      ```
          - name: Make application.yaml
            run: |
              cd feed-server
              touch ./src/main/resources/application.yaml
              echo "${{ secrets.AWS_FEED_PROPERTIES }}" > ./src/main/resources/application.yaml
      ```
 
- 4단계: Bitnami Kafka 설정
    - Kafka 설정: Bitnami Kafka를 설정하고 실행하는 액션을 사용하여 Kafka 서비스를 시작합니다.
      ```
          - name: Start Bitnami Kafka
            uses: bbcCorp/kafka-actions@v1  
      ```
 
- 5단계: JDK 21 설정 및 gradlew 실행 권한 부여
    - JDK 21 설정: temurin JDK 21을 설정하여 Java 빌드 환경을 준비합니다.
    - gradlew 실행 권한 부여: gradlew 스크립트에 실행 권한을 부여하여 실행 가능하도록 합니다.
      ```
          - name: Set up JDK 21
            uses: actions/setup-java@v4
            with:
              java-version: '21'
              distribution: 'temurin'
              
          - name: Grant execute permission for gradlew
            run: chmod +x ./feed-server/gradlew
      ```
 
- 6단계: AWS 자격 증명 설정
    - AWS 자격 증명 설정: AWS 자격 증명을 설정하여 AWS 리소스에 접근할 수 있도록 합니다.
      ```
          - name: Configure AWS credentials
            uses: aws-actions/configure-aws-credentials@v4
            with:
              aws-region: ${{ vars.AWS_REGION }}
              role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/${{ vars.AWS_CI_ROLE }}
      ```
 
- 7단계: Amazon ECR 로그인
    - ECR 로그인: Amazon ECR에 로그인하여 Docker 이미지 푸시를 위한 인증을 처리합니다.
      ```
          - name: Login to Amazon ECR
            id: login-ecr
            uses: aws-actions/amazon-ecr-login@v2
      ```
 
- 8단계: 태그 이름 추출 및 태그 확인
    - 태그 이름 추출: GitHub 태그 이름을 추출하여 IMAGE_TAG 환경 변수에 저장합니다.
    - 태그 확인: 추출된 IMAGE_TAG 환경 변수를 출력하여 확인합니다.
      ```
          - name: Extract tag name
            id: extract_tag
            shell: bash
            run: echo "IMAGE_TAG=${GITHUB_REF#refs/tags/}" >> $GITHUB_ENV
      
          - name: Confirm Image Tag
            run: echo $IMAGE_TAG
      ```
 
- 9단계: Docker 이미지 빌드 및 푸시 (Jib 사용)
    - Docker 이미지 빌드 및 푸시: gradlew jib 명령어를 사용하여 feed-server 디렉토리에서 Docker 이미지를 빌드하고, Amazon ECR에 푸시합니다. IMAGE_TAG를 사용하여 태그가 지정된 이미지를 푸시합니다.
      ```
          - name: Build and Push Docker image with Jib
            run: |
              cd feed-server
              ./gradlew jib --image=${{ steps.login-ecr.outputs.registry }}/${{ vars.ECR_FEED_SERVER_REPO }}:${{ env.IMAGE_TAG }}
      ```
