---
title: Create a virtual network - quickstart - Azure PowerShell
titleSuffix: Azure Virtual Network
description: In this quickstart, you create a virtual network using the Azure portal. A virtual network lets Azure resources communicate with each other and with the internet.
author: asudbring
ms.service: virtual-network
ms.topic: quickstart
ms.date: 04/13/2022
ms.author: allensu
ms.custom: devx-track-azurepowershell, mode-api
#Customer intent: I want to create a virtual network so that virtual machines can communicate with privately with each other and with the internet.
---

# Quickstart: Create a virtual network using PowerShell

A virtual network lets Azure resources, like virtual machines (VMs), communicate privately with each other, and with the internet. 

In this quickstart, you learn how to create a virtual network. After creating a virtual network, you deploy two VMs into the virtual network. You then connect to the VMs from the internet, and communicate privately over the virtual network.

## Prerequisites

- An Azure account with an active subscription. [Create an account for free](https://azure.microsoft.com/free/?WT.mc_id=A261C142F).
- Azure PowerShell installed locally or Azure Cloud Shell

If you choose to install and use PowerShell locally, this article requires the Azure PowerShell module version 5.4.1 or later. Run `Get-Module -ListAvailable Az` to find the installed version. If you need to upgrade, see [Install Azure PowerShell module](/powershell/azure/install-Az-ps). If you're running PowerShell locally, you also need to run `Connect-AzAccount` to create a connection with Azure.

## Create a resource group and a virtual network

There are a handful of steps you have to walk through to get your resource group and virtual network configured.

### Create the resource group

Before you can create a virtual network, you have to create a resource group to host the virtual network. Create a resource group with [New-AzResourceGroup](/powershell/module/az.Resources/New-azResourceGroup). This example creates a resource group named **CreateVNetQS-rg** in the **Eastus** location:

```azurepowershell-interactive
$rg = @{
    Name = 'CreateVNetQS-rg'
    Location = 'EastUS'
}
New-AzResourceGroup @rg
```

### Create the virtual network

Create a virtual network with [New-AzVirtualNetwork](/powershell/module/az.network/new-azvirtualnetwork). This example creates a default virtual network named **myVNet** in the **EastUS** location:

```azurepowershell-interactive
$vnet = @{
    Name = 'myVNet'
    ResourceGroupName = 'CreateVNetQS-rg'
    Location = 'EastUS'
    AddressPrefix = '10.0.0.0/16'    
}
$virtualNetwork = New-AzVirtualNetwork @vnet
```

### Add a subnet

Azure deploys resources to a subnet within a virtual network, so you need to create a subnet. Create a subnet configuration named **default** with [Add-AzVirtualNetworkSubnetConfig](/powershell/module/az.network/add-azvirtualnetworksubnetconfig):

```azurepowershell-interactive
$subnet = @{
    Name = 'default'
    VirtualNetwork = $virtualNetwork
    AddressPrefix = '10.0.0.0/24'
}
$subnetConfig = Add-AzVirtualNetworkSubnetConfig @subnet
```

### Associate the subnet to the virtual network

You can write the subnet configuration to the virtual network with [Set-AzVirtualNetwork](/powershell/module/az.network/Set-azVirtualNetwork). This command creates the subnet:

```azurepowershell-interactive
$virtualNetwork | Set-AzVirtualNetwork
```

## Create virtual machines

Create two VMs in the virtual network.

### Create the first VM

Create the first VM with [New-AzVM](/powershell/module/az.compute/new-azvm). When you run the next command, you're prompted for credentials. Enter a user name and password for the VM:

```azurepowershell-interactive
$vm1 = @{
    ResourceGroupName = 'CreateVNetQS-rg'
    Location = 'EastUS'
    Name = 'myVM1'
    VirtualNetworkName = 'myVNet'
    SubnetName = 'default'
}
New-AzVM @vm1 -AsJob
```

The `-AsJob` option creates the VM in the background. You can continue to the next step.

When Azure starts creating the VM in the background, you'll get something like this back:

```powershell
Id     Name            PSJobTypeName   State         HasMoreData     Location             Command
--     ----            -------------   -----         -----------     --------             -------
1      Long Running... AzureLongRun... Running       True            localhost            New-AzVM
```

### Create the second VM

Create the second VM with this command:

```azurepowershell-interactive
$vm2 = @{
    ResourceGroupName = 'CreateVNetQS-rg'
    Location = 'EastUS'
    Name = 'myVM2'
    VirtualNetworkName = 'myVNet'
    SubnetName = 'default'
}
New-AzVM @vm2
```

You'll have to create another user and password. Azure takes a few minutes to create the VM.

> [!IMPORTANT]
> Don't continue with the next step until Azure's finished.  You'll know it's done when it returns output to PowerShell.

[!INCLUDE [ephemeral-ip-note.md](../../includes/ephemeral-ip-note.md)]

## Connect to a VM from the internet

To get the public IP address of the VM, use [Get-AzPublicIpAddress](/powershell/module/az.network/get-azpublicipaddress).

This example returns the public IP address of the **myVM1** VM:

```azurepowershell-interactive
$ip = @{
    Name = 'myVM1'
    ResourceGroupName = 'CreateVNetQS-rg'
}
Get-AzPublicIpAddress @ip | select IpAddress
```

Open a command prompt on your local computer. Run the `mstsc` command. Replace `<publicIpAddress>` with the public IP address returned from the last step:

> [!NOTE]
> If you've been running these commands from a PowerShell prompt on your local computer, and you're using the Az PowerShell module version 1.0 or later, you can continue in that interface.

```cmd
mstsc /v:<publicIpAddress>
```
1. If prompted, select **Connect**.

1. Enter the user name and password you specified when creating the VM.

    > [!NOTE]
    > You may need to select **More choices** > **Use a different account**, to specify the credentials you entered when you created the VM.

1. Select **OK**.

1. You may receive a certificate warning. If you do, select **Yes** or **Continue**.

## Communicate between VMs

1. In the Remote Desktop of **myVM1**, open PowerShell.

1. Enter `ping myVM2`.

    You'll get a reply message like this:

    ```powershell
    PS C:\Users\myVM1> ping myVM2

    Pinging myVM2.ovvzzdcazhbu5iczfvonhg2zrb.bx.internal.cloudapp.net
    Request timed out.
    Request timed out.
    Request timed out.
    Request timed out.

    Ping statistics for 10.0.0.5:
        Packets: Sent = 4, Received = 0, Lost = 4 (100% loss),
    ```

    The ping fails, because it uses the Internet Control Message Protocol (ICMP). By default, ICMP isn't allowed through your Windows firewall.

1. To allow **myVM2** to ping **myVM1** in a later step, enter this command:

    ```powershell
    New-NetFirewallRule –DisplayName "Allow ICMPv4-In" –Protocol ICMPv4
    ```

    That command lets ICMP inbound through the Windows firewall.

1. Close the remote desktop connection to **myVM1**.

1. Repeat the steps in [Connect to a VM from the internet](#connect-to-a-vm-from-the-internet). This time, connect to **myVM2**.

1. From a command prompt on the **myVM2** VM, enter `ping myVM1`.

    You'll get a reply message like this:

    ```cmd
    C:\windows\system32>ping myVM1

    Pinging myVM1.e5p2dibbrqtejhq04lqrusvd4g.bx.internal.cloudapp.net [10.0.0.4] with 32 bytes of data:
    Reply from 10.0.0.4: bytes=32 time=2ms TTL=128
    Reply from 10.0.0.4: bytes=32 time<1ms TTL=128
    Reply from 10.0.0.4: bytes=32 time<1ms TTL=128
    Reply from 10.0.0.4: bytes=32 time<1ms TTL=128

    Ping statistics for 10.0.0.4:
        Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
    Approximate round trip times in milli-seconds:
        Minimum = 0ms, Maximum = 2ms, Average = 0ms
    ```

    You receive replies from **myVM1**, because you allowed ICMP through the Windows firewall on the **myVM1** VM in a previous step.

1. Close the remote desktop connection to **myVM2**.

## Clean up resources

When you're done with the virtual network and the VMs, use [Remove-AzResourceGroup](/powershell/module/az.resources/remove-azresourcegroup) to remove the resource group and all the resources it has:

```azurepowershell-interactive
Remove-AzResourceGroup -Name 'CreateVNetQS-rg' -Force
```

## Next steps

In this quickstart: 

* You created a default virtual network and two VMs. 
* You connected to one VM from the internet and communicated privately between the two VMs.

Private communication between VMs is unrestricted in a virtual network. 

Advance to the next article to learn more about configuring different types of VM network communications:
> [!div class="nextstepaction"]
> [Filter network traffic](tutorial-filter-network-traffic.md)
