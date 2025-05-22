# Telegraf Azure Monitor 연동 가이드

## 시작 전 준비사항

- **가장 권장되는 방법:** 시스템 관리 ID가 할당된 VM 또는 VMSS에서 실행
- **Telegraf(Entra ID) 인증 옵션:**
  1. Managed ID
  2. Service Principal
  3. X.509 인증서
  4. Resource Owner Password

자세한 내용은 [Telegraf Azure Monitor 인증 공식 문서](https://github.com/influxdata/telegraf/tree/release-1.15/plugins/outputs/azure_monitor#authentication)를 참고하세요.

---

## 설치 방법

아래 스크립트로 Telegraf를 설치할 수 있습니다.

[Telegraf 설치 스크립트](https://raw.githubusercontent.com/vinil-v/gpu-ib-monitoring/refs/heads/main/scripts/gpu-ib-mon_setup.sh)

---

## 설정 방법

설치가 완료된 후(`gpu-ib-mon_setup.sh` 실행 완료 시), 아래 파일을 수정하세요.

- **설정 파일:** `/etc/telegraf/telegraf.conf`

### [[outputs.azure_monitor]] 섹션 값 수정

- `namespace_prefix` (선택)
- `region` (필수)
- `resource_id` (필수)

---

### Azure Monitor 출력 설정 예시

````toml
# Send aggregate metrics to Azure Monitor
[[outputs.azure_monitor]]
  ## Timeout for HTTP writes.
  # timeout = "20s"

  ## Set the namespace prefix, defaults to "Telegraf/<input-name>".
  # namespace_prefix = "Telegraf/"

  ## Azure Monitor doesn't have a string value type, so convert string
  ## fields to dimensions (a.k.a. tags) if enabled. Azure Monitor allows
  ## a maximum of 10 dimensions so Telegraf will only send the first 10
  ## alphanumeric dimensions.
  # strings_as_dimensions = false

  ## Both region and resource_id must be set or be available via the
  ## Instance Metadata service on Azure Virtual Machines.
  #
  ## Azure Region to publish metrics against.
  ##   ex: region = "southcentralus"
  # region = ""
  #
  ## The Azure Resource ID against which metric will be logged, e.g.
  ##   ex: resource_id = "/subscriptions/<subscription_id>/resourceGroups/<resource_group>/providers/Microsoft.Compute/virtualMachines/<vm_name>"
  # resource_id = ""
````

## Azure 인증 방법 설정

### Managed Identity 사용

Azure VM에서 Telegraf를 실행 중이고 Managed Identity를 사용하는 경우, Telegraf는 자동으로 인증을 처리합니다.

```toml
[[outputs.azure_monitor]]
  region = "eastus"
  resource_id = "/subscriptions/<subscription_id>/resourceGroups/<resource_group>/providers/Microsoft.Compute/virtualMachines/<vm_name>"
```

### Service Principal 사용

Telegraf가 Azure VM 외부에서 실행되거나 특정 리소스를 대상으로 메트릭을 전송해야 하는 경우, Service Principal을 사용하여 인증할 수 있습니다.

```toml
[[outputs.azure_monitor]]
  region = "eastus"
  resource_id = "/subscriptions/<subscription_id>/resourceGroups/<resource_group>/providers/Microsoft.Compute/virtualMachines/<vm_name>"
  azure_tenant_id = "<tenant_id>"
  azure_client_id = "<client_id>"
  azure_client_secret = "<client_secret>"
```

> Service Principal을 사용하는 경우, 해당 애플리케이션에 **Monitoring Metrics Publisher** 역할을 할당해야 합니다.

---
