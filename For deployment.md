
## 배포 순서
1. RHPDS에서 'OCP4 Getting Started Workshop (2025)' 카탈로그 선택하여 배포하기
2. 아래 명령어 수행하기 (모두 실행하셔야 합니다)

### (1) 실습용 앱에 필요한 권한 추가
필수로 수행해야 하는 '권한' 모듈(parksmap-permissions.adoc)을 실습 리스트에서 제거했기 때문에, 유저 대신 실습 구성자가 필요한 명령어를 대신 수행해야 합니다. (parksmap 애플리케이션이 route의 label을 검색해서 API 엔드포인트를 찾으려면 아래 작업이 필요함)
```
for ns in $(oc get projects -o name | grep '^project.project.openshift.io/wksp-' | cut -d/ -f2); do
  oc policy add-role-to-user view -z default -n "$ns"
done
```

### (2) 리소스 쿼타 설정 (리소스 관리 모듈 관련)
```
cat <<'EOF' > workshop-quota-limits.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: workshop-quota
spec:
  hard:
    pods: "12"
EOF
for ns in $(oc get projects -o name | grep '^project.project.openshift.io/wksp-' | cut -d/ -f2); do
  oc apply -f workshop-quota-limits.yaml -n "$ns"
done
```

### (3) 일반 유저들이 노드의 정보를 볼 수 있도록 권한 주기 (노드 스케줄링 모듈 관련)
* N을 유저 숫자만큼으로 수정해주세요.
```
# view-nodes 권한 생성
oc create clusterrole view-nodes --verb=get,list,watch --resource=nodes
# user1~userN에게 권한 주기 (N 수정 필요)
for i in $(seq 1 N); do
  oc adm policy add-cluster-role-to-user view-nodes "user$i"
done
```

### (4) 노드 테인트 설정 (Taint/Toleration 모듈 관련)
* machineset에서 워커노드의 machine replicas 수를 1개 더 늘린 후에, 워커 노드 중 1대에 다음과 명령어를 실행하세요. ('노드이름' 부분을 실제 노드 이름으로 교체하세요)
```
oc get nodes
oc adm taint nodes 노드이름 workshop-demo=scheduling:NoSchedule
```

### (5) 랩가이드 교체하기
```
for ns in $(oc get projects -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | grep -E '^showroom'); do
  echo "Updating deployment/showroom in project: $ns"
oc -n "$ns" patch deployment showroom --type='strategic' -p='
spec:
  template:
    spec:
      initContainers:
      - name: git-cloner
        env:
        - name: GIT_REPO_URL
          value: 'https://github.com/rh-bj/ocp4-getting-started-showroom'
        - name: GIT_REPO_REF
          value: 'showroom-4-21-console'
      containers:
      - name: content
        env:
          - name: GIT_REPO_URL
            value: 'https://github.com/rh-bj/ocp4-getting-started-showroom'
          - name: GIT_REPO_REF
            value: 'showroom-4-21-console'
'
done
```
