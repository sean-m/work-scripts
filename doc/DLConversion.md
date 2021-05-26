# Notes On Changes
The DLConversion.ps1 script is a bit unruly and is missing some essential documentation. Some of that will be documented here, as well as some notes on what changes should be made to the upstream script if attempting to merge with our fork.

## Normalize Upstream Before Merging
Convert all boolean and null literals to their lowercase version: $TRUE -> $true, $Null -> $null, etc. Find replace works fine for this. Hundreds of entries will show in a diff if you try merging without changing them in the upstream source first. I would have stuck with the scripts own convention, upppercase, lowercase, pascal-case but it kind of just did everything all over the place. 

### Merge Steps
1. Create a feature branch from 'main'.
1. Pull down the upstream script.
2. Normalize literal values: $true, $false, $null.
3. Merge into src/DLConversion.ps1 with your favorite merge tool.
4. Verify the merge was clean.
5. Checkout 'main' and merge in your feature branch.


## Credentials
The script lists two credential files that it assumes are in `C:\Scripts\` or some other path specified in the `$script:credentialFilePath` variable. These files are assumed to be CliXML serialized credential objects. That is variables that store the output of `Get-Credential` but written out to a file on the disk. These are stored as SecureString objects with the MS CAPI sub-system. It's a standard API that Windows uses when "good enough" encryption is fine. CAPI how Windows stores passwords in memory, it's used in Windows Credential Manager, etc.

Here's how you store a credential in a file:
```powershell

# The place we're storing the credential (will bomb if C:\Scripts isn't there)
$outputFile = "C:\Scripts\someCredential.xml"

$cred = Get-Credential -Message "The user account we're storing in $outputFile"

# Write the credential to a file, confirm overwriting 
# if the file already exists.
$cred | Export-Clixml -Path $outputFile -Confirm
```

That snippet can be used to create the files DLConversion expects.

## Script Flags
Not an exhaustive list, just the interesting bits.

### dlToConvert
This is the identity reference for the target group so can be a distinguished name or mail attribute. Any format accepted as the Get-DistributionGroup -Identity parameter.

### ignoreInvalidDLMember
Default: `$false`
This causes the script to bomb if there are any missing members. If set to $true it logs the missing member and continues. Mine the logs for missing memberships and resolving later could be useful.

### ignoreInvalidManagedByMember
Default: `$false`
Similar to ignoreInvalidDLMember, if $true it logs and skips group managers that exist in premise but not in the cloud.

### convertToContact
Default: `$true`
After cloud group is created, on-premise is sequestered/removed, a contact is created on-premise with the same mail and other attributes as the DL. Presumably this allows on-premise groups that aren't migrated to maintain that contact as a member. For mail routed through Exchange on-prem this works fine.

### requireInteractiveCredentials
Default: `$false`
Prompts for credentials to Exchange, EXO and AADC. These are read in from a [credential](#user-content-credentials) file by default.

### automaticAdConnectSync  (custom)
Default: $true
This is the default behavior for the upstream DLConversion script. It initiates a remote PowerShell session to a designated Azure AD Connect server in order trigger a delta sync and the correct time. When set to $false, the script will pause so a sync can be triggered manually. This also helps with running multiple copies of the script concurrently; delta syncs can be triggered when all instances are blocking and proceed at the same time.

### forceDomainReplication  (custom)
Default: $true
This is the default behavior for the upstream DLConversion script. When $true, a remote connection to a specified domain controller is made and `repadmin /syncall /A` is invoked.

**Note:** if you aren't forcing AD replication, increase the replication timeout, it's set to 1 minute by default.

### logSessionID  (custom)
Default: `[string]::Empty`
An optional ID to specify for a given script instance. By default the script logs to DLConversion.log but if an id is specified, it writes to DLConversion-\[id\].log. When running multiple instances, logging would collide and make debugging a nightmare.