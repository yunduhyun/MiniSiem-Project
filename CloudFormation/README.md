# CloudFormation

mini-siem Project에서 쓰는 AWS 인프라를 코드로 만든 템플릿 2개와 사용법을 담은 폴더이다. 리전은 ap-northeast-2(서울) 기준이다.

## 구성

| 파일              | 스택 이름              | 역할                               | 유지 기간                   |
| ----------------- | ---------------------- | ---------------------------------- | --------------------------- |
| `Persistent.yaml` | `mini-siem-persistent` | 네트워크, 로그 버킷 5개, 로깅 설정 | 계속 유지                   |
| `Ephemeral.yaml`  | `mini-siem-ephemeral`  | NAT, 보안 그룹, EC2 2대, ALB, WAF  | 비용 이슈로 사용시에만 유지 |

스택을 둘로 나눈 이유는 비용이다. NAT 게이트웨이와 ALB는 켜 둔 시간만큼 요금이 나가므로, 로그가 쌓이는 버킷과 로깅 설정은 유지용 스택에 두고 나머지는 필요할 때만 올린다.

## 수집하는 로그

| 로그                        | 설정 위치       | 저장 버킷                         |
| --------------------------- | --------------- | --------------------------------- |
| CloudTrail                  | Persistent.yaml | `mini-siem-cloudtrail-<계정ID>`   |
| VPC Flow Logs               | Persistent.yaml | `mini-siem-vpcflow-<계정ID>`      |
| Route 53 Resolver 쿼리 로그 | Persistent.yaml | `mini-siem-resolver-<계정ID>`     |
| ALB 액세스 로그             | Persistent.yaml | `mini-siem-alb-<계정ID>`          |
| WAF 로그                    | Persistent.yaml | `aws-waf-logs-mini-siem-<계정ID>` |

모든 버킷은 퍼블릭 접근을 막고, AES256 암호화를 쓰며, 30일이 지난 로그를 자동으로 지운다. (`LogRetentionDays`로 변경 가능). 스택을 지워도 로그는 남도록 `Retain` 정책을 적용하였다.

## 아키텍처 요약

- VPC 1개, 퍼블릭 서브넷 2개와 프라이빗 서브넷 2개(가용 영역 2a, 2b)
- 웹 서버(nginx) 2대는 프라이빗 서브넷에 두고, NAT를 통해서만 외부와 통신합니다. 접속은 SSH 대신 SSM을 사용한다
- 인터넷에서 오는 요청은 WAF, ALB, 웹 서버 순서로 지나간다
- CloudTrail은 멀티 리전 트레일로, 다른 리전에서 호출하는 API 기록도 로깅한다.

## 배포 방법

AWS 콘솔의 CloudFormation에서 올린다.

1. **유지용 스택**: "스택 생성"에서 `Persistent.yaml`을 "먼저" 올리고, 스택 이름을 `mini-siem-persistent`로 지정한다.
2. 스택이 `CREATE_COMPLETE`가 되면 Exports 메뉴에서 값 8개가 보이는지 확인한다.
3. **임시용 스택**: `Ephemeral.yaml`을 올리고, 스택 이름은 `mini-siem-ephemeral`로 지정한다. 마지막 화면에서 **IAM 리소스 생성 승인**을 반드시 체크한다(EC2 역할을 생성해야 하기 때문).
4. 두 스택 모두 파라미터 `ProjectName`은 같은 값(`mini-siem`)이어야 한다 -> 필수. 임시용 스택은 이 값으로 유지용 스택의 Export를 가져온다.

사전에 템플릿 문법은 `cfn-lint`로 확인하였다. 남는 경고는 `W3010`(가용 영역을 직접 지정함)과 `W2001`(미사용 파라미터)뿐이며 동작에는 영향이 없다

```bash
cfn-lint Persistent.yaml Ephemeral.yaml
```

## 동작 확인

임시용 스택이 올라온 뒤 Outputs의 `AlbDnsName`으로 확인한다.

| 확인 항목          | 방법                                                       | 정상 결과                                        |
| ------------------ | ---------------------------------------------------------- | ------------------------------------------------ |
| 로드 밸런싱        | 브라우저나 `curl`로 ALB 주소에 여러 번 접속                | `mini-siem web1`, `mini-siem web2`가 번갈아 나옴 |
| 대상 상태          | EC2, 대상 그룹, Targets                                    | 2대 모두 healthy                                 |
| WAF 차단           | `curl -i "http://<ALB 주소>/?q=<script>alert(1)</script>"` | 403 Forbidden                                    |
| ALB 로그           | ALB 버킷의 `AWSLogs/<계정ID>/elasticloadbalancing/`        | 5분쯤 뒤 `.gz` 파일 생성                         |
| WAF 로그           | WAF 버킷의 `AWSLogs/` 아래                                 | 요청 후 몇 분 안에 `.log.gz` 파일 생성           |
| CloudTrail         | CloudTrail 버킷                                            | 5~15분 뒤 파일 생성                              |
| VPC Flow, Resolver | 각 버킷                                                    | 트래픽이 생긴 뒤 파일 생성                       |

## 삭제 방법

1. **임시용 스택을 먼저 삭제합니다.**-> 매우 중요! 유지용 스택의 Export를 쓰고 있어서, 순서가 바뀌면 유지용 스택 삭제가 막힌다
2. 유지용 스택은 앞으로 사용을 할 계획이 없을 때 삭제한다.
3. 버킷은 `Retain`이라 스택을 지워도 남는다. 완전히 정리하려면 S3 콘솔에서 비우고 직접 삭제해야한다.

스택 생성이 실패해서 `ROLLBACK_COMPLETE`가 되면, 그 스택과 이미 만들어진 버킷을 먼저 지우고 다시 올려야 한다. 같은 이름의 버킷이 남아 있으면 재시도가 실패!!).

## ⚠️ ⚠️ ⚠️ ⚠️ 비용 주의 !!⚠️ ⚠️ ⚠️ ⚠️

- 임시용 스택은 NAT 게이트웨이와 ALB가 시간당 요금이 나간다. 테스트가 끝나면 바로 삭제해야한다 -> 진짜 비쌈..
- 유지용 스택은 S3 저장 비용 외에 큰 요금이 없다.

## 보안 유의 사항

- 템플릿은 계정 ID를 `${AWS::AccountId}`로 참조하므로 계정 정보가 코드에 들어 있지 않다
- 웹 서버는 SSH 포트를 열지 않고, ALB에서 오는 80번 포트만 허용한다.
