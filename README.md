# Defender for Servers - P1/P2 Coverage Initiative

Azure Policy カスタムイニシアティブ — 各 Virtual Machine / VMSS / Arc Machine に対してリソース単位の **Defender for Servers** の **P1** および **P2** の有効化状態を個別に監査します。

## Deploy to Azure

### イニシアティブ（P1 + P2 一括チェック）— 推奨

P1/P2 両方のポリシー定義 + イニシアティブ + 割り当てを一括デプロイします。

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fhisashin0728%2FCheck-DefenderForServersResourceLevel%2Frefs%2Fheads%2Fmain%2Fazuredeploy-servers-initiative.json)

### P1 チェックポリシーのみ

Defender for Servers P1 の有効化を監査するポリシーを単体でデプロイします。

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fhisashin0728%2FCheck-DefenderForServersResourceLevel%2Frefs%2Fheads%2Fmain%2Fazuredeploy-servers-p1.json)

### P2 チェックポリシーのみ

Defender for Servers P2 の有効化を監査するポリシーを単体でデプロイします。

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fhisashin0728%2FCheck-DefenderForServersResourceLevel%2Frefs%2Fheads%2Fmain%2Fazuredeploy-servers-p2.json)

---

## 概要

### イニシアティブ

| 項目 | 値 |
|---|---|
| イニシアティブ名 | `custom-defender-for-servers-initiative` |
| 表示名 | `[Custom] Defender for Servers - P1/P2 Coverage Initiative` |
| カテゴリ | Security Center |
| バージョン | 1.0.0 |
| 含まれるポリシー | P1 チェック、P2 チェック（各1つ） |

### 個別ポリシー

| ポリシー | 名前 | subPlan チェック値 |
|---|---|---|
| P1 チェック | `custom-defender-for-servers-p1-audit` | `P1` |
| P2 チェック | `custom-defender-for-servers-p2-audit` | `P2` |

| 共通項目 | 値 |
|---|---|
| モード | All |
| Effect | AuditIfNotExists（デフォルト） / Disabled |
| 対象リソース | `Microsoft.Compute/virtualMachines`、`Microsoft.Compute/virtualMachineScaleSets`、`Microsoft.HybridCompute/machines` |
| チェック対象 | `Microsoft.Security/pricings` (name: VirtualMachines) |

---

## 動作

```
VM / VMSS / Arc Machine が存在する
  ├─ [P1 ポリシー] pricingTier == "Standard" かつ subPlan == "P1" か？
  │    ├─ YES → Compliant (P1)
  │    └─ NO  → NonCompliant
  │
  └─ [P2 ポリシー] pricingTier == "Standard" かつ subPlan == "P2" か？
       ├─ YES → Compliant (P2)
       └─ NO  → NonCompliant
```

### 判定結果の見方

| VM の実際の状態 | P1 ポリシー | P2 ポリシー | 意味 |
|---|---|---|---|
| Defender 無効（Free） | NonCompliant | NonCompliant | Defender for Servers 未有効 |
| P1 有効 | **Compliant** | NonCompliant | P1 のみ有効 |
| P2 有効 | NonCompliant | **Compliant** | P2 有効 |

### チェック内容

| プロパティ | エイリアス | チェック内容 |
|---|---|---|
| `properties.pricingTier` | `Microsoft.Security/pricings/pricingTier` | `Standard` であること |
| `properties.subPlan` | `Microsoft.Security/pricings/subPlan` | `P1` または `P2` と一致すること |

### Pricing API レスポンス例

```
GET {vmResourceId}/providers/Microsoft.Security/pricings/VirtualMachines?api-version=2024-01-01
```

```json
{
  "properties": {
    "pricingTier": "Standard",
    "subPlan": "P2",
    "inherited": "True",
    "inheritedFrom": "/subscriptions/{subscriptionId}"
  }
}
```

> **注意:** リソースレベルで直接 **設定** できるのは P1 のみです。P2 はサブスクリプションレベルでのみ設定可能ですが、API の **読み取り** ではサブスクリプションから継承された P2 も返されます（`inherited: "True"`）。
> 本ポリシーは、設定元に関係なくリソースに対する有効なプラン（P1/P2）を検出します。

---

## パラメーター

### イニシアティブパラメーター

| パラメーター | 型 | 許容値 | デフォルト | 説明 |
|---|---|---|---|---|
| `effectP1` | String | `AuditIfNotExists`, `Disabled` | `AuditIfNotExists` | P1 チェックポリシーの Effect |
| `effectP2` | String | `AuditIfNotExists`, `Disabled` | `AuditIfNotExists` | P2 チェックポリシーの Effect |

---

## デプロイ方法

### Azure CLI によるデプロイ

```bash
# 管理グループスコープへのデプロイ（ポリシー定義 + イニシアティブ + 割り当て）
az deployment mg create \
  --management-group-id <MANAGEMENT_GROUP_ID> \
  --location japaneast \
  --template-file azuredeploy-servers-initiative.json \
  --parameters azuredeploy-servers-initiative.parameters.json
```

### Azure PowerShell によるデプロイ

```powershell
New-AzManagementGroupDeployment `
  -ManagementGroupId "<MANAGEMENT_GROUP_ID>" `
  -Location "japaneast" `
  -TemplateFile "azuredeploy-servers-initiative.json" `
  -TemplateParameterFile "azuredeploy-servers-initiative.parameters.json"
```

---

## ファイル構成

| ファイル | 説明 |
|---|---|
| `policy-defender-servers-p1.json` | P1 チェック ポリシー定義（スタンドアロン JSON） |
| `policy-defender-servers-p2.json` | P2 チェック ポリシー定義（スタンドアロン JSON） |
| `initiative-defender-servers.json` | イニシアティブ定義（スタンドアロン JSON） |
| `azuredeploy-servers-initiative.json` | ARM テンプレート（ポリシー定義 + イニシアティブ + 割り当て） |
| `azuredeploy-servers-initiative.parameters.json` | ARM テンプレートのパラメーターファイル |
| `azuredeploy-servers-p1.json` | ARM テンプレート（P1 ポリシー単体デプロイ） |
| `azuredeploy-servers-p2.json` | ARM テンプレート（P2 ポリシー単体デプロイ） |

---

## 利用可能なエイリアス一覧（参考）

`Microsoft.Security/pricings` で利用可能な主要なポリシーエイリアス:

| エイリアス | 説明 |
|---|---|
| `Microsoft.Security/pricings/pricingTier` | Free / Standard |
| `Microsoft.Security/pricings/subPlan` | P1 / P2 |
| `Microsoft.Security/pricings/inherited` | True / False（継承有無） |
| `Microsoft.Security/pricings/inheritedFrom` | 継承元のスコープ ID |
| `Microsoft.Security/pricings/enablementTime` | 有効化日時 |
| `Microsoft.Security/pricings/extensions[*].name` | 拡張機能名 |
| `Microsoft.Security/pricings/extensions[*].isEnabled` | 拡張機能の有効/無効 |

---

## 参考リンク

- [Pricings - Get (REST API)](https://learn.microsoft.com/en-us/rest/api/defenderforcloud/pricings/get?view=rest-defenderforcloud-2024-01-01)
- [Azure Policy の AuditIfNotExists 効果](https://learn.microsoft.com/ja-jp/azure/governance/policy/concepts/effects#auditifnotexists)
- [Azure Policy イニシアティブ定義](https://learn.microsoft.com/ja-jp/azure/governance/policy/concepts/initiative-definition-structure)
