---
name: solK8sGtw
versionSource: VERSION
sourceDesign: README.md
---

# solK8sGtw productReadme

## A. 產品定位與責任邊界

`solK8sGtw` 提供 Kubernetes / Helm 基本部署指引、TLS 憑證代辦基礎設施，以及 `sysSysGtwDemo` 系統閘道入口驗測範例。本 `productReadme` 是方案層自然語言操作腳本，供自然人與 AI Agent 依步驟部署、觀測、驗測與復原；釋出後即為產品 REPO 根目錄 `README.md`。

責任邊界如下：

* `sysK8sGtw` 負責 ingress-nginx、cert-manager、DuckDNS DNS-01 webhook、ClusterIssuer 與 private CA issuer 基礎能力。
* `sysSysGtwDemo` 負責建立 ACME 與 Private CA 兩個 HTTPS 驗測入口，回應 `sys-gtw-demo-ok` 作為閘道可達性驗測 marker。
* `modSysGtwDemo` 是 demo / stub 驗測入口，不代表正式業務服務能力。
* DNS provider、ACME 服務與 Kubernetes cluster 是外部執行條件，必須先備妥。

## B. 前置條件

對應 `docProgTest#01` 與 `solStory#1`。

1. 確認 Windows PowerShell 7 可使用指定執行檔：`C:\Users\User\AppData\Local\Microsoft\WindowsApps\pwsh.exe`。
2. 確認 Docker Desktop 與 Kubernetes context 可用。
3. 確認 Helm 3、kubectl、Python 可執行。
4. 確認 Lens 或等價 Kubernetes 觀測工具可觀察 namespace、pod、service、ingress、certificate 與 secret。

最小安裝路徑：

1. 安裝 Docker Desktop for Windows，於 Docker Desktop Settings > Kubernetes 勾選 Enable Kubernetes，套用後等待 Kubernetes running。
2. 安裝 Helm 3，並確認 `helm.exe` 位於目前 shell 的 `PATH`。
3. 安裝或確認 `kubectl.exe` 可用；Docker Desktop 啟用 Kubernetes 後通常會提供 docker-desktop context。
4. 安裝 Python 3，並確認 `python --version` 可回傳版本。
5. 若使用 Lens，連線到目前 `kubectl config current-context` 顯示的 cluster；若不使用 Lens，以下 `kubectl get` 與 `helm status` 指令即為等價觀測方式。

驗證指令：

```powershell
C:\Users\User\AppData\Local\Microsoft\WindowsApps\pwsh.exe -NoLogo -NoProfile -Command '$PSVersionTable.PSVersion'
docker --version
helm version --short
kubectl version --client
kubectl config current-context
kubectl cluster-info
python --version
kubectl get namespace
helm list --all-namespaces
```

成功判定：

* Kubernetes context 指向可用 cluster。
* Helm 與 kubectl 可連線。
* 後續 `solk8sgtw` namespace 可被建立或更新。
* Docker Desktop Kubernetes 顯示 running，或 `kubectl cluster-info` 可連線到預期 cluster。

失敗處理：

* 找不到 `helm` 或 `kubectl` 時，先修正 `PATH` 或重新安裝對應工具，再重新開啟 PowerShell 7。
* `kubectl cluster-info` 無法連線時，先確認 Docker Desktop Kubernetes 已啟用、目前 context 是預期 cluster，必要時執行 `kubectl config use-context docker-desktop`。
* Python 不可用時，安裝 Python 3 或修正 Windows App execution alias / PATH 後再執行 code stage。

## C. DNS 與 Secret 設定

對應 `docProgTest#02`、`solStory#2` 與 `solCase#2.1`。

必要設定：

* `DUCKDNS_TOKEN`：DuckDNS DNS-01 驗證用 token。本案允許以 `DUCKDNS_TOKEN` 代表 README 中的 `DNS_PROVIDER_TOKEN / DUCKDNS_TOKEN`。
* `ACME_EMAIL`：ACME 帳號聯絡 email。
* ACME host：`solk8sgtw-syssysgtwdemo.duckdns.org`。
* Private CA host：`solk8sgtw-syssysgtwdemo.127.0.0.1.nip.io`，此 host 不屬於 ACME DNS-01 入口。

在目前 PowerShell 7 session 設定必要環境變數：

```powershell
$env:DUCKDNS_TOKEN = '<your-duckdns-token>'
$env:ACME_EMAIL = '<your-acme-email@example.com>'
```

若已存在 Kubernetes secret，可先檢查內容再決定是否重匯：

```powershell
kubectl -n solk8sgtw get secret solk8sgtw-dns01 private-ca-root-tls-secret --ignore-not-found
kubectl -n solk8sgtw describe secret solk8sgtw-dns01
```

操作：

> 風險提示：本步驟會在目前 Kubernetes context 建立或更新 `solk8sgtw` namespace 內的 DNS-01 與 Private CA secret，並可能將 Private CA root 憑證匯入 Windows CurrentUser Root certificate store。若不確定目前 context 或不允許修改使用者信任根，請先停止。

```powershell
& '3a.buildStage(sysK8sGtw)\deployPkg.tmp\1.scripts\importK8sSecret.ps1' -NonInteractive
```

成功判定：

* `solk8sgtw-dns01` secret 存在於 `solk8sgtw` namespace。
* `private-ca-root-tls-secret` 存在於 `solk8sgtw` namespace。
* `solk8sgtw-syssysgtwdemo.duckdns.org` 解析到本機驗測入口所需位址。
* 若本次使用環境變數匯入，`$env:DUCKDNS_TOKEN` 與 `$env:ACME_EMAIL` 在執行腳本的同一 PowerShell session 中有值。

失敗判定：

* `DUCKDNS_TOKEN` 與既有 K8S secret 皆不存在時，必須中止，不得以假值繼續。
* `ACME_EMAIL` 與既有 K8S secret 皆不存在時，必須中止。

復原方式：

```powershell
kubectl -n solk8sgtw delete secret solk8sgtw-dns01 private-ca-root-tls-secret --ignore-not-found
certmgr.msc
```

開啟 `certmgr.msc` 後，於 Current User 的 Trusted Root Certification Authorities 中移除本方案匯入的 Private CA root 憑證；若不確定憑證指紋，應先保留並交由維運人員確認。

## D. 系統建置與部署

對應 `solCase#1.2`、`solCase#1.3`、`setAct常例Adm部署更新Helm系統`。

建置模組 image：

```powershell
& '2.codeStage(sysK8sGtw)\modK8sGtw\2.codeStage.ps1'
& '2.codeStage(sysSysGtwDemo)\modSysGtwDemo\2.codeStage.ps1'
```

建置系統部署包：

```powershell
& '3a.buildStage(sysK8sGtw)\3a.buildStage.ps1'
& '3a.buildStage(sysSysGtwDemo)\3a.buildStage.ps1'
```

部署順序：

```powershell
& '3a.buildStage(sysK8sGtw)\deployPkg.tmp\1.scripts\install.ps1'
& '3a.buildStage(sysSysGtwDemo)\deployPkg.tmp\1.scripts\install.ps1'
```

成功判定：

* `solk8sgtw-sysk8sgtw` 與 `solk8sgtw-syssysgtwdemo` Helm release 為 deployed。
* `sys-k8s-gtw` 與 `sys-sys-gtw-demo` deployment rollout 成功。
* `sys-sys-gtw-demo-acme` 與 `sys-sys-gtw-demo-private-ca` ingress 存在。
* 不使用 Lens 時，可用 `helm -n solk8sgtw status solk8sgtw-sysk8sgtw`、`helm -n solk8sgtw status solk8sgtw-syssysgtwdemo`、`kubectl -n solk8sgtw get pod,svc,ingress,certificate,secret` 作為等價觀測。

解除安裝：

```powershell
& '3a.buildStage(sysSysGtwDemo)\deployPkg.tmp\1.scripts\uninstall.ps1'
& '3a.buildStage(sysK8sGtw)\deployPkg.tmp\1.scripts\uninstall.ps1'
```

## E. TLS 憑證代辦驗測

對應 `docProgTest#03`、`solStory#3`、`solCase#3.1`、`solCase#3.2`。

ACME 入口應使用：

* host：`solk8sgtw-syssysgtwdemo.duckdns.org`
* issuer：`public-letsencrypt-duckdns-prod-clusterissuer`
* secret：`sys-sys-gtw-demo-acme-tls`

Private CA 入口應使用：

* host：`solk8sgtw-syssysgtwdemo.127.0.0.1.nip.io`
* issuer：`private-ca-clusterissuer`
* secret：`sys-sys-gtw-demo-private-ca-tls`

驗證指令：

```powershell
kubectl -n solk8sgtw get ingress sys-sys-gtw-demo-acme
kubectl -n solk8sgtw get ingress sys-sys-gtw-demo-private-ca
kubectl -n solk8sgtw get certificate
kubectl -n solk8sgtw get secret sys-sys-gtw-demo-acme-tls
kubectl -n solk8sgtw get secret sys-sys-gtw-demo-private-ca-tls
```

安全失敗判定：

* `tlsIssuer` 缺失或錯誤時，對應 certificate 不得被判定 Ready。
* DNS-01 授權資料缺失時，ACME 憑證不得被判定簽發成功。
* 測試腳本或 AI Agent 必須以 K8S condition、secret、order/challenge 或 HTTP/TLS 結果判定，不得只用舊 secret 或快取結果判定成功。

## F. SysGtw 入口驗測

對應 `docProgTest#04`、`solStory#4`、`solCase#4.1`、`solCase#4.2`。

本節兩組 `curl.exe` 與後續 `kubectl` 檢查即為 `runAct自訂Adm驗測SysGtw` 的操作入口；若任一入口未達成功判定，該 runAct 不得視為完成。

驗測 ACME 入口：

```powershell
curl.exe --ipv4 -sS https://solk8sgtw-syssysgtwdemo.duckdns.org
curl.exe --ipv4 -sS -w '\nHTTP_STATUS=%{http_code}\n' https://solk8sgtw-syssysgtwdemo.duckdns.org
```

驗測 Private CA 入口：

```powershell
curl.exe --ipv4 --ssl-no-revoke -sS https://solk8sgtw-syssysgtwdemo.127.0.0.1.nip.io
curl.exe --ipv4 --ssl-no-revoke -sS -w '\nHTTP_STATUS=%{http_code}\n' https://solk8sgtw-syssysgtwdemo.127.0.0.1.nip.io
```

成功判定：

* DNS 解析成功。
* HTTP 狀態為 2xx。
* TLS issuer / Secret 與入口 host 對應正確。
* 回應內容包含 `sys-gtw-demo-ok`。
* 不得用 `curl --resolve` 取代真實 DNS 檢查。

安全失敗判定：

* `sysK8sGtw`、ingress controller、`sysSysGtwDemo` 任一必要連線不可達時，應得到 404、503、連線失敗或 certificate not Ready 等可辨識狀態。
* 測試腳本不得以快取或舊狀態誤判入口驗測成功。

## G. 文件測試與方案測試對照

本 `productReadme` 承接：

* `docProgTest#01`：K8S/Helm 基本環境。
* `docProgTest#02`：DNS 與入口 host 設定。
* `docProgTest#03`：TLS 憑證代辦與安全失敗。
* `docProgTest#04`：SysGtw 入口驗測與安全失敗。

本 `productReadme` 支援：

* `e2eTest#01`：K8S/Helm 基本環境。
* `e2eTest#02`：DNS 與入口 host。
* `e2eTest#03`：TLS 憑證代辦成功路徑。
* `e2eTest#04`：TLS 與 DNS 異常安全失敗。
* `e2eTest#05`：SysGtw 入口。
* `e2eTest#06`：入口連線異常安全失敗。

## H. 發布資訊

部署組態來源為 README `<IV.A 部署組態>`：

* 開發 REPO：`https://github.com/twMoonBear-Laboratory/solK8sGtw1`
* 產品 REPO：`https://github.com/twStellerWhale-Ocean2/solK8sGtw1`
* productReadme 來源：`3b.buildStage(solK8sGtw)/productReadme.md`

產品 REPO 根目錄 `README.md` 由本 `productReadme` 發布；`3b.buildStage(solK8sGtw)/3b.buildStage.ps1` 產出 `3b.buildStage(solK8sGtw)/deployPkg.tmp/productReadme/README.md` 作為發布來源。
