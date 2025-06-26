---
title: ルート ルールを設定せずに Ingress の標準設定を使用できますか？
weight: 40
---

簡単な `Ingress` 設定は、`Host`、`TLS`、および基本的な `Path` の正確な一致を使用するだけで、ルート ルールを設定する必要はありません。
`Path` には、`Ingress` リソースでは `.` 文字を使用しないでください。

例えば、以下の `Ingress` リソースは、`Host` が `example.com` で、`URL` が `/helloworld` のリクエストに一致します。

{{< text bash >}}
$ kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
  annotations:
    kubernetes.io/ingress.class: istio
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /helloworld
        pathType: Prefix
        backend:
          service:
            name: myservice
            port:
              number: 8000
EOF
{{< /text >}}

しかし、以下のルールは機能しません。なぜなら、`Path` に正規表現が使用されており、`ingress.kubernetes.io` 注釈が追加されているためです。

{{< text bash >}}
$ kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: this-will-not-work
  annotations:
    kubernetes.io/ingress.class: istio
    # 除入口クラス以外の他の入口注釈は受け入れられません
    ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /hello(.*?)world/
        pathType: Prefix
        backend:
          service:
            name: myservice
            port:
              number: 8000
EOF
{{< /text >}}
