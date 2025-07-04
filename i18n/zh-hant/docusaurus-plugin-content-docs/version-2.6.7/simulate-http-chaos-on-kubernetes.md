---
title: Simulate HTTP Faults
---

本文說明如何透過在 Chaos Mesh 中建立 HTTPChaos 實驗來模擬 HTTP 故障。

## HTTPChaos 介紹

HTTPChaos 是 Chaos Mesh 提供的故障類型。透過建立 HTTPChaos 實驗，您可以模擬 HTTP 請求和響應處理過程中的故障場景。目前 HTTPChaos 支援模擬以下故障類型：

- `abort`：中斷連線

- `delay`：在請求或響應中注入延遲

- `replace`：替換 HTTP 請求或響應訊息中的部分內容

- `patch`：在 HTTP 請求或響應訊息中添加額外內容

HTTPChaos 支援組合不同故障類型。若在建立 HTTPChaos 實驗時同時配置多種 HTTP 故障類型，實驗運行時注入故障的順序為 `abort` -> `delay` -> `replace` -> `patch`。當 `abort` 故障導致短路時，連線將直接中斷。

有關 HTTPChaos 配置的詳細說明，請參閱下方的[欄位說明](#field-description)。

## 注意事項

在注入 HTTPChaos 支援的故障前，請注意以下事項：

- 目標 Pod 上沒有運行的 Chaos Mesh 控制管理器。

- 規則預設會影響 Pod 中的客戶端和伺服器端，若需僅影響單側，請參閱[指定作用側](#specify-side)章節。

- 應停用 HTTPS 訪問，因為目前不支援注入 HTTPS 連線。

- 為使 HTTPChaos 注入生效，客戶端應避免重複使用 TCP 通訊端。這是因為 HTTPChaos 不會影響故障注入前透過 TCP 通訊端發送的 HTTP 請求。

- 在生產環境中謹慎使用非冪等請求（如大多數 POST 請求）。若使用此類請求，故障注入後目標服務可能無法透過重複請求恢復正常狀態。

## 使用 Chaos Dashboard 建立實驗

1. 開啟 Chaos Dashboard，點擊頁面上的 **NEW EXPERIMENT** 建立新實驗：

   ![create an experiment](./img/create-new-exp.png)

2. 在 **Choose a Target** 區域選擇 **HTTP FAULT** 並選取具體行為（如 `RESPONSE ABORT`），然後填寫具體配置。

   ![create HTTP fault](./img/create-new-httpchaos.png)

3. 提交實驗。

   以上範例中，您已配置將「Response abort」故障注入 Port 80 的所有請求。

## 使用 YAML 檔案建立實驗

Chaos Mesh 亦支援使用 YAML 配置檔案建立 HTTPChaos 實驗。在 YAML 檔案中，您可以模擬單一 HTTP 故障類型或組合不同 HTTP 故障類型。

### `abort` 範例

1. 將實驗配置寫入 `http-abort-failure.yaml` 檔案，範例如下：

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: HTTPChaos
   metadata:
     name: test-http-chaos
   spec:
     mode: all
     selector:
       labelSelectors:
         app: nginx
     target: Request
     port: 80
     method: GET
     path: /api
     abort: true
     duration: 5m
   ```

   根據此配置範例，Chaos Mesh 將在指定 Pod 注入 `abort` 故障持續 5 分鐘。故障注入期間，目標 Pod 在 `/api` 路徑透過 port 80 發送的 GET 請求將被中斷。

2. 準備好配置檔案後，使用 `kubectl` 建立實驗：

   ```bash
   kubectl apply -f ./http-abort-failure.yaml
   ```

### 故障組合範例

1. 將實驗配置寫入 `http-failure.yaml` 檔案，如下方範例所示：

   ```yaml
   apiVersion: chaos-mesh.org/v1alpha1
   kind: HTTPChaos
   metadata:
     name: test-http-chaos
   spec:
     mode: all
     selector:
       labelSelectors:
         app: nginx
     target: Request
     port: 80
     method: GET
     path: /api/*
     delay: 10s
     replace:
       path: /api/v2/
       method: DELETE
     patch:
       headers:
         - ['Token', '<one token>']
         - ['Token', '<another token>']
       body:
         type: JSON
         value: '{"foo": "bar"}'
     duration: 5m
   ```

   根據此配置範例，Chaos Mesh 將依序注入 `delay` 故障、`replace` 故障以及 `patch` 故障。

2. 準備好配置檔案後，使用 `kubectl` 建立實驗：

   ```bash
   kubectl apply -f ./http-failure.yaml
   ```

## 欄位說明

### 通用欄位說明

當故障注入的 `target` 設定為 `Request` 或 `Response` 時，通用欄位具有意義。

| Parameter | Type | Description | Default value | Required | Example |
| --- | --- | --- | --- | --- | --- |
| `mode` | string | Specifies the mode of the experiment. The mode options include `one` (selecting a random pod), `all` (selecting all eligible pods), `fixed` (selecting a specified number of eligible pods), `fixed-percent` (selecting a specified percentage of Pods from the eligible pods), and `random-max-percent` (selecting the maximum percentage of Pods from the eligible pods). |  | yes | `one` |
| `value` | string | Provides parameters for the `mode` configuration depending on the value of `mode`. |  | no | 1 |
| `target` | string | Specifies whether the target of fault injuection is `Request` or `Response`. The [`target`-related fields](#description-for-target-related-fields) should be configured at the same time. |  | yes | Request |
| `port` | int32 | The TCP port that the target service listens on. |  | yes | 80 |
| `path` | string | The URI path of the target request. Supports [Matching wildcards](https://www.wikiwand.com/en/Matching_wildcards). | Takes effect on all paths by default. | no | /api/\* |
| `method` | string | The HTTP method of the target request method. | Takes effect for all methods by default. | no | GET |
| `request_headers` | map[string]string | Matches request headers to the target service. | Takes effect for all requests by default. | no | Content-Type: application/json |
| `abort` | bool | Indicates whether to inject the fault that interrupts the connection. | false | no | true |
| `delay` | string | Specifies the time for a latency fault. | 0 | no | 10s |
| `replace.headers` | map[string]string | Specifies the key pair used to replace the request headers or response headers. |  | no | Content-Type: application/xml |
| `replace.body` | []byte | Specifies request body or response body to replace the fault (Base64 encoded). |  | no | eyJmb28iOiAiYmFyIn0K |
| `patch.headers` | [][]string | Specifies the attached key pair of the request headers or response headers with patch faults. |  | no | - [Set-Cookie, one cookie] |
| `patch.body.type` | string | Specifies the type of patch faults of the request body or response body. Currently, it only supports [`JSON`](https://tools.ietf.org/html/rfc7396). |  | no | JSON |
| `patch.body.value` | string | Specifies the fault of the request body or response body with patch faults. |  | no | `{"foo": "bar"}` |
| `duration` | string | Specifies the duration of a specific experiment. |  | yes | 30s |
| `scheduler` | string | Specifies the scheduling rules for the time of a specific experiment. |  | no | 5 \* \* \* \* |
| `tls.secretName` | string | SecretName represents the name of required secret resource. The secrete must combined with data `{"tls.certName":cert, "tls.KeyName":key, "tls.caName":ca}` |  | no | "http-tls-scr" |
| `tls.secretNamespace` | string | SecretNamespace represents the namespace of required secret resource,should be the same with deployment/chaos-controller-manager in most cases |  | no | "chaos-mesh" |
| `tls.certName` | string | CertName represents the data name of cert file in secret, `tls.crt` for example |  | no | "tls.crt" |
| `tls.KeyName` | string | KeyName represents the data name of key file in secret, `tls.key` for example |  | no | "tls.key" |
| `tls.caName` | string | CAName represents the data name of ca file in secret, `ca.crt` for example |  | no | "ca.crt" |

:::note

- 使用 YAML 檔案建立實驗時，`replace.body` 必須是替換內容的 base64 編碼形式。

- 使用 Kubernetes API 建立實驗時，無需對替換內容進行編碼，只需將其轉換為 `[]byte` 並放入 `httpchaos.Spec.Replace.Body` 欄位。範例如下：

```golang
httpchaos.Spec.Replace.Body = []byte(`{"foo": "bar"}`)
```

:::

### `target` 相關欄位說明

#### `Request` 相關欄位

The `Request` field is a meaningful when the `target` set to `Request` during the fault injection.

| Parameter | Type | Description | Default value | Required | Example |
| --- | --- | --- | --- | --- | --- |
| `replace.path` | string | Specifies the URI path used to replace content. |  | no | /api/v2/ |
| `replace.method` | string | Specifies the replaced content of the HTTP request method. |  | no | DELETE |
| `replace.queries` | map[string]string | Specifies the replaced key pair of the URI query. |  | no | foo: bar |
| `patch.queries` | [][]string | Specifies the attached key pair of the URI query with patch faults. |  | no | - [foo, bar] |

#### `Respond`-related fields

The `Response` is a meaningful when the `target` set to `Response` during the fault injection.

| Parameter | Type | Description | Default value | Required | Example |
| --- | --- | --- | --- | --- | --- |
| `code` | int32 | Specifies the status code responded by `target`. | Takes effect for all status codes by default. | no | 200 |
| `response_headers` | map[string]string | Matches request headers to `target`. | Takes effect for all responses by default. | no | Content-Type: application/json |
| `replace.code` | int32 | Specifies the replaced content of the response status code. |  | no | 404 |

## 指定作用端

預設情況下，規則將同時影響 Pod 中的客戶端和伺服器端，但可透過選擇請求標頭來限定僅影響單一端點。

本節提供指定作用端的範例，您可根據實際情況調整規則中的標頭選擇器。

### 客戶端

若需僅對 Pod 中的客戶端注入故障而不影響伺服器端，可透過請求中的 `Host` 標頭篩選請求/回應。

例如，若需中斷所有發往 `http://example.com/` 的請求，可應用以下 YAML 配置：

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: HTTPChaos
metadata:
  name: test-http-client
spec:
  mode: all
  selector:
    labelSelectors:
      app: some-http-client
  target: Request
  port: 80
  path: '*'
  request_headers:
    Host: 'example.com'
  abort: true
```

### 伺服器端

若需僅對 Pod 中的伺服器端注入故障而不影響客戶端，同樣可透過請求中的 `Host` 標頭篩選請求/回應。

例如，若需中斷所有發往服務 `nginx.nginx.svc` 後端伺服器的請求，可應用以下 YAML 配置：

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: HTTPChaos
metadata:
  name: test-http-server
spec:
  mode: all
  selector:
    labelSelectors:
      app: nginx
  target: Request
  port: 80
  path: '*'
  request_headers:
    Host: 'nginx.nginx.svc'
  abort: true
```

其他場景下（特別是注入來自外部的傳入請求時），可透過請求中的 `X-Forwarded-Host` 標頭篩選請求/回應。

例如，若需中斷所有發往公共閘道 `nginx.host.org` 後端伺服器的請求，可應用以下 YAML 配置：

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: HTTPChaos
metadata:
  name: test-http-server
spec:
  mode: all
  selector:
    labelSelectors:
      app: nginx
  target: Request
  port: 80
  path: '*'
  request_headers:
    X-Forwarded-Host: 'nginx.host.org'
  abort: true
```

## TLS

若需在基於 TLS 的連線中注入故障，使用者應啟用 TLS 模式。我們的代理在此扮演中介角色，因此使用者需同時具備兩種身份：作為遠端伺服器時需提供受信任的 CA，作為客戶端時需信任伺服器的 CA。

因此在下方的密鑰資料中，使用者需要自行建立其 TLS 金鑰、CA 與 CRT。

```
{
  "tls.certName":cert,
  "tls.KeyName":key,
  "tls.caName":ca
}
```

若需要建立新的 TLS 伺服器並將連線注入其中，應執行下列步驟：

1. 建立自己的根 CA 私鑰和根 CA 憑證：

   ```
   openssl req -newkey rsa:4096  -x509  -sha512  -days 365 -nodes -out ca.crt -keyout ca.key
   ```

2. 建立伺服器的憑證簽署要求 (CSR)：

   ```
   openssl genrsa -out server.key 2048
   openssl req -new -key server.key -out server.csr
   ```

3. 為伺服器撰寫擴展檔案 `server.ext`，內容如下：

   ```
   authorityKeyIdentifier=keyid,issuer
   basicConstraints=CA:FALSE
   keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
   subjectAltName = @alt_names

   [alt_names]
   IP.1 = X.X.X.X
   ```

4. 產生伺服器的憑證：

   ```
   openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -sha256 -extfile server.ext
   ```

5. 將 CA `ca.crt` 加入用戶端。

6. 將 `server.key`、`server.crt` 和 `ca.crt` 放入密鑰 (secret) 中，並提供給 TLS 模式使用。

若需注入用戶端，應使 HTTP Chaos 的代理行為如同遠端伺服器，只需將上述 `server.ext` 編輯為指定網域。

範例：

```
subjectAltName = @alt_names

[alt_names]
DNS.1 = *.domain.com
IP.1 = xxx.xxx.xxx.xxx
```

## 本地除錯

若不確定特定故障注入的效果，可使用 [rs-tproxy](https://github.com/chaos-mesh/rs-tproxy) 在本機測試相應功能。Chaos Mesh 也透過 chaos-tproxy 提供 HTTP Chaos。