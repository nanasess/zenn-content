---
title: "Cloudflare のリソースを Terraform にインポートするための API 活用"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "Cloudflare", "API", "Terraform" ]
published: false
---

*この記事は、まだ書きかけです*

Cloudflare API ドキュメント
https://developers.cloudflare.com/api/

## DNS record

https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs/resources/record
https://developers.cloudflare.com/api/operations/dns-records-for-a-zone-list-dns-records

``` shell
ZONE_ID=ゾーンID
API_TOKEN=APIトークン
curl -X GET "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records" \
     -H "Authorization: Bearer ${API_TOKEN}" \
     -H "Content-Type:application/json"
```

以下のような結果が返ってくるので、 `<zone_id>/<id>` でインポートする
``` json
{
  "errors": [],
  "messages": [],
  "result": [
    {
      "content": "198.51.100.4",
      "name": "example.com",
      "proxied": false,
      "type": "A",
      "comment": "Domain verification record",
      "created_on": "2014-01-01T05:20:00.12345Z",
      "id": "023e105f4ecef8ad9ca31a8372d0c353",
      "locked": false,
      "meta": {
        "auto_added": true,
        "source": "primary"
      },
      "modified_on": "2014-01-01T05:20:00.12345Z",
      "proxiable": true,
      "tags": [
        "owner:dns-team"
      ],
      "ttl": 3600,
      "zone_id": "023e105f4ecef8ad9ca31a8372d0c353",
      "zone_name": "example.com"
    }
  ],
  "success": true,
  "result_info": {
    "count": 1,
    "page": 1,
    "per_page": 20,
    "total_count": 2000
  }
}
```

### import

https://registry.terraform.io/providers/cloudflare/cloudflare/latest/docs/resources/record#import

``` text
import {
    id = "<ZONE_ID>/<id>"
    to = cloudflare_record.example
}
```

