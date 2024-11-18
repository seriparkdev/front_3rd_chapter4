# 프론트엔드 배포 파이프라인

## 개요

GitHub Actions에 워크플로우를 작성해 다음과 같이 배포가 진행되도록 합니다.

![pipeline drawio](https://github.com/user-attachments/assets/4887c713-9fc3-4475-82d7-102022fa6777)


1. 저장소를 체크아웃합니다.
2. Node.js 18.x 버전을 설정합니다.
3. 프로젝트 의존성을 설치합니다.
4. Next.js 프로젝트를 빌드합니다.
5. AWS 자격 증명을 구성합니다.
6. 빌드된 파일을 S3 버킷에 동기화합니다.
7. CloudFront 캐시를 무효화합니다.

<br/>

## 주요 링크

- S3 버킷 웹사이트 엔드포인트: http://hanghaeplus3.s3-website-ap-southeast-2.amazonaws.com/
- CloudFrount 배포 도메인 이름: https://dxlinx0dzob0o.cloudfront.net/

<br/>

## 주요 개념

### GitHub Actions과 CI/CD 도구
   
   1. CI(Continuous Integration, 지속적 통합): 코드의 변경사항을 정기적으로 **빌드** 및 **테스트**하여 공유 레포지토리에 **통합**하는 것.

```yaml
   steps:
      - name: Checkout repository
      uses: actions/checkout@v2

      - name: Install dependencies
      run: npm ci

      - name: Build
        run: npm run build
```
    
    
2. CD(Continuous Delivery/Deployment, 지속적 배포): 지속적인 서비스 제공(Continuous Delivery) 및 지속적인 배포(Continuous Deployment)를 의미. 코드 변경 사항이 production 환경까지 릴리즈 되는 과정.
```yaml
    	- name: Configure AWS credentials
          uses: aws-actions/configure-aws-credentials@v1
          with:
            aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
            aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
            aws-region: ${{ secrets.AWS_REGION }}

      	- name: Deploy to S3
          run: |
            aws s3 sync out/ s3://${{ secrets.S3_BUCKET_NAME }} --delete

      	- name: Invalidate CloudFront cache
          run: |
            aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
```

3. CI/CD 도구
  - Jenkins
  - GitLab CI
  - CircleCI
  - Github Actions

<br/>
  
### S3와 스토리지
Amazon S3(Simple Storage Service)는 AWS에서 제공하는 클라우드 객체 스토리지 서비스이다. 데이터를 객체 단위로 저장하기 때문에 파일은 하나의 객체로 취급된다.

각 객체는 데이터와 메타데이터로 구성되며 고유 식별자를 통해 객체를 구분할 수 있다. 이런 객체들은 하나 이상의 버킷에 저장된다. HTML, CSS, JavaScript 파일들을 S3 버킷에 업로드하면 따로 서버 없이도 웹사이트로 운영할 수 있다.

- hanghaeplus3 => 버킷
- `404.html`, `favicon.ico`와 같은 파일 혹은 폴더 => 객체
<img width="1266" alt="스크린샷 2024-11-18 오후 9 26 16" src="https://github.com/user-attachments/assets/4cfc59e8-0fde-471c-8f78-d6812d5cdced">


### CloudFront와 CDN
CloudFront는 AWS의 CDN 서비스로 전 세계 여러 지역에 위치한 엣지 로케이션을 통해 콘텐츠를 캐싱하고 배포함. S3와 CloudFront를 연동하면 사용자는 위치적으로 가까운 엣지 로케이션으로부터 컨텐츠를 받을 수 있어 로딩 속도가 향상된다.


### 캐시 무효화(Cache Invalidation)
CDN은 캐시 무효화 기능을 제공하는데 캐시 무효화란 엣지 서버에 저장된 캐시 콘텐츠를 강제로 삭제하거나 업데이트 하는 프로세스다. 강제 삭제를 통해 최신 콘텐츠가 사용자에게 제공될 수 있다. 웹사이트가 긴급 수정되거나, 새로운 버전을 배포하는 경우에 필요한 기능이다.

### Repository secret과 환경변수
Repository secret은 GitHub와 같은 저장소에서 저장 관리하는 암호화된 값이다. API 키, 데이터베이스 비밀번호와 같은 민감한 정보를 저장한다. 이는 GitHub Actions workflow에서 접근 가능하다.
![](https://velog.velcdn.com/images/parkseridev/post/bc634549-9ca5-4f9b-ab96-6dfb219a35ad/image.png)


