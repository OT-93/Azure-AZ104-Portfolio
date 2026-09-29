# Microsoft Entra Domain Services — Hands-on Lab

## Overview

🇺🇸 This lab demonstrates how to deploy and validate **Microsoft Entra Domain Services** in Microsoft Azure, configure the required virtual networking, deploy a Windows Server virtual machine, establish VNet peering, and join the VM to the managed domain.

🇯🇵 このラボでは、Microsoft Azure 上で **Microsoft Entra Domain Services** を構築し、必要な仮想ネットワークを設定します。さらに Windows Server 仮想マシンをデプロイし、VNet ピアリングを構成して、VM をマネージド ドメインに参加させます。

---

## Objectives

🇺🇸

* Deploy Microsoft Entra Domain Services
* Configure the required virtual network and subnets
* Configure DNS for Domain Services
* Deploy a Windows Server 2022 virtual machine
* Configure VNet peering between separate VNets
* Join the Windows Server VM to the managed domain
* Validate DNS resolution and connectivity

🇯🇵

* Microsoft Entra Domain Services をデプロイする
* 必要な仮想ネットワークとサブネットを構成する
* Domain Services 用の DNS を構成する
* Windows Server 2022 仮想マシンをデプロイする
* 別々の VNet 間で VNet ピアリングを構成する
* Windows Server VM をマネージド ドメインに参加させる
* DNS 名前解決とネットワーク接続を検証する

---

## Environment

| Resource                           | Configuration                                 |
| ---------------------------------- | --------------------------------------------- |
| Subscription                       | Azure Free Trial                              |
| Domain                             | `giuseppettn.onmicrosoft.com`                 |
| Domain Services region             | Japan West                                    |
| Domain Services VNet               | `vnet-az104-domainservices`                   |
| Domain Services VNet address space | `10.0.0.0/16`                                 |
| Domain Services subnet             | `DomainServices` — `10.0.0.0/24`              |
| Additional subnet                  | `VMSubnet` — `10.0.1.0/24`                    |
| Domain controller 1                | `10.0.0.4`                                    |
| Domain controller 2                | `10.0.0.5`                                    |
| VM                                 | `vm-az104-ds`                                 |
| VM region                          | Japan East                                    |
| VM VNet                            | `vm-az104-ds-vnet`                            |
| VM VNet address space              | `10.1.0.0/16`                                 |
| VM subnet                          | `default` — `10.1.1.0/24`                     |
| VM private IP                      | `10.1.1.4`                                    |
| VM OS                              | Windows Server 2022 Datacenter: Azure Edition |
| VM size                            | `Standard_B2ats_v2`                           |
| Public IP                          | None                                          |

### Subnet Naming Note

🇺🇸 `VMSubnet` (`10.0.1.0/24`) exists inside the Domain Services VNet, but the Windows Server VM used in this lab is deployed in a separate VNet (`vm-az104-ds-vnet`) and therefore uses its `default` subnet (`10.1.1.0/24`).

🇯🇵 `VMSubnet` (`10.0.1.0/24`) は Domain Services VNet 内に存在しますが、このラボで使用する Windows Server VM は別の VNet (`vm-az104-ds-vnet`) にデプロイされているため、`default` サブネット (`10.1.1.0/24`) を使用しています。

---

## Architecture

```mermaid
%%{init: {"themeVariables": {"fontFamily": "Arial, Helvetica, sans-serif", "fontSize": "15px"}, "flowchart": {"htmlLabels": false, "nodeSpacing": 60, "rankSpacing": 70, "padding": 24}}}%%
flowchart TB

    EntraID["Microsoft Entra ID"]
    AADDS["Microsoft Entra<br/>Domain Services&nbsp;&nbsp;"]

    EntraID -->|"Synchronization"| AADDS

    subgraph DS_VNET["vnet-az104-domainservices<br/>10.0.0.0/16"]
        direction TB

        subgraph DS_SUBNET["DomainServices&nbsp;&nbsp;<br/>10.0.0.0/24"]
            direction LR
            DC1["Domain Controller<br/>10.0.0.4"]
            DC2["Domain Controller<br/>10.0.0.5"]
        end

        VM_SUBNET["VMSubnet<br/>10.0.1.0/24"]
    end

    AADDS --- DS_VNET

    subgraph VM_VNET["vm-az104-ds-vnet<br/>10.1.0.0/16"]
        subgraph DEFAULT_SUBNET["default<br/>10.1.1.0/24"]
            VM["vm-az104-ds<br/>10.1.1.4<br/>Windows Server 2022<br/>Datacenter: Azure Edition"]
        end
    end

    DS_VNET <-->|"VNet Peering"| VM_VNET

    %% --- Stile ---
    classDef identity fill:#0078D4,stroke:#005A9E,stroke-width:2px,color:#FFFFFF
    classDef dc fill:#DEECF9,stroke:#0078D4,stroke-width:2px,color:#1B1B1B
    classDef vm fill:#DFF6DD,stroke:#107C10,stroke-width:2px,color:#1B1B1B
    classDef emptySubnet fill:#F3F2F1,stroke:#8A8886,stroke-width:2px,stroke-dasharray:5 5,color:#1B1B1B

    class EntraID,AADDS identity
    class DC1,DC2 dc
    class VM vm
    class VM_SUBNET emptySubnet

    style DS_VNET fill:transparent,stroke:#0078D4,stroke-width:2px
    style VM_VNET fill:transparent,stroke:#0078D4,stroke-width:2px
    style DS_SUBNET fill:transparent,stroke:#8A8886,stroke-width:2px,stroke-dasharray:5 5
    style DEFAULT_SUBNET fill:transparent,stroke:#8A8886,stroke-width:2px,stroke-dasharray:5 5
```

---

## Key Points

🇺🇸

* Microsoft Entra ID synchronizes identities to Microsoft Entra Domain Services.
* Domain Services provides managed domain functionality, including domain authentication and DNS.
* The Domain Services VNet contains the managed domain controllers.
* The Windows Server VM is deployed in a separate VNet.
* VNet peering provides private network connectivity between the two VNets.
* The VM has no public IP address.
* DNS resolution and TCP connectivity to the Domain Services domain controllers were successfully validated.

🇯🇵

* Microsoft Entra ID の ID 情報が Microsoft Entra Domain Services に同期されます。
* Domain Services は、ドメイン認証や DNS などのマネージド ドメイン機能を提供します。
* Domain Services VNet 内にマネージド ドメイン コントローラーが配置されます。
* Windows Server VM は別の VNet にデプロイされています。
* VNet ピアリングによって、2 つの VNet 間のプライベート ネットワーク接続を確立しています。
* VM にはパブリック IP アドレスを設定していません。
* Domain Services のドメイン コントローラーに対する DNS 名前解決と TCP 接続を正常に検証しました。

---

# Implementation

## 1. Microsoft Entra Domain Services

🇺🇸 Microsoft Entra Domain Services was deployed using the managed domain:

`giuseppettn.onmicrosoft.com`

The service was deployed in the **Japan West** region and configured with the dedicated `DomainServices` subnet.

🇯🇵 Microsoft Entra Domain Services を以下のマネージド ドメインとしてデプロイしました。

`giuseppettn.onmicrosoft.com`

サービスは **Japan West** リージョンにデプロイし、専用の `DomainServices` サブネットを使用するように構成しました。

---

## 2. Virtual Network and DNS

🇺🇸 The Domain Services VNet uses the following address space:

`10.0.0.0/16`

The `DomainServices` subnet is:

`10.0.0.0/24`

The VNet also contains the `VMSubnet` subnet:

`10.0.1.0/24`

The managed domain controllers use:

* `10.0.0.4`
* `10.0.0.5`

🇯🇵 Domain Services VNet には以下のアドレス空間を設定しています。

`10.0.0.0/16`

`DomainServices` サブネット：

`10.0.0.0/24`

また、VNet 内には以下の `VMSubnet` も存在します。

`10.0.1.0/24`

マネージド ドメイン コントローラーの IP アドレス：

* `10.0.0.4`
* `10.0.0.5`

---

## 3. Windows Server Virtual Machine

🇺🇸 A Windows Server 2022 Datacenter: Azure Edition VM was deployed in Japan East.

The VM uses:

* Name: `vm-az104-ds`
* Size: `Standard_B2ats_v2`
* VNet: `vm-az104-ds-vnet`
* VNet address space: `10.1.0.0/16`
* Subnet: `default`
* Subnet address space: `10.1.1.0/24`
* Private IP: `10.1.1.4`
* Public IP: None

🇯🇵 Windows Server 2022 Datacenter: Azure Edition VM を Japan East にデプロイしました。

VM の構成：

* 名前：`vm-az104-ds`
* サイズ：`Standard_B2ats_v2`
* VNet：`vm-az104-ds-vnet`
* VNet アドレス空間：`10.1.0.0/16`
* サブネット：`default`
* サブネット アドレス空間：`10.1.1.0/24`
* プライベート IP：`10.1.1.4`
* パブリック IP：なし

---

## 4. VNet Peering

🇺🇸 VNet peering was configured between:

`vnet-az104-domainservices`

and

`vm-az104-ds-vnet`

The peering status was successfully confirmed from both VNets.

🇯🇵 以下の 2 つの VNet 間で VNet ピアリングを構成しました。

`vnet-az104-domainservices`

と

`vm-az104-ds-vnet`

両方の VNet からピアリングの状態が正常であることを確認しました。

---

# Validation

## Domain Join

🇺🇸 The Windows Server VM successfully joined the managed domain.

The domain membership was validated from the VM with:

`PartOfDomain : True`

🇯🇵 Windows Server VM がマネージド ドメインへの参加に成功したことを確認しました。

VM 上で以下の結果を確認しました。

`PartOfDomain : True`

---

## DNS Resolution

🇺🇸 DNS resolution for the managed domain was validated using `nslookup`.

`giuseppettn.onmicrosoft.com` resolved to the Domain Services domain controllers:

* `10.0.0.5`
* `10.0.0.4`

🇯🇵 `nslookup` を使用してマネージド ドメインの DNS 名前解決を確認しました。

`giuseppettn.onmicrosoft.com` が以下の Domain Services ドメイン コントローラーへ名前解決されることを確認しました。

* `10.0.0.5`
* `10.0.0.4`

---

## DNS Connectivity

🇺🇸 TCP connectivity to DNS port 53 was tested from the VM:

```powershell
Test-NetConnection 10.0.0.4 -Port 53
Test-NetConnection 10.0.0.5 -Port 53
```

Both tests returned:

`TcpTestSucceeded : True`

🇯🇵 VM から DNS の TCP 53 番ポートへの接続を確認しました。

```powershell
Test-NetConnection 10.0.0.4 -Port 53
Test-NetConnection 10.0.0.5 -Port 53
```

両方のテストで以下の結果を確認しました。

`TcpTestSucceeded : True`

---

# Screenshots

## 1. Resource Group

🇺🇸 Overview of the resource group and the resources deployed for the lab.

🇯🇵 ラボ用のリソース グループと、デプロイされたリソースの概要です。

![Resource Group overview](screenshots/01_resource-group-overview.png)

![Domain Services resources](screenshots/02_rg-domainservices-resources-1.png)

![Domain Services resources](screenshots/03_rg-domainservices-resources-2.png)

---

## 2. Microsoft Entra Domain Services

🇺🇸 Overview of the Microsoft Entra Domain Services deployment, health status, properties, and replica sets.

🇯🇵 Microsoft Entra Domain Services のデプロイ、正常性、プロパティ、およびレプリカ セットの概要です。

![Domain Services health](screenshots/04_entra-ds-health.png)

![Domain Services properties](screenshots/05_entra-ds-properties-1.png)

![Domain Services properties](screenshots/06_entra-ds-properties-2.png)

![Domain Services replica sets](screenshots/07_entra-ds-replica-sets.png)

---

## 3. Virtual Networks

🇺🇸 Overview of the virtual networks and subnet configuration used in the lab.

🇯🇵 ラボで使用した仮想ネットワークとサブネット構成の概要です。

![Domain Services VNet](screenshots/08_ds-vnet-overview.png)

![Domain Services VNet subnets](screenshots/09_ds-vnet-subnets.png)

![VM VNet](screenshots/10_vm-vnet-overview.png)

![VM VNet subnets](screenshots/11_vm-vnet-subnets.png)

---

## 4. VNet Peering

🇺🇸 Overview of the VNet peering configuration and peering status between the two VNets.

🇯🇵 2 つの VNet 間で構成した VNet ピアリングと、その接続状態の概要です。

![Domain Services VNet peering](screenshots/12_ds-vnet-peering-summary.png)

![Domain Services VNet peering details](screenshots/13_ds-vnet-peering-detail.png)

![VM VNet peering](screenshots/14_vm-vnet-peering-summary.png)

![VM VNet peering details](screenshots/15_vm-vnet-peering-detail.png)

---

## 5. Virtual Machine

🇺🇸 Overview of the Windows Server VM, its networking configuration, network interface, NSG rules, image and disk configuration, and domain join extension.

🇯🇵 Windows Server VM の概要、ネットワーク構成、ネットワーク インターフェース、NSG ルール、イメージとディスク構成、およびドメイン参加拡張機能の概要です。

![Virtual machine overview](screenshots/16_vm-overview.png)

![VM networking](screenshots/17_vm-networking-summary.png)

![VM network interface](screenshots/18_vm-network-settings-nic.png)

![Network security group rules](screenshots/19_vm-network-settings-nsg-rules.png)

![VM image and disk details](screenshots/20_vm-image-disk-details.png)

![Domain join extension](screenshots/21_vm-domainjoin-extension.png)

---

## 6. Validation

🇺🇸 Validation results confirming domain membership, DNS resolution, and DNS network connectivity between the Windows Server VM and the Domain Services domain controllers.

🇯🇵 Windows Server VM のドメイン参加、DNS 名前解決、および Domain Services ドメイン コントローラーへの DNS ネットワーク接続を確認した結果です。

![Domain join validation](screenshots/22_validation-domainjoin.png)

![DNS resolution validation](screenshots/23_validation-nslookup.png)

![DNS connectivity validation](screenshots/24_validation-testnetconnection.png)

---

# Security Considerations

🇺🇸

This environment was created as a hands-on learning lab.

The Windows Server VM does not have a public IP address, and connectivity to the managed domain is provided through private VNet connectivity and VNet peering.

The NSG configuration shown in the screenshots is intended for this lab environment and should be reviewed and restricted appropriately in a production environment.

🇯🇵

この環境は実習・学習目的のラボとして構築しています。

Windows Server VM にはパブリック IP アドレスを設定しておらず、マネージド ドメインへの接続にはプライベートな VNet 接続と VNet ピアリングを使用しています。

スクリーンショットに表示されている NSG 構成はラボ環境向けのものであり、本番環境では適切なアクセス制御と制限を検討する必要があります。

---

# Key Takeaways

🇺🇸

This lab provided hands-on experience with:

* Microsoft Entra Domain Services
* Managed domain controllers
* Azure virtual networking
* Subnet design
* DNS configuration
* VNet peering
* Windows Server deployment
* Domain joining
* DNS and network connectivity validation

🇯🇵

このラボを通して、以下の Azure 技術を実際に構築・検証しました。

* Microsoft Entra Domain Services
* マネージド ドメイン コントローラー
* Azure 仮想ネットワーク
* サブネット設計
* DNS 構成
* VNet ピアリング
* Windows Server デプロイ
* ドメイン参加
* DNS およびネットワーク接続の検証
