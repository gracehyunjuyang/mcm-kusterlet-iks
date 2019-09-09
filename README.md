# IKS (IBM Cloud Kubernetes) 클러스터 연결하기

참고 링크: https://www.ibm.com/support/knowledgecenter/SSBS6K_3.2.0/mcm/installing/mcm_iks.html

1. IBM Cloud 에 로그인 후 Kubernetes 클릭 
2. `Clusters` 클릭 후 타겟 대상으로 등록할 클러스터를 선택
3. `개요(Overview)` 페이지에서 `접근(Access)` 탭 클릭
4. `kubeconfig.zip` 파일 다운로드 클릭하여 <import_config_directory>     에 위치시킵니다. <import_config_directory> 는 IKS 클러스터 관련 파일을 모아두는 위치로,폴더명은 임의로 명명하시면 됩니다. 저의 경우 `/install/iks` 로 설정했습니다.
5. 터미널에서 <import_config_directory> 로 이동합니다. 
6. 다음의 명령을 수행하여 yaml 파일을 추출합니다.  `kube-config-<zone>-<cluster_name>.yaml` 파일은  `ca-<zone>-<cluster_name>.pem` 파일과 연관이 있습니다. 이 모든 파일은 현재 디렉토리에 그대로 저장되어 있어야 합니다. 
    ```
    unzip -j kubeconfig.zip
    ```
7. ```kube-config-<zone>-<cluster_name>.yaml``` 파일 내용을 확인합니다. 이 때 `clusters`, `contexts`, `users`별로 각각 하나의 값을 가지고 있어야 합니다. 예를 들면 아래와 같습니다.
    ```
    apiVersion: v1
    clusters:
    - cluster:
    certificate-authority: ca-<zone>-<cluster_name>.pem
    server: <target_cluster_kuberentes_api_seerver>
    name: <cluster_name>
    contexts:
    - context:
    cluster: <cluster_name>
    namespace: default
    user: <username>
    name:  <cluster_name>
    current-context: <cluster_name>
    kind: Config
    preferences: {}
    users:
    - name: <username>
    user:
    auth-provider:
        config:
        client-id: kube
        client-secret: kube
        id-token: <id-token>
        idp-issuer-url: https://iam.bluemix.net/identity
        refresh-token: <refresh-token>
        name: oidc
    ```
* [예시 파일 참고](./kube-config-mel01-ikscluster.yml)

8. KUBECONFIG 환경 변수 설정  
    ```
    export KUBECONFIG=kube-config-<zone>-<cluster_name>.yaml
    ```
9. 여기서 kubectl CLI로 연결된 IKS 로 접속 되는지 확인하십시오. 설정된 토큰으로 접속 가능한지 여부를 확인하기 위함입니다. 에러 발생시, 파일을 삭제하고 위의 과정을 다시 하셔야 합니다. 
    ```
    kubetl get nodes
    ````
    노드 버전 정보에 IKS 가 나오는지 확인 


## IBM Multicloud Manager 의 `cluster-import.yaml` 설정 파일 생성하기
1. 터미널에서 `<import_config_directory>` 디렉토리로 이동합니다.
2. cloudctl로 hub-cluster에 로그인합니다.
    ```
    cloudctl login -a https://<Hub Cluster Master Host>:<Cluster Master API Port> --skip-ssl-validation-->
    ```
    ```
    cloudctl login -a https://hubcluster.icp:8443 --skip-ssl-validation
    ```
3. 아래의 명령을 실행하여 configuration template, cluster-import.yaml 파일이 생성합니다. <cluster-name>은 hub cluster에 등록할 클러스터 이름이며, <cluster_namespace>는 클러스터 리소스를 등록할 hub cluster 의 네임스페이스입니다:
    ```
    cloudctl mc cluster template <cluster_name> -n <cluster_namespace> > cluster-import.yaml
    ```
    ```
    cloudctl mc cluster template ikscluster -n mcm-iks > cluster-import.yaml
    ```  
    `cluster-import.yaml` 파일이 자동으로 생성됩니다.  
4. `cluster-import.yaml` 파일을 열어 파라미터를 적절하게 수정합니다. 
* default_admin_user: The cluster administrator username for the target managed cluster
* container_runtime:The container runtime used in the cluster, currently supported options are docker and containerd
* 멀티 클러스터 엔드포인트는 [config.yaml 파일 수정하기](https://www.ibm.com/support/knowledgecenter/SSBS6K_3.2.0/installing/config_yaml.html#mcm-endpoint) 링크를 참고하시기 바랍니다. 
* [수정 파일 예시](./cluster-import.yaml) 


## 클러스터 Import 하기 
이제 클러스터를 연결할 모든 준비가 되었습니다. 
1. 터미널에서 <import_config_directory>로 이동합니다. 
2. *hub-cluster* 에 로그인합니다. 
    ```
    cloudctl login -a https://<Cluster Master Host>:<Cluster Master API Port> --skip-ssl-validation
    ```
3. 관리 대상 클러스터를 연결하기 위해 아래의 명령어를 실행합니다. 
    ```
    cloudctl mc cluster import -f ./cluster-import.yaml -K ./kube-config-mel01-ikscluster.yml | tee cluster-import.log
    ```
4. 클러스터가 성공적으로 Import 되었는지 확인 하십시오.
* IBM Multicloud Manager hub cluster에 로그인
* 내비게이션 바에서 `Clusters` 클릭 
* 새롭게 import 된 관리 대상 클러스터를 리스트에서 확인 
* 상태(status) 가 **Ready** 로 표시됨을 확인 Ensure the the status is Ready. 
* 환경에 따라 status 가 표기되는 데에 시간이 수 분 걸릴 수 있습니다.