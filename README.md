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

- GitHub Actions과 CI/CD 도구
    - CI(Continuous Integration, 지속적 통합): 코드의 변경사항을 정기적으로 **빌드** 및 **테스트**하여 공유 레포지토리에 **통합**하는 것.
        ```yaml
        # 해당 프로젝트의 CI 과정
           steps:
              - name: Checkout repository
              uses: actions/checkout@v2
        
              - name: Install dependencies
              run: npm ci
        
              - name: Build
                run: npm run build
        ```
    - CD(Continuous Delivery/Deployment, 지속적 배포): 지속적인 서비스 제공(Continuous Delivery) 및 지속적인 배포(Continuous Deployment)를 의미. 코드 변경 사항이 production 환경까지 릴리즈 되는 과정.
        ```yaml
        # 해당 프로젝트의 CD 과정
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
    - CI/CD 도구: Jenkins, GitLab CI, CircleCI, GitHub Actions
      - GitHub Actions  
        - 해당 프로젝트에서 사용한 CI/CD 도구
        -  GitHub이 제공하는 CI/CD(지속적 통합/배포) 플랫폼
        - GitHub 코드 변경 시 자동으로 워크플로우 실행
        - GitHub의 코드, 이슈, PR과 직접 상호작용
        
  
- S3와 스토리지
  - Amazon S3(Simple Storage Service): AWS에서 제공하는 클라우드 객체 스토리지 서비스. 데이터를 객체 단위로 저장하기 때문에 파일은 하나의 객체로 취급됨. 각 객체는 데이터와 메타데이터로 구성되며 고유 식별자를 통해 객체를 구분할 수 있다. 객체는 하나 이상의 버킷에 저장되며 HTML, CSS, JavaScript 파일들을 S3 버킷에 업로드하면 따로 서버 없이도 웹사이트로 운영할 수 있음.

    - hanghaeplus3 => 버킷
    - `404.html`, `favicon.ico`와 같은 파일 혹은 폴더 => 객체
    
<img width="1266" alt="스크린샷 2024-11-18 오후 9 26 16" src="https://github.com/user-attachments/assets/4cfc59e8-0fde-471c-8f78-d6812d5cdced">


- CloudFront와 CDN
  - CloudFront: AWS의 CDN 서비스로 전 세계 여러 지역에 위치한 엣지 로케이션을 통해 콘텐츠를 캐싱하고 배포함. S3와 CloudFront를 연동하면 사용자는 위치적으로 가까운 엣지 로케이션으로부터 컨텐츠를 받을 수 있어 로딩 속도가 향상된다.
  - CDN(Content Delivery Network): 전 세계 여러 지역에 분산된 서버 네트워크.

- 캐시 무효화(Cache Invalidation)
  -  CDN의 캐시 무효화란 엣지 서버에 저장된 캐시 콘텐츠를 강제로 삭제하거나 업데이트 하는 프로세스다. 강제 삭제를 통해 최신 콘텐츠가 사용자에게 제공될 수 있다. 웹사이트가 긴급 수정되거나, 새로운 버전을 배포하는 경우에 필요한 기능이다.

- Repository secret과 환경변수
  - 저장소에서 저장 관리하는 암호화된 값. API 키, 데이터베이스 비밀번호와 같은 민감한 정보를 저장한다. 이는 GitHub Actions workflow에서 접근 가능하다.

  ![](https://velog.velcdn.com/images/parkseridev/post/bc634549-9ca5-4f9b-ab96-6dfb219a35ad/image.png)
    ```yaml
        # deployment yml file
        - name: Configure AWS credentials
          uses: aws-actions/configure-aws-credentials@v1
          with:
            aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
            aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
            aws-region: ${{ secrets.AWS_REGION }}
    ```

<br/>

## CDN과 성능최적화

1. 개요
   - 목적
     - 기존 S3 호스팅 환경과 CloudFront CDN 전환 후의 성능 비교 분석한다.
   - 사용 기술 및 환경
     - 프레임워크: Next.js
     - 호스팅: AWS S3 → CloudFront CDN
     - 배포 방식: Static Export
     - 성능 측정: Chrome Devtools Network
   - 측정 방법
        - Next.js 프로젝트를 빌드 후 S3 버킷에 업로드하여 정적 웹사이트를 호스팅 했을 때와 해당 S3 버킷을 Origin으로 하는 CloudFront Distribution을 생성하여 배포했을 때의 성능을 측정하여 비교한다.


2. 성능 측정 결과 


| CDN 도입 전 -> 후                                                                                        |
|------------------------------------------------------------------------------------------------------|
| <img alt="s3" src="https://github.com/user-attachments/assets/1b23dbfc-ff81-4c63-910b-95bd5aea9639"> |


| Metric | S3 | CDN | Improvement |
|--------|-----|-----|-------------|
| Load Time | 1.34s | 263ms | **80.4%**   |
| Total Transfer | 706 kB | 425 kB | 39.8%       |
| DOM Content Loaded | 612ms | 61ms | **90.0%**   |
| Finish Time | 7.11s | 6.93s | 2.5%        |
- 결과 분석
  - S3에서 CloudFront CDN으로 전환 후, 전반적인 성능이 대폭 개선됨. 특히 초기 로딩 성능과 DOM 구성 시간에서 큰 향상을 보였음.
       - 파일 압축: 대부분의 파일이 30-70% 압축
       -  응답 시간: 평균 80% 이상 단축
       -  일관된 성능: CDN이 더 안정적인 응답 시간 제공
       - 리소스 효율성: 동일 콘텐츠를 더 적은 리소스로 전송

3. CDN을 통한 주요 개선 요인 분석 
   - 지리적으로 가까운 엣지 로케이션에서 컨텐츠를 제공하여 응답 시간 감소
   - 미리 압축된 정적 콘텐츠를 제공하므로 서버 부하를 줄임 


4. 성능 개선으로 인한 기대 이점
   - 페이지 로딩 속도 향상으로 이탈률 감소
       - 더 빠른 상호작용으로 전환율 증가
   - 비용 절감
       - 데이터 전송량 40% 감소로 대역폭 비용 절감
       - 서버 리소스 사용 효율화
       - 인프라 비용 최적화
   - 시스템 성능
       - 서버 부하 감소
       - 안정적인 서비스 제공
       - 확장성 개선