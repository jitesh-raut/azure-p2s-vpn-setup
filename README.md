To configure the server settings for a Point-to-Site (P2S) VPN Gateway with certificate authentication in Azure, follow these steps:
### Step 1: Create a Virtual Network (VNet)

1. **Sign in to the Azure Portal**.
2. In the search bar at the top of the portal page, enter **"virtual network"** and select **Virtual network** from the Marketplace search results to open the **Virtual network** page.
3. On the **Virtual network** page, click **Create** to open the **Create virtual network** page.
4. On the **Basics** tab, configure the following settings:

   - **Subscription**: Ensure the correct subscription is selected.
   - **Resource group**: Select an existing resource group or create a new one.
   - **Name**: Enter a name for your virtual network.
   - **Region**: Select the region where you want the resources to reside.

   ![Virtual Network Basics](https://learn.microsoft.com/en-us/azure/includes/media/vpn-gateway-basic-vnet-rm-portal-include/basics.png)

5. Click **Next: Security** and leave the default values for this exercise.
6. Click **IP Addresses** to configure the IP settings:

   - **IPv4 address space**: Adjust the default address space or add a new one (e.g., `10.1.0.0/16`).

7. Click **+ Add subnet** to create a subnet:

   - **Subnet name**: Enter a name (e.g., `FrontEnd`).
   - **Subnet address range**: Specify the address range (e.g., `10.1.0.0/24`).

8. Review the IP addresses and remove any unnecessary address spaces or subnets.
9. Click **Review + create** to validate the settings.
10. After validation, click **Create** to create the virtual network.

### Step 2: Create a Gateway Subnet

1. On the page for your virtual network, select **Subnets** in the left pane.
2. Click **+ Gateway subnet** to open the **Add subnet** pane.
3. The **Name** is automatically set to `GatewaySubnet`. Adjust the IP address range if necessary (e.g., `10.1.255.0/27`).
4. Click **Save** to save the subnet.

### Step 3: Create the VPN Gateway

1. In the search bar at the top of the portal page, enter **"virtual network gateway"**. Select **Virtual network gateway** from the Marketplace search results to open the **Create virtual network gateway** page.
2. On the **Basics** tab, fill in the following details:

   ![VPN Gateway Instance Details](https://learn.microsoft.com/en-us/azure/includes/media/vpn-gateway-add-gw-portal/instance-details.png)

   - **Subscription**: Select the subscription you want to use.
   - **Resource group**: This will be autofilled when you select your virtual network.
   - **Name**: Enter a name for your gateway.
   - **Region**: Ensure it matches the region of the virtual network.
   - **Gateway type**: Select **VPN**.
   - **SKU**: Choose the desired gateway SKU (e.g., `VpnGw2`).
   - **Generation**: Select **Generation2**.
   - **Virtual network**: Select the virtual network created in Step 1.
   - **Gateway subnet address range or Subnet**: Ensure the gateway subnet is selected.

3. Configure the **Public IP address**:

   - **Public IP address type**: Select **Standard**.
   - **Public IP address**: Leave **Create new** selected.
   - **Public IP address name**: Enter a name for the public IP address.
   - **Public IP address SKU**: This is auto-selected.
   - **Assignment**: Typically auto-selected as **Static**.
   - **Enable active-active mode**: Select **Disabled** unless needed.
   - **Configure BGP**: Select **Disabled** unless required.

4. Click **Review + create** to validate the settings.
5. After validation, click **Create** to deploy the VPN gateway. Note that creating a gateway can take up to 45 minutes or more.



### Creating and Managing Certificates

#### Create a Self-Signed Root Certificate and Generate Client Certificates

**Requirements**: 
- Windows 10 or later, or Windows Server 2016 or later.

**Steps**:

1. **Open PowerShell with Elevated Privileges**:
   - From a computer running Windows 10 or later, or Windows Server 2016, open a Windows PowerShell console with elevated privileges.

2. **Create a Self-Signed Root Certificate**:
   - Use the following script to create a self-signed root certificate named 'P2SRootCert'. This certificate will be automatically installed in 'Certificates-Current User\Personal\Certificates'.
   - You can view the certificate by opening `certmgr.msc` or Manage User Certificates.

   ```powershell
   $params = @{
       Type = 'Custom'
       Subject = 'CN=P2SRootCert'
       KeySpec = 'Signature'
       KeyExportPolicy = 'Exportable'
       KeyUsage = 'CertSign'
       KeyUsageProperty = 'Sign'
       KeyLength = 2048
       HashAlgorithm = 'sha256'
       NotAfter = (Get-Date).AddMonths(24)
       CertStoreLocation = 'Cert:\CurrentUser\My'
   }
   $cert = New-SelfSignedCertificate @params
   ```

3. **Generate a Client Certificate**:
   - Each client computer that connects to a VNet using point-to-site must have a client certificate installed.
   - Use the following script to generate a client certificate from the self-signed root certificate:

   ```powershell
   $params = @{
       Type = 'Custom'
       Subject = 'CN=P2SChildCert'
       DnsName = 'P2SChildCert'
       KeySpec = 'Signature'
       KeyExportPolicy = 'Exportable'
       KeyLength = 2048
       HashAlgorithm = 'sha256'
       NotAfter = (Get-Date).AddMonths(18)
       CertStoreLocation = 'Cert:\CurrentUser\My'
       Signer = $cert
       TextExtension = @(
           '2.5.29.37={text}1.3.6.1.5.5.7.3.2'
       )
   }
   New-SelfSignedCertificate @params
   ```

#### Creating Additional Client Certificates

1. **Identify the Self-Signed Root Certificate**:
   - If using a new PowerShell session, list the installed certificates:

   ```powershell
   Get-ChildItem -Path "Cert:\CurrentUser\My"
   ```

   - Locate the self-signed root certificate from the list and copy its thumbprint.

2. **Declare a Variable for the Root Certificate**:
   - Replace `<THUMBPRINT>` with the thumbprint of the root certificate:

   ```powershell
   $cert = Get-ChildItem -Path "Cert:\CurrentUser\My\<THUMBPRINT>"
   ```

#### Export the Root Certificate Public Key (.cer)

1. **Open Manage User Certificates**:
   - Locate the self-signed root certificate under 'Certificates - Current User\Personal\Certificates'.
   - Right-click the certificate, select All Tasks -> Export.
   - Follow the Certificate Export Wizard steps to export the certificate in Base-64 encoded X.509 (.CER) format.

2. **Retrieve Certificate Data**:
   - Open the exported certificate file with a text editor like Notepad.
   - Copy the necessary certificate data for uploading to Azure.

#### Export the Self-Signed Root Certificate and Private Key (Optional)

1. **Backup the Root Certificate**:
   - Follow the steps in Export a client certificate to export the self-signed root certificate as a .pfx file.

#### Export the Client Certificate

1. **Open Manage User Certificates**:
   - Locate the client certificates in 'Certificates - Current User\Personal\Certificates'.
   - Right-click the desired client certificate, select All Tasks -> Export.
   - Follow the Certificate Export Wizard steps to export the client certificate with the private key and include all certificates in the certification path.

#### Add Address Pool

### 1. Navigate to Point-to-Site Configuration
1. **Go to your Virtual Network Gateway**: Navigate to the virtual network gateway that you created in the previous section.
2. **Open Point-to-Site Configuration**: In the left pane, select **Point-to-site configuration**.
3. **Configure Now**: Click **Configure now** to open the configuration page.

### 2. Add Address Pool
1. **Address Pool Box**: On the Point-to-site configuration page, in the **Address pool** box, add the private IP address range you want to use. Ensure that this range does not overlap with your on-premises network or the VNet you are connecting to.
    - Minimum subnet mask: 29-bit for active/passive configuration, 28-bit for active/active configuration.
    
    ![Configuration Address Pool](https://learn.microsoft.com/en-us/azure/vpn-gateway/media/vpn-gateway-howto-point-to-site-resource-manager-portal/configuration-address-pool.png)

### 3. Specify Tunnel and Authentication Type
1. **Tunnel Type**: On the Point-to-site configuration page, select the **Tunnel type**. For this exercise, choose **IKEv2 and OpenVPN(SSL)**.
    
    ![Configuration Tunnel Type](https://learn.microsoft.com/en-us/azure/vpn-gateway/media/vpn-gateway-howto-point-to-site-resource-manager-portal/configuration-tunnel-type.png)
2. **Authentication Type**: Select **Azure certificate** for the authentication type.
    
    ![Configuration Authentication Type](https://learn.microsoft.com/en-us/azure/vpn-gateway/media/vpn-gateway-howto-point-to-site-resource-manager-portal/configuration-authentication-type.png)

### 4. Upload Root Certificate Public Key Information
1. **Export Root Certificate**: Make sure you have exported the root certificate as a Base-64 encoded X.509 (.CER) file.
2. **Open Certificate**: Open the certificate with a text editor like Notepad. Copy the certificate data as one continuous line without carriage returns or line feeds.
    
    ![Notepad Root Certificate](https://learn.microsoft.com/en-us/azure/vpn-gateway/media/vpn-gateway-howto-point-to-site-resource-manager-portal/notepad-root-cert.png)
3. **Paste Certificate Data**: Navigate to your Virtual Network Gateway -> Point-to-site configuration page in the Root certificate section. Paste the certificate data into the **Public certificate data** field and name the certificate.
4. **Save Configuration**: Select **Save** at the top of the page to save all configuration settings.

### 5. Generate VPN Client Profile Configuration Files
1. **Generate Configuration Files**:
    - In the Azure portal, go to the virtual network gateway.
    - On the virtual network gateway page, select **Point-to-site configuration**.
    - At the top of the page, select **Download VPN client**. This generates the configuration package used to configure VPN clients.
    
    ![Download Configuration](https://learn.microsoft.com/en-us/azure/includes/media/vpn-gateway-generate-profile-portal/download-configuration.png)
2. **Unzip Configuration Package**: Once the package is generated, unzip the file to view the folders. Use the files to configure your VPN client.

### 6. Configure VPN Clients and Connect to Azure
- **Download Link for Windows**: Use [this link](https://learn.microsoft.com/en-us/azure/vpn-gateway/point-to-site-vpn-client-certificate-windows-azure-vpn-client) to download the Azure VPN Client for Windows.
- **Verify Connection**:
    1. Open an elevated command prompt.
    2. Run `ipconfig /all`.
    3. Verify that the IP address received is from the address pool you specified.

### 7. Connect to a Virtual Machine
1. **Open Remote Desktop Connection**: Enter **RDP** or **Remote Desktop Connection** in the search box on the taskbar, then select **Remote Desktop Connection**.
2. **Connect to VM**: Enter the private IP address of the VM to connect.


## Best Practices:
When users have Microsoft 365 Basic licenses and you need to configure Point-to-Site (P2S) VPN connections in Azure, it's important to consider best practices that align with both the capabilities of Microsoft 365 Basic and the security and operational needs of your organization. Here are the best practices:

### 1. **Use Certificate-based Authentication**
Microsoft 365 Basic does not include advanced security features like Azure Active Directory Premium, which means you cannot use Conditional Access or other advanced authentication mechanisms. Therefore, the best practice is to use certificate-based authentication for your VPN connections.

- **Generate and Distribute Certificates**: Ensure that root and client certificates are securely generated and distributed. Use a secure method for distributing the client certificates to users.
- **Regularly Rotate Certificates**: Periodically update the certificates to enhance security.

### 2. **Choose Appropriate Tunnel Types**
Given the varied support for VPN clients across different operating systems, select tunnel types that are widely supported:

- **IKEv2 and OpenVPN (SSL)**: This combination is recommended as it provides flexibility for connecting from different operating systems including Windows, Linux, and macOS.
  - Windows clients can use the native VPN client or the Azure VPN Client for OpenVPN.
  - Linux clients can use the Azure VPN Client for OpenVPN or strongSwan for IKEv2.

### 3. **Configure Address Pools Carefully**
Ensure that the address pool you configure does not overlap with your on-premises network or the Azure Virtual Network (VNet) to avoid routing issues.

- **Use a Non-overlapping Address Range**: Choose a private IP address range (e.g., 10.2.0.0/24) that does not conflict with any existing network ranges.

### 4. **Secure Distribution of VPN Client Configuration**
After generating the VPN client profile configuration files, ensure secure distribution to end users.


### 5. **Implement Network Security Best Practices**
Ensure that your network security configurations are robust to protect your Azure resources.

- **Network Security Groups (NSGs)**: Configure NSGs to control inbound and outbound traffic to your Azure VNets.
- **Firewalls**: Use Azure Firewall or other firewall solutions to further protect your network.


References:

https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-howto-point-to-site-resource-manager-portal

https://learn.microsoft.com/en-us/azure/vpn-gateway/point-to-site-about

![vpn-gateway-howto-point-to-site-rm-ps](https://learn.microsoft.com/en-us/azure/vpn-gateway/media/vpn-gateway-howto-point-to-site-rm-ps/point-to-site-diagram.png)


