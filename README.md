# tailscale-acl

두리안 tailnet ACL을 코드로 관리하는 저장소예요. `policy.hujson`을 PR로 고치면 자동으로 검사하고, main에 머지되면 Tailscale에 적용돼요.

공개 저장소라서 `policy.hujson`에 **이메일을 넣지 않아요.** 사람은 Tailscale 역할로만 지정해요.

- `autogroup:member`: tailnet 팀원 전체 (Grafana TCP 3000, Tailscale Serve 경유)
- `autogroup:admin`: 관리 콘솔에서 **Admin** 역할을 준 운영 담당 5명 (온프렘 SSH, OpenStack 대시보드, 태그 소유)

누가 운영 담당인지는 콘솔의 Users → 역할로 정해요. 파일에 이메일 모양이 들어가면 워크플로가 실패해요.

Grafana는 팀 Tailnet에 연결한 PC에서 `http://100.107.149.49:3000`으로 접속해요. Monitor 기기에 `tag:durian-monitor`가 붙어 있고 Tailscale Serve TCP 3000이 설정되어 있어야 해요. Grafana 자체는 localhost:3000에서만 수신하며, Prometheus(9090)와 Alertmanager(9093)는 SSM 포트 포워딩으로만 접근해요.

## 흐름

| 언제 | 하는 일 | 자격 증명 |
|---|---|---|
| PR 올림 | 문법 + `tests` 검사 (적용 안 함) | 읽기 전용 (`policy_file:read`) |
| main 머지 | 검사 통과하면 Tailscale에 적용 | 쓰기 (`policy_file`), main에서만 |

고정 비밀번호(OAuth secret, API 키) 없이 GitHub OIDC 토큰으로 로그인해요(Tailscale workload identity federation). 키가 새어 나갈 일이 없어요.

## 처음 한 번 설정 (Tailscale Owner/Admin: 최동경)

0. 관리 콘솔 → **Users**에서 운영 담당(AWS 인프라·OpenStack 인프라·OpenStack 컨테이너·배포 자동화·보안)에게 **Admin** 역할, 나머지는 Member
1. 관리 콘솔 → **Settings → Trust credentials → Credential → OpenID Connect** 로 2개 만들어요.

   | 이름 | Issuer | Subject | Scope |
   |---|---|---|---|
   | `gitops-test` | GitHub | `repo:ktcloud-durian/tailscale-acl:pull_request` | `policy_file:read` |
   | `gitops-apply` | GitHub | `repo:ktcloud-durian/tailscale-acl:ref:refs/heads/main` | `policy_file` |

   만들면 나오는 **Client ID**와 **Audience**(`api.tailscale.com/<client ID>` 모양)를 적어 둬요.
2. GitHub 저장소 → Settings → Secrets and variables → Actions → **New repository secret** 5개
   - `TS_TAILNET` (General settings의 tailnet ID)
   - `TS_TEST_CLIENT_ID`, `TS_TEST_AUDIENCE`
   - `TS_APPLY_CLIENT_ID`, `TS_APPLY_AUDIENCE`
3. 저장소의 `policy.hujson`(v3, 이메일 없음)으로 PR → test 통과 확인 → 머지 → apply 성공 확인. 콘솔의 지금 정책과 달라지는 점: groups 대신 autogroup, SSH 규칙 src가 autogroup:member(실제 접속은 tcp:22 grant로 admin만)
4. 잘 되면 **Policy file management → Prevent edits in the admin console** 켜고 External reference에 이 저장소 주소 입력

## 저장소 규칙

main 규칙: PR 필수, 승인 1명, 강제 푸시 금지, Bypass 없음, 필수 검사 `acl`.

## 급할 때

Owner/Admin은 콘솔에서 **Edit anyway**로 고칠 수 있어요. 다음 apply가 콘솔 수정을 덮어쓰니까, 고친 내용은 바로 PR로도 올려 주세요.
