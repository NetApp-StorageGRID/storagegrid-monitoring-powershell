# StorageGRID Monitoring

## StorageGRID health check

Install StorageGRID PowerShell Module in PowerShell 5 or later (PowerShell 6 is available for Windows, Mac OS and Linux):

```powershell
Install-Module -Name StorageGRID-Webscale -Scope CurrentUser
```

Define Admin Node FQDN (please change to your admin node FQDN!)

```powershell
$AdminNode = "cbc-sg-admin.muccbc.hq.netapp.com"
```

Connect as Administrator user to create a new group for monitoring:

```powershell
Connect-SgwServer -Name $AdminNode
```

We recommend to use an AD group as they work with Single-Sign-On as well but a local group also works.

To grant alarm management rights to an existing AD group named **monitoring** use:

```powershell
$Group = New-SgwGroup -UniqueName federated-group/monitoring -DisplayName monitoring -AlarmAcknowledgment
```

Then please provide the username and password of an AD monitoring user:

```powershell
$Credential = Get-Credential -Message "Please insert username and password for AD monitoring user (username must be in format domain\username or username@domain)"
```

To create a local group with alarm management use:

```powershell
$Group = New-SgwGroup -UniqueName monitoring -DisplayName monitoring -AlarmAcknowledgment
```

If using a local group, a local user needs to be created as well:

```powershell
$Credential = Get-Credential -Message "Please insert username and password for local monitoring user"
$Group | New-SgwUser -UniqueName $Credential.UserName -Password $Credential.GetNetworkCredential().Password
```

Now create a new profile to safely store monitoring user credentials and admin node name. The username and password will be stored in $HOME/.sgw/credentials of the currently logged in user and will only be readable by that user. This follows Microsoft best practices for Windows and is the same behavior as for SSH keys on Mac OS X and Linux.

```powershell
New-SgwProfile -ProfileName "monitoring" -Name $AdminNode -Credential $Credential
```

The system health can now be retrieved using the following command. The health check does not return acknowledged alarms (use `Get-SgwAlarms` if you need them as well).

```powershell
$Health = Get-SgwHealth -ProfileName "monitoring"
```

The following could be a check to create tickets in a monitoring system

```powershell
$Health = Get-SgwHealth -ProfileName "monitoring"

if ($Health.alarms.critical -gt 0 -or $Health.alarms.major -gt 0 -or $Health.alarms.minir -gt 0) {
    # wait 1 minute and check if alarms have been autocleared
    Start-Sleep -Seconds 60
    $Health = Get-SgwHealth -ProfileName "monitoring"

    if ($Health.alarms.critical -gt 0) {
        Write-Warning "Currently there are $($Health.alarms.critical) critical alarms, opening a high priority ticket"
        # check if a ticket for a critical condition already exists, otherwise create a ticket with high priority (e.g. needs immediate attention)
    }
    elseif ($Health.alarms.major -gt 0) {
        Write-Warning "Currently there are $($Health.alarms.major) major alarms and no critical alarms, opening a medium priority ticket"
        # check if a ticket for a critical or major condition already exists, otherwise create a ticket with medium priority (e.g. needs attention as soon as possible during business hours)
    }
    elseif ($Health.alarms.minor -gt 0) {
        Write-Warning "Currently there are $($Health.alarms.minor) minor alarms and no major or critical alarms, opening a low priority ticket"
        # check if a ticket for a critical or major condition already exists, otherwise create a ticket with low priority (e.g. should be checked during business hours)
    }
}
```

All current alarms can be retrieved using

```powershell
$Alarms = Get-SgwAlarms -ProfileName "monitoring"
```

To include alread acknowledged alarms, use

```powershell
$Alarms = Get-SgwAlarms -ProfileName "monitoring" -IncludeAcknowledged 
```

## S3 service availability monitoring

Install S3 PowerShell Module in PowerShell 5 or later (PowerShell 6 is available for Windows, Mac OS and Linux):

```powershell
Install-Module -Name S3-Client -Scope CurrentUser
```

Define Admin Node FQDN (please change to your admin node FQDN!)

```powershell
$AdminNode = Read-Host -Prompt "Please admin node FQDN (e.g. admin-node.example.com)"
```

For S3 service availability monitoring a separate tenant should be created and a bucket used for monitoring. First, connect to the StorageGRID admin node with a user which has the privilege to create new tenants:

```powershell
Connect-SgwServer -Name $AdminNode
```

Now create a new tenant. To prevent accidental or malicious usage of large amounts of space in the grid with this tenant, it is a good idea to set a quota

If Single-Sign-On is not activated, use the following steps to create the tenant and connect to the tenant

```powershell
$Credential = Get-Credential -Message "Please insert password for monitoring tenant root user" -UserName "root"
$Account = New-SgwAccount -Name monitoring -Capabilities s3,management -UseAccountIdentitySource $false -AllowPlatformServices $false -Quota 1gb -Password $Credential.GetNetworkCredential().Password
$Account | Connect-SgwServer -Name $AdminNode -Credential $Credential
Get-SgwGroups | Update-SgwGroup -S3FullAccess
```

If Single-Sign-On is activated, use these steps

```powershell
$RootGroup = Read-Host -Prompt "Please specify which AD group should be granted root access for this tenant (you can use the same monitoring group)"
$Account = New-SgwAccount -Name monitoring -Capabilities s3,management -UseAccountIdentitySource $false -AllowPlatformServices $false -Quota 1gb -GrantRootAccessToGroup $RootGroup
$Credential = Get-Credential -Message "Please insert username and password for AD monitoring user (username must be in format domain\username or username@domain)"
$Account | Connect-SgwServer -Name $AdminNode -Credential $Credential
Get-SgwGroups | Update-SgwGroup -S3FullAccess
```

Now we need to create an S3 Credential to be used for S3 access

```powershell
$AccessKey = New-SgwS3AccessKey
```

Now create a new profile to safely store the S3 credentials and S3 endpoint. The S3 access key and secret access key will be stored in $HOME/.aws/credentials of the currently logged in user and will only be readable by that user. This follows AWS best practices.

```powershell
$EndpointUrl = Read-Host -Prompt "Please insert S3 endpoint URL including protocol and port (e.g. https://s3.example.com:8082)"
New-AwsProfile -ProfileName monitoring -EndpointUrl $EndpointUrl -AccessKey $AccessKey.accessKey -SecretKey $AccessKey.secretAccessKey
```

The following is a script to check S3 service availability and should be executed regularly (I would recommend every 5 minutes):

```powershell
$BucketName = [System.Guid]::NewGuid().guid
Write-Information "Creating bucket $BucketName"
try {
    New-S3Bucket -ProfileName monitoring -BucketName $BucketName
}
catch {
    Throw "Bucket was not created successfully"
}

# wait 10 seconds to ensure that bucket shows up in listing
Start-Sleep -Seconds 10
$Buckets = Get-S3Buckets -ProfileName monitoring

if ($Buckets.Count -ne 1) {
    Write-Warning "Bucket count is $($Buckets.Count), probably the last service availability check was interrupted or did not complete or Bucket creation did not succeed"
} 
elseif ($Buckets.BucketName -notmatch $BucketName) {
    Throw "Bucket $BucketName not found"
}

# create 10,000 empty objects
foreach ($Key in 1..10000) {
    Write-S3Object -ProfileName monitoring -BucketName $BucketName -Key $Key
}

# create a sparse 1GB file
$FilePath = New-TemporaryFile
$File = [io.file]::Create($FilePath)
$File.SetLength(1gb)
$File.Close()

$OriginalFileHash = $FilePath | Get-FileHash

# upload 1GB file
Write-S3Object -ProfileName monitoring -BucketName $BucketName -InFile $FilePath

$FilePath | Remove-Item

# read 1GB file
Read-S3Object -ProfileName monitoring -BucketName $BucketName -Key $FilePath.Name -OutFile $FilePath

$DownloadedFileHash = $FilePath | Get-FileHash

if ($DownloadedFileHash.Hash -ne $OriginalFileHash) {
    Throw "Downloaded file hash does not match file hash of local file"
}

# list all objects
$Objects = Get-S3Objects -BucketName $BucketName

if ($Objects.Count -ne 10001) {
    Throw "Number of objects in listing does not equal 10001"
}

# remove bucket and force deletion of all objects in the bucket
Remove-S3Bucket -ProfileName monitoring -BucketName $BucketName -Force

# wait 10 seconds to ensure that bucket does not show up in listing anymore
Start-Sleep -Seconds 10
$Buckets = Get-S3Buckets -ProfileName monitoring

if ($Buckets.Count -ne 0) {
    Throw "Bucket count is $($Buckets.Count), bucket deletion seems to have failed"
}
```